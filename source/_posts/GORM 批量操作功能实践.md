---
title: GORM 批量操作功能实践
top: false
cover: false
toc: true
mathjax: true
date: 2021-03-29 16:27:38
password:
summary:
tags:
categories:
---


## 前言

本文基于 gorm v1.9.11 编写。

GORM 是 golang 语言的较流行的 ORM 框架之一，本质上是对 golang sql 包的封装。

团队在使用过程中遇到了一些问题，为了加深理解，更好的使用 gorm，故阅读了部分源码，并给出 gorm Scan() 方法 和 Updates() 方法的流程解析。

同时使用的版本不支持批量创建与更新，不能满足业务需求，故参照相关资料，完成了基于 gorm 的批量查询与批量更新操作。

## GORM 结构体

在正式内容之前，我们需要先了解下 gorm 的核心结构体，这些结构体是 gorm 所有功能的承载者.

GORM 核心结构体主要分为为:

- Scope，查询的全局管家，持有一次查询的全部信息
- DB，查询操作的执行者，持有本次查询所有需要执行的 Callback、查询条件、数据库连接、表结构信息等
- ModelStruct，传入的结构体信息，gorm 依此将数据库表与具体结构体映射起来
- StructField，ModelStruct 主要持有的数据，gorm 依此将数据库表的列与结构体的字段映射起来。

### *gorm.Scope

Scope 是一次查询的的全局管家，持有本次查询的所有上下文信息，包括 db 连接、结构体信息、查询条件、查询参数、查询结果等。
其中比较重要的字段有：

- Search：持有所有查询条件，包括 join、group 等。
- SQLVars：查询的参数
- Value：传入的结构体
- db：gorm 自己封装的 DB，持有着数据库连接

```go
type Scope struct {
	Search          *search       // 过滤条件
	Value           interface{}   // 传入的结构体
	SQL             string        // 拼接的 sql
	SQLVars         []interface{} // 拼接的 sql 参数
	primaryKeyField *Field        // 主键
	fields          *[]*Field     // 传入的结构体解析出的字段

	db          *DB       // 持有的 db
	instanceID  string    // 标记 scope 的唯一名称
	selectAttrs *[]string // 解析出的字段
	skipLeft    bool      // 是否跳过剩余 Callback
}
```

### *gorm.DB

DB 在 gorm 中负责具体的数据库执行操作。

核心的字段包括：

- db：实际的数据库操作执行者，实际由 *sql.DB 和 *sql.Tx 实现
- callbacks：本次数据库查询需要执行的所有 callback

```go
// DB contains information for current db connection
type DB struct {
	sync.RWMutex             // 锁
	Value        interface{} // 传入的结构体
	Error        error       // DB 操作的 error
	RowsAffected int64       // DB 操作数据库返回的影响的行数

	// single db
	db                SQLCommon    // 实际的 db 操作执行者
	blockGlobalUpdate bool         // 为true时，可以在update在没有where条件是报错，避免全局更新
	logMode           logModeValue // 日志模式，gorm提供了三种
	logger            logger       // 日志
	search            *search      // 搜索条件
	values            sync.Map     // 存储 value

	// global db
	parent        *DB       // 父 DB, 主要获取 Callback
	callbacks     *Callback // 此 DB 对应的 Callback
	dialect       Dialect   // 适配不同的数据库的接口
	singularTable bool      // 表名是否为单数

	// function to be used to override the creating of a new timestamp
	nowFuncOverride func() time.Time
}
```

### *gorm.ModelStruct

ModelStruct 结构并没有过多信息，主要就是自身解析出的 Fields 和 表名。

```go
// ModelStruct model definition
type ModelStruct struct {
	PrimaryFields []*StructField // 主键
	StructFields  []*StructField // model 中解析出的字段
	ModelType     reflect.Type   // model 的反射类型

	defaultTableName string     // model 对应的表名
	l                sync.Mutex // 锁
}
```

### *gorm.StructField

StructField 持有的信息相对较多，都是通过反射从传入的 model 中解析出来的。

