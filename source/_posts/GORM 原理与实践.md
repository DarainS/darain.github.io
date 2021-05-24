---
title: GORM 原理与实践
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


## 架构设计

GORM 采用 callback 设计作为数据处理的核心。Callback 本质是一种面向过程的编程思想，类似于请求的中间件和 Java 中的 AOP 技术。

这样做的好处是完全解耦业务逻辑，每一层的 callback 可以完全忽略另一层 callback 的存在，而只处理自己的逻辑。

一个典型的查询流程如下：

![20210329164641](https://raw.githubusercontent.com/DarainS/Images/main/images/20210329164641.png)

GORM 将具体的 DB 做操，如拼接 SQL、执行 SQL、写回结果等操作都放到了 Callback 中。

## 核心结构体分析

### *gorm.Scope

Scope 是一次查询的的全局管家，持有本次查询的所有上下文信息，包括 db 连接、结构体信息、查询条件、查询参数、查询结果等。
其中比较重要的字段有：

- Search：持有所有查询条件，包括 join、group 等。
- SQLVars：查询的参数
- Value：传入的结构体
- db：gorm 自己封装的 DB，持有着数据库连接

```go
// Scope contain current operation's information when you perform
// any operation on the database
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
- logger：日志的记录者
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

## 主要功能

### 连接池

gorm 并没有实现自己的连接池，而是直接使用的 *sql.DB 的连接池。

*sql.DB 获取连接的方法如下:

```go
// conn returns a newly-opened or cached *driverConn.
func (db *DB) conn(ctx context.Context, strategy connReuseStrategy) (*driverConn, error) {
	db.mu.Lock()
	if db.closed {
		db.mu.Unlock()
		return nil, errDBClosed
	}
	// Check if the context is expired.
	select {
	default:
	case <-ctx.Done():
		db.mu.Unlock()
		return nil, ctx.Err()
	}
	lifetime := db.maxLifetime

	// Prefer a free connection, if possible.
	numFree := len(db.freeConn)
	if strategy == cachedOrNewConn && numFree > 0 {
		conn := db.freeConn[0]
		copy(db.freeConn, db.freeConn[1:])
		db.freeConn = db.freeConn[:numFree-1]
		conn.inUse = true
		db.mu.Unlock()
		if conn.expired(lifetime) {
			conn.Close()
			return nil, driver.ErrBadConn
		}
		// Lock around reading lastErr to ensure the session resetter finished.
		conn.Lock()
		err := conn.lastErr
		conn.Unlock()
		if err == driver.ErrBadConn {
			conn.Close()
			return nil, driver.ErrBadConn
		}
		return conn, nil
	}

	// Out of free connections or we were asked not to use one. If we're not
	// allowed to open any more connections, make a request and wait.
	if db.maxOpen > 0 && db.numOpen >= db.maxOpen {
		// Make the connRequest channel. It's buffered so that the
		// connectionOpener doesn't block while waiting for the req to be read.
		req := make(chan connRequest, 1)
		reqKey := db.nextRequestKeyLocked()
		db.connRequests[reqKey] = req
		db.waitCount++
		db.mu.Unlock()

		waitStart := time.Now()

		// Timeout the connection request with the context.
		select {
		case <-ctx.Done():
			// Remove the connection request and ensure no value has been sent
			// on it after removing.
			db.mu.Lock()
			delete(db.connRequests, reqKey)
			db.mu.Unlock()

			atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

			select {
			default:
			case ret, ok := <-req:
				if ok && ret.conn != nil {
					db.putConn(ret.conn, ret.err, false)
				}
			}
			return nil, ctx.Err()
		case ret, ok := <-req:
			atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

			if !ok {
				return nil, errDBClosed
			}
			if ret.err == nil && ret.conn.expired(lifetime) {
				ret.conn.Close()
				return nil, driver.ErrBadConn
			}
			if ret.conn == nil {
				return nil, ret.err
			}
			// Lock around reading lastErr to ensure the session resetter finished.
			ret.conn.Lock()
			err := ret.conn.lastErr
			ret.conn.Unlock()
			if err == driver.ErrBadConn {
				ret.conn.Close()
				return nil, driver.ErrBadConn
			}
			return ret.conn, ret.err
		}
	}

	db.numOpen++ // optimistically
	db.mu.Unlock()
	ci, err := db.connector.Connect(ctx)
	if err != nil {
		db.mu.Lock()
		db.numOpen-- // correct for earlier optimism
		db.maybeOpenNewConnections()
		db.mu.Unlock()
		return nil, err
	}
	db.mu.Lock()
	dc := &driverConn{
		db:        db,
		createdAt: nowFunc(),
		ci:        ci,
		inUse:     true,
	}
	db.addDepLocked(dc, dc)
	db.mu.Unlock()
	return dc, nil
}
```

这段代码的逻辑相对清晰：

1. 当有空闲连接时，直接返回空闲连接
2. 没有空闲连接，而且不允许获取新连接，阻塞并一直尝试获取连接直到超时
3. 没有空闲连接，但是允许获取新连接时，获取新连接并返回

当然实际上连接池做的事情并不止这些，想要详细了解可以参加 [Golang SQL连接池梳理](https://www.cnblogs.com/ZhuChangwu/p/13412853.html)。

### 事务

与连接池类似，gorm 直接复用了 *sql.Tx 来进行事务。

我们前面说了 gorm 的所有 DB 操作都是由 SQLCommon 这个 interface 来具体执行的的。而这个接口实际上有两个实现：分别为 *sql.DB 和 *sql.Tx。

其中 *sql.DB 实际上是全局唯一的，即所有 *gorm.DB 果没有开启事务，那么都会由同一个 *sql.DB 来执行。并且由 *sql.DB 来管理连接池。

而当开启事务时，*gorm.DB 持有的 SQLCommon 就由 *sql.DB 变为了 *sql.Tx。由全局唯一的连接池变为了一个具体的连接。

```go
func (db *DB) begin(ctx context.Context, opts *TxOptions, strategy connReuseStrategy) (tx *Tx, err error) {
	dc, err := db.conn(ctx, strategy)
	if err != nil {
		return nil, err
	}
    // 从连接池返回了实际的连接
	return db.beginDC(ctx, dc, dc.releaseConn, opts)
}
```

gorm 通过SQLCommon 屏蔽了底层的 DB 的具体实现，使得事务对其操作都是无感知的。


### 主要查询与实现

接下来我们会介绍增删改查等操作的具体流程。

都是通过 callback 来实现的，gorm 默认的 callback 可以在 callback.go/DefaultCallback 变量的调用处查看。

#### First() 与 Scan() 方法

以下是 GORM 进行 First() 查询的核心流程。

![20210329202126](https://raw.githubusercontent.com/DarainS/Images/main/images/20210329202126.png)

具体步骤为:

1. 注入结构体 user
2. 添加查询条件 where...
3. 调用 callbacks

其中最核心的 callback 为 queryCallback(callback_query.go 文件中)。

这个 Callback 只做了三件事：

1. 拼接 SQL
2. 执行 SQL
3. 写回结果

其中拼接 SQL 是调用的是 prepareQuerySQL 方法，scope 会根据之前设置的查询条件和查询参数拼接出可执行的 SQL。

```go
func (scope *Scope) prepareQuerySQL() {
	if scope.Search.raw {
		scope.Raw(scope.CombinedConditionSql())
	} else {
		scope.Raw(fmt.Sprintf("SELECT %v FROM %v %v", scope.selectSQL(), scope.QuotedTableName(), scope.CombinedConditionSql()))
	}
	return
}
```

执行 SQL 则是直接调用 SQLCommon 接口的 Query 方法。

写回结果则是将 DB 返回的列名和数据使用反射写入到存储结果的容器中。

Scan() 方法与 First() 方法类似，主要的区别在于 Scan 方法会获取多行结果反射到传入的 slice 中。

#### 其它 方法

实际上其它方法的核心与 First() 方法是完全类似的，只是具体细节不同。
本质上都是分为拼接 SQl，执行 SQL，反射结果三部分。

这部分内容偏业务细节，就不再详细阐述了，想要了解可以查看对应的 Callback 代码。

## 团队实践

在使用 gorm 的过程中，我们总结了一些相对有效的实践，在此分享。

需要注意的是，为了便于阅读，我们将本文贴出的部分代码简化为了伪代码，实际代码会做兼容与边际检查以保证其健壮性，仅提供解决问题的思路，并不建议使用到生产环境。

### 指导原则

1. 采用 GORM 推荐做法
2. 降低开发者的心智负担
3. 显式优于隐式

### 事务

为了降低开发者的心智负担，以及保证代码的健壮性，避免数据库连接泄漏，我们简单封装了 gorm.DB，使得可以简化事务行为。

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
    //  这里没有兼容
    tx := s.db.Begin()
    if tx.Error != nil {
        return tx.Error
    }
    s.db = tx
	defer func() {
		if r := recover(); r != nil {
            // 如果因为 panic 而没有显示调用 Rollback 函数, 会导致 DB 连接泄漏
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
```

这种封装使得开发者不再需要显式的 begin、commit、rollback，但要求所有的 DB 操作都必须传入 *Repo，具有比较强的侵入性。

不过这样就不必再显式的操作事务，以及处理嵌套事务的问题，对于新入职的同事比较友好，是一种能够显著降低开发者心智负担的实践。


## CTime 和 MTime 的回调

按照部门的规范，所有表都应该尽量带有 ctime 和 mtime 字段。如果需要显示更新的话，一旦忘记就可以认为是代码 bug 了，对开发者十分不友好，所以我们加入了这两个回调。

```go
// 省略了注册 callback 的代码
// ...
// ...
// updateCtimeAndMtimeForCreateCallback will set `CTime`, `MTime` when creating
func updateCtimeAndMtimeForCreateCallback(scope *gorm.Scope) {
    if !scope.HasError() {
        now := time.Now().Unix()
        if createdAtField, ok := scope.FieldByName("CTime"); ok {
			if createdAtField.IsBlank {
				_ = createdAtField.Set(now)
			}
		}
		if updatedAtField, ok := scope.FieldByName("MTime"); ok {
			if updatedAtField.IsBlank {
				_ = updatedAtField.Set(now)
			}
		}
	}
}
```

## Delete time 的回调

除了 ctime 和 mtime 之外，我们还有软删除的需求。软删除后不应该再查询出对应的 rows。

我们依然采用了回调的方式来实现，但是这种行为虽然减少了代码量与开发者的心智负担，但是这确实违背了“显式优于隐式”这一原则。特别是对于新入职的同事，如果不知道我们有这种隐含行为，很容易就踩到了坑。

所以我们甚至考虑移除这种回调，只是没有做出最终决定，我们将这种方式分享出来供大家借鉴，希望大家能做出自己的权衡。


```go
// 省略了注册 callback 的代码
// ...
func ignoreSoftDeleteItems(scope *gorm.Scope) {
	if !scope.HasError() {
        // constant.QueryDeletedRows 是一个字符串常量，用来标记是否查询或更新软删除的数据。所以软删除的数据其实还可以查询出来
		if val, ok := scope.Get(constant.QueryDeletedRows); ok && val != nil {
			return
        }
        
        deletedTimeField, hasDeletedTimeField := scope.FieldByName("DeleteTime")
		if !scope.Search.Unscoped && hasDeletedTimeField {
			scope.Search.Where(fmt.Sprintf("%s = ?", scope.Quote(deletedTimeField.DBName)), 0)
		}
	}
}
```

如果想要操作已经软删除的数据的话，使用下面的代码即可：

```go
db.Set(constant.QueryDeletedRows, true).XXX()
```

### 批量分批查询

在某些业务场景，我们需要查询出某一个表的所有符合条件数据，所以我们对这种查询进行了封装。

需要特别注意的是，我们要求被查询的表必须具有 int 类型的 `id` 作为唯一键(可以不是主键)，才能够使用这个查询。


```go
// model 是结构体的指针, list 是结构体 slice 的指针
func QueryAll(query *gorm.DB, model interface{}, list interface{}) (err error) {
	if reflect.TypeOf(list).Kind() != reflect.Ptr {
		return exception.ReflectError
	}
	var (
        maxId int64 = 0
        result []interface{}
    )
    for {
        // queryPageLimit 是一个数字常量, 我们定为 5000
		err = query.Model(model).Where("id > ?", maxId).Order("id ASC", true).Limit(queryPageLimit).Scan(list).Error
		if err != nil {
			return
		}
		val := reflect.ValueOf(list)
		for val.Kind() == reflect.Ptr {
			val = val.Elem()
		}
		for i := 0; i < val.Len(); i++ {
			result = append(result, val.Index(i).Interface())
		}
		if val.Len() < queryPageLimit {
			break
        }
        // 获得最后一个查询结果, 也就是最大的 id
		lastEle := val.Index(queryPageLimit - 1).Elem()
		idVal := lastEle.FieldByName("ID")
		if idVal.IsValid() {
			maxId = int64(idVal.Uint())
		} else if idVal = lastEle.FieldByName("Id"); idVal.IsValid() {
			maxId = int64(idVal.Uint())
		} else {
            var maxIds []int64
            // 如果结构体有 id 字段这里其实不会走到
			err = query.Model(model).Where("id > ?", maxId).Order("id ASC").Limit(queryPageLimit).Pluck("id", &maxIds).Error
			if err != nil {
				return
			}
			maxId = maxIds[len(maxIds)-1]
		}
    }
	
	listVal := reflect.ValueOf(list)
	listEle := listVal.Elem()

	var isPtr bool
	// slice 元素的类型
	if typ.Elem().Elem().Kind() == reflect.Ptr {
		isPtr = true
	}
	toAdd := make([]reflect.Value, 0, len(result))
	for _, _res := range result {
		ele := reflect.ValueOf(_res).Elem()
		if isPtr {
			toAdd = append(toAdd, ele.Addr())
		} else {
			toAdd = append(toAdd, ele)
		}
	}
	listEle.Set(reflect.MakeSlice(listEle.Type(), 0, len(toAdd)))
	listEle.Set(reflect.Append(listEle, toAdd...))
	return nil
}
...

// 调用方式:
var (
    saleOrder SaleOrder
    saleOrderList []*SaleOrder
)
err = db.QueryAll(db.Model(saleOrder).Where("ctime >= ?", 160000000), &saleOrder, &saleOrderList)
if err != nil {
    // ...
}
// 这里就已经将所有查询结果写入到 saleOrderList 中, 可以使用了
for _, saleOrder := range saleOrderList {
    // ...
}
```

这段代码有三个不太好的地方，一个是对要查询的表结构和数据库字段有固定的要求，当然如果每个人都遵循团队的数据库设计规范，这个问题并不存在。

第二个地方是，为了保持分片查询的有序性，这段代码在查询时进行了强制的 reorder("id ASC") 操作，使得查询丢失了原有的排序信息。不过我们目前没有在全量查询中必须进行排序，或者说，如果我们需要对查询结果进行排序，那么我们会使用 sort 包来完成，这里影响不大。

另一个地方是它将查询结果又返回给了入参 list。一个参数等于既做了出参又做了入参，这不算是一种良好的编码方式。但是由于 golang 还不支持泛型，为了简化类型约束，便于开发者的使用，不得已而这样写了，算是一种权衡之后的结果。值得一提的是，json.Unmarshal() 方法也采用了这样的入参即出参的方式来简化使用。

### 批量创建

由于 gorm 并未提供批量创建的功能，因此我们写了一个拼接批量创建 SQL 的函数，用于批量创建操作。

### 批量更新

由于 gorm 并未提供批量更新的功能，因此我们使用 IF THEN 控制流，通过主键 id 拼接更新字段的 SQL，完成了批量更新不同数据的功能。

### 监控

通过注册 Callback，我们实现了对 DB 操作的监控。


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