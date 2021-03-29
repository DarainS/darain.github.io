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

当然实际上连接池做的事情并不止这么多，想要详细了解可以参加 [这篇文章](https://www.cnblogs.com/ZhuChangwu/p/13412853.html)。

### 事务

与连接池类似，gorm 直接复用了 *sql.Tx 来进行事务。

我们前面说了 gorm 的所有 DB 操作都是由 SQLCommon 这个interface 实现的。而这个接口实际上有两个实现：分别为 *sql.DB 和 *sql.Tx。

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