```go
// StructField model field's struct definition
type StructField struct {
	DBName          string // 字段对应的数据库列名
	Name            string // 字段在结构体中的名字
	Names           []string // 字段对应 Struct 的子字段
	IsPrimaryKey    bool // 字段是否是主键
	IsNormal        bool // 是否是普通字段
	IsIgnored       bool // 字段是否应该被忽略
	IsScanner       bool // 字段是否实现了 Scanner 接口
	HasDefaultValue bool // 字段是否有默认值(gorm tag 的 default value)
	Tag             reflect.StructTag // 字段反射出的 Tag
	TagSettings     map[string]string // 解析出的 Tag
	Struct          reflect.StructField // 反射出的 Field
	IsForeignKey    bool // 是否是外键(gorm tag)
	Relationship    *Relationship // 字段的映射关系

	tagSettingsLock sync.RWMutex // 读写锁
}
```

## GORM 数据库操作流程解析

这里先了解 GORM 的几个完整的数据库操作流程示例。

为了方便演示与理解，我们定义一个简单的表结构：

```go
type User struct {
	Id    int64  `gorm:"column:id"`
	Name  string `gorm:"column:name"`
}

func (User) TableName() string {
	return "user_tab"
}
```

然后执行以下伪代码:

```go
var (
	db *gorm.DB
)
...
...
user := &User{
	Id:   0,
	Name: "zhangsan",
}
tx := db.Begin()
err = tx.Create(user).Error
if err != nil {
	tx.Rollback()
	return err
}
err = tx.Commit().Error
if err != nil {
	return err
}	err = db.First(user).Error
if err != nil {
	return err
}
user.Name = "zhangsan2"
err = db.Model(user).Updates(user).Error
if err != nil {
	return err
}
return nil
```

这段代码中，涉及到了事务、创建、查找、更新四种主要操作。我们会依次解释这些操作中 gorm 各结构体数据的变化。

### 开启事务

```go
tx := db.Begin()
```

gorm 本身并没有实现自己的事务机制，这里本质是对 *sql.DB 的封装，实际上是调用的是 *sql.DB 的 Begin 方法。

gorm 的所有 DB 操作都是由 db SQLCommon 这个 interface 来具体执行的的。而这个接口实际上有两个实现：分别为 *sql.DB 和 *sql.Tx。

其中 *sql.DB 实际上是全局唯一的，即所有 *gorm.DB 果没有开启事务，那么都会由同一个 *sql.DB 来执行。并且由 *sql.DB 来管理连接池。

而当开启事务时，*gorm.DB 持有的 SQLCommon 就由 *sql.DB 变为了 *sql.Tx。由全局唯一的连接池变为了一个具体的连接。

sql.DB.Begin() 方法会返回一个开启事务的数据量连接，gorm.DB 此时会保存此连接到字段 db 中，并返回出来。

这样后续所有的数据库操作都会使用同一个 shop。

### 创建

```go
err = tx.Create(user1).Error
```

gorm 所有的绝大多数操作都是由 Callback 来实现的，其中 Create 操作的 Callback 就在 callback_create.go 中，核心操作为 createCallback。

createCallback 会使用 (*Scope).GetModelStruct() 方法解析传入的结构体的所有字段，然后拼接创建的 SQL 语句。

