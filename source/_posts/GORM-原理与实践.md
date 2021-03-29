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

gorm 采用 callback 设计作为数据处理的核心。Callback 本质是切面的编程思想，类似于请求的中间件和 Java 中的 AOP 技术。

这样做的好处是完全解耦业务逻辑，每一层的 callback 可以完全忽略另一层 callback 的存在，而只处理自己的逻辑。

一个典型的查询流程如下：

![20210329164641](https://raw.githubusercontent.com/DarainS/Images/main/images/20210329164641.png)

GORM 将拼接 SQL、执行 SQL、写回结果等操作都放到了 Callback 中。

## 核心结构体分析

### *gorm.Scope

Scope 是一次查询的的全局管家，持有本次查询的所有信息。
其中比较重要的字段有：

- Search：持有所有查询条件，包括 join、group 等。
- SQLVars：查询的参数
- Value：传入的结构体
- db：gorm 自己封装的 DB

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

- db：实际的数据库操作执行者，实际由 *database.DB 和 *database.Tx 实现
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

ModelStruct

StructField

主要介绍这几个 struct 的字段和功能。



主要功能
连接池(复用*database/sql.DB)

事务

日志

介绍这几个功能的实现原理及核心代码。

核心函数与数据
First 方法

Scan 方法

Create 方法

Update 方法

Delete 方法

SQL 拼接

数据写回

介绍这几个函数的流程和核心代码。

团队实践
事务
delete_time 回调
批量插入
批量更新
日志及 CAT
遇到的其它问题与解决方案