字段 id 的解析结果为:
![image](https://raw.githubusercontent.com/DarainS/Images/main/images/20210621212821.png)

字段 name 的解析结果为:
![image](https://raw.githubusercontent.com/DarainS/Images/main/images/20210621213252.png)

拼接出的 SQL 与结构体为:
![image](https://raw.githubusercontent.com/DarainS/Images/main/images/20210621213211.png)

之后调用 (*Scope).SQLDB().Exec() 方法执行 SQL，同时设置主键 Id。

这样一次创建操作就完成了。比较核心的操作是如何解析结构体，以及从解析出的结构体拼接出 SQl。

后面我们会讲具体的应用实例。

### 事务回滚

```go
if err != nil {
	tx.Rollback()
	return err
}
```

开启事务之后 (*DB).db 持有的就是 *sql.Tx 实例了，这里就是调用了 (*sql.Tx).Rollback() 方法。

### 事务提交

```go
err = tx.Commit().Error
```

同样的，这里调用的 (*sql.Tx).Commit() 方法

### 查询操作

```go
err = db.First(user).Error
```

First 函数的 Callback 存储在 callback_query.go 文件中，核心函数为 queryCallback。

与创建操作不同的是，查询操作需要额外解析查询条件，并拼接 SQL。

解析出的查询条件如下：
![查询条件](https://raw.githubusercontent.com/DarainS/Images/main/images/20210621213951.png)

之后调用 (*Scope) prepareQuerySQL() 方法拼接出 SQL。

拼接出的 SQL 如下：

![SQL](https://raw.githubusercontent.com/DarainS/Images/main/images/20210621214050.png)

然后执行 SQL 并调用 (*Scope).scan() 方法将结果反射到结构体中。

比较值得学习的是通过查询语句拼接 SQL 的操作和 scan 方法将数据库行反射到结构体中。

### 更新操作

```go
err = db.Model(user).Updates(user).Error
```

Updates 函数的 Callback 存储在 callback_update.go 文件中，核心函数为 updateCallback。

对于 gorm 来说，更新操作既用到了查询语句拼接，也用到了创建操作的字段映射，其实就是创建+查询的组合。

### 回顾

gorm 所有的操作本质上就是做了三件事。

1. 拼接 SQL
2. 执行 SQL
3. 写回结果

其中逻辑做多的在于拼接 SQL，SQL 拼好了，通过调用 golang 的 sql 包就足够完成 db 的操作了。


## 团队实践

在使用 gorm 的过程中，我们总结了一些相对有效的实践，在此分享。

需要注意的是，为了便于阅读，我们将本文贴出的部分代码简化为了伪代码，实际代码会做兼容与边际检查以保证其健壮性，仅提供解决问题的思路，并不建议使用到生产环境。

### 事务

为了降低开发者的心智负担，以及保证代码的健壮性，避免数据库连接泄漏，我们简单封装了 gorm.DB 的事务操作，简化事务行为。

```go
type Repo struct {
	context     context.Context
	db          *gorm.DB
}

// 事务代码的封装
func (s *Repo) WithTransaction(handleFunc  func() error) error {
    if s.db == nil {
        s.db = db.GetDB()
    }
    tx := s.db.Begin()
    if tx.Error != nil {
        return tx.Error
    }
    s.db = tx
	defer func() {
		if r := recover(); r != nil {
			_ = s.Rollback()
			panic(r)
		}
	}()

	err := handleFunc()
	if err != nil {
		_ = s.Rollback()
		return err
	}
	// handle ctx timeout
	if s.Err() != nil {
		_ = s.Rollback()
		return s.Err()
	}
	err = s.Commit()
	if err != nil {
		return err
	}
	return nil
}

// 使用事务
err = r.WithTransaction(func() error {
    // do something
    return nil
})

// 对比
tx := db.Begin()
err = tx.Create(user).Error
if err != nil {
	tx.Rollback()
	return err
}
err = tx.Create(user2).Error
if err != nil {
	tx.Rollback()
	return err
}
user, err = QueryUser()
if err != nil {
	tx.Rollback()
	return err
}
err = tx.Commit().Error
if err != nil {
	return err
}
```

这种封装要求所有的 DB 操作都必须传入 *Repo，使得开发者不再需要显式的 begin、commit、rollback，
不必再显式的操作事务，以及处理嵌套事务的问题，能够显著降低开发者心智负担。

### 批量创建

由于 gorm 并未提供批量创建的功能，因此我们写了一个拼接批量创建 SQL 的函数，用于批量创建操作。

核心逻辑就是拼接出形如 INSERT INTO  XX_tab (field1, field2) VALUES (xxx1,xxx2),(xxx3,xxx4) 这样的 SQL

其中表名可以通过 (*Scope).QuotedTableName() 方法获得。

```go
func batchInsert(db *gorm.DB, objArr []interface{}, ) error {
	mainObj := objArr[0] // 要求传入的 model 都是同一个表
	mainScope := db.NewScope(mainObj)
	mainFields := mainScope.Fields() // 利用 gorm 帮助解析出结构体字段
	fieldDBNames := make([]string, 0, len(mainFields))
	for i := range mainFields {
		// 跳过主键、空值、以及显式忽略的字段
		if (mainFields[i].IsPrimaryKey && mainFields[i].IsBlank) || (mainFields[i].IsIgnored) {
			continue
		}
		if str := mainFields[i].Tag.Get("gorm"); str == "" || str == "-" {
			continue
		}
		// 拼接出需要查询的字段
		fieldDBNames = append(fieldDBNames, mainScope.Quote(mainFields[i].DBName))
	}
	...
}
```

需要创建的结构体的字段名则通过上述代码获得，其中fieldDBNames 存储的就是数据库中的字段名称。

而需要追加的字段名通过类似的方式，使用 scope.AddToVars(fields[i].Field.Interface() 加到查询条件中。

```go
placeholdersArr := make([]string, 0, len(objArr))
for _, obj := range objArr {
	scope := db.NewScope(obj)
	fields := scope.Fields()
	placeholders := make([]string, 0, len(fields))
	for i := range fields {
		if (fields[i].IsPrimaryKey && fields[i].IsBlank) || (fields[i].IsIgnored) {
			continue
		}
		if str := mainFields[i].Tag.Get("gorm"); str == "" || str == "-" {
			continue
		}
		placeholders = append(placeholders, scope.AddToVars(fields[i].Field.Interface())) // 将字段值加到 scope.SQLVars 中, 同时 placeholders 追加了 $$$
	}
	placeholdersStr := "(" + strings.Join(placeholders, ", ") + ")"
	placeholdersArr = append(placeholdersArr, placeholdersStr)
	mainScope.SQLVars = append(mainScope.SQLVars, scope.SQLVars...)
}
```

这样我们就可以拼出真实 SQL 并执行：

```go
sql := fmt.Sprintf("INSERT INTO %s (%s) VALUES %s",
		mainScope.QuotedTableName(),
		strings.Join(fieldDBNames, ", "),
		strings.Join(placeholdersArr, ", "),
	)
mainScope.Raw(sql)
result, err := mainScope.SQLDB().Exec(mainScope.SQL, mainScope.SQLVars...)
```

这样就完成了批量创建函数。

### 批量更新

由于 gorm 并未提供批量更新的功能，因此我们使用 CASE THEN 控制流，通过主键 id 拼接更新字段的 SQL，完成了批量更新不同数据的功能。

#### 目标

拼接出形如 UPDATE XX_tab SET field1 = (CASE WHEN id=id1 THEN value1 WHEN id=id2 THEN value2), field2 = (CASE WHEN id=id1 THEN value1 WHEN id=id2 THEN value2) WHERE id IN (id1,id2) 的 SQL。

定义函数名

```go
// objArr 为要更新的表结构的 list
// updateFields 为要更新的字段的数据库名
func bulkUpdateImplement(db *gorm.DB, objArr []interface{}, updateFields []string) error {
```

定义存储数据的 map

```go
	updateFieldNameValuesMap := make(map[string][]interface{}, len(updateFields))
	for _, updateField := range updateFields {
		updateFieldNameValuesMap[updateField] = make([]interface{}, 0, len(updateFields))
	}
```	

初始化 mainScope

```go
mainObj := objArr[0]
mainScope := db.NewScope(mainObj)
```

获取主键

```go
primaryKey := mainScope.Quote(mainScope.PrimaryKey())
primaryKeyValues := make([]interface{}, 0, len(objArr))
```

获取要更新的字段名与其对应的书剑

```go
	for _, obj := range objArr {
		scope := db.NewScope(obj)
		fields := scope.Fields()
		if scope.PrimaryKeyZero() {
			return errors.New("primary key is empty")
		}
		// 获取主键值
		primaryKeyValues = append(primaryKeyValues, scope.PrimaryKeyValue())

		for _,field := range fields {
			if tag := field.Tag.Get("gorm"); tag == "" || tag == "-" {
				continue
			}
			if field.IsPrimaryKey {
				continue
			}
			dbName := field.DBName
			if _, exist := updateFieldNameValuesMap[dbName]; exist {
				// 获取要更新的字段值
				updateFieldNameValuesMap[dbName] = append(
					updateFieldNameValuesMap[dbName], field.Field.Interface())
			}
		}
	}
	...
}
```

拼接 CASE WHEN END SQL:

```go
setSQLs := make([]string, 0, len(updateFields))
for field, values := range updateFieldNameValuesMap {
	caseSQLs := make([]string, 0, len(primaryKeyValues))
	for i, value := range values {
		caseSQLs = append(caseSQLs, fmt.Sprintf("WHEN %s=%s THEN %s", primaryKey, mainScope.AddToVars(primaryKeyValues[i]), mainScope.AddToVars(value)))
	}
	setSQLs = append(setSQLs, fmt.Sprintf("%s = (CASE %s END)", mainScope.Quote(field), strings.Join(caseSQLs, " ")))
}
primaryKeyPlaceHolders := make([]string, len(primaryKeyValues))
for i, val := range primaryKeyValues {
	primaryKeyPlaceHolders[i] = mainScope.AddToVars(val)
}
```

组装 SQL:
```go
sql := fmt.Sprintf("UPDATE %s SET %s WHERE %s IN (%s)",
	mainScope.QuotedTableName(),
	strings.Join(setSQLs, ","),
	primaryKey,
	strings.Join(primaryKeyPlaceHolders, ","),
)
mainScope.Raw(sql)
```

执行 SQL:

```go
result, err := mainScope.SQLDB().Exec(mainScope.SQL, mainScope.SQLVars...)
```

演示代码:

```go
var users []*User
err = db.Model(&User{}).Limit(2).Scan(&users).Error
if err != nil {
	return err
}
for _, user := range users {
	user.Name = user.Name + "2"
}
err = BulkUpdate(db, users, []string{"name"})
if err != nil {
	return err
}
```

解析结果:
![解析结果](https://raw.githubusercontent.com/DarainS/Images/main/images/20210622143628.png)

这样通过 CASE WHEN END 的方式就完成了对数据的批量更新操作。

### 日志

目前上述两个批量操作还没有日志，这里可以参照 gorm 的日志记录加上日志。

```go
t := time.Now()
result, err := mainScope.SQLDB().Exec(mainScope.SQL, mainScope.SQLVars...)
if err != nil {
	msg := fmt.Sprintf("bulk update error: sql=%s, sqlVars=%+v", mainScope.SQL, mainScope.SQLVars)
	return errors.WithMessage(err, msg)
}
rows, _ := result.RowsAffected()
_, file, line, _ := runtime.Caller(2)
fileWithLineNum := fmt.Sprintf("%v:%v", file, line)
logger.LogInfo(gorm.LogFormatter(true, "sql", fileWithLineNum, time.Since(t), mainScope.SQL, mainScope.SQLVars, rows))
```

### 类型兼容

目前两个接口的签名都要求传入的 []interface{} 类型的数据，需要调用者进行类型的转换，不太友好，通过下面的方法可以将 interface{} 转化为 []interface{} 再进行操作。

```go
func BulkUpdate(db *gorm.DB, objArr interface{}, updateFields []string) (err error) {
	val := reflect.ValueOf(objArr)
	var arr []interface{}
	l := val.Len()
	for i := 0; i < l; i++ {
		e := val.Index(i)
		for e.Kind() == reflect.Ptr {
			e = e.Elem()
		}
		arr = append(arr, e.Interface())
	}
	err = bulkUpdateImplement(db, arr, updateFields)
	if err != nil {
		return 
	}
	return nil
}
```

## 参考资料
1. Golang SQL连接池梳理
[https://www.cnblogs.com/ZhuChangwu/p/13412853.html](https://www.cnblogs.com/ZhuChangwu/p/13412853.html
)
2. Gorm 源码分析(二) 简单query分析
 [https://segmentfault.com/a/1190000019490869]( https://segmentfault.com/a/1190000019490869
)
3. GORM源码阅读与分析
 [https://jiajunhuang.com/articles/2019_03_19-gorm.md.html](https://jiajunhuang.com/articles/2019_03_19-gorm.md.html)
4. Gorm 的 Create 操作 源码分析 [https://www.jianshu.com/p/f46518774267](https://www.jianshu.com/p/f46518774267)
5. GORM源码解读
 [https://juejin.cn/post/6844904033648394254](https://juejin.cn/post/6844904033648394254), [https://juejin.cn/post/6844904047774793735](https://juejin.cn/post/6844904047774793735)
6. gorm源码解读 [https://blog.csdn.net/cexo425/article/details/78831055](https://blog.csdn.net/cexo425/article/details/78831055)