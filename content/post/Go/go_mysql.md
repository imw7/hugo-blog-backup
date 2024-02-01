---
title: Go语言操作MySQL
tags: [MySQL, 数据库]
categories: [Go]
date: 2019-07-13 00:00:00
toc: true
---

`MySQL`是业界常用的关系型数据库，本文介绍了`Go`语言如何操作`MySQL`数据库。<!--more-->

# Go操作MySQL

## 连接

`Go`语言中的`database/sql`包提供了保证`SQL`或类`SQL`数据库的泛用接口，并不提供具体的数据库驱动。使用`database/sql`包时必须注入（至少）一个数据库驱动。

我们常用的数据库基本上都有完整的第三方实现。例如：[MySQL驱动](https://github.com/go-sql-driver/mysql)

### 下载依赖

```bash
go get -u github.com/go-sql-driver/mysql
```

### 使用MySQL驱动

```go
func Open(driverName, dataSourceName string) (*DB, error)
```

`Open`打开一个`dirverName`指定的数据库，`dataSourceName`指定数据源，一般至少包括数据库文件名和其它连接必要的信息。

```go
import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
    // DSN:Data Source Name
    // dsn := "用户名:密码@tcp(ip:端口)/数据库的名字"
	dsn := "user:password@tcp(127.0.0.1:3306)/dbname"
	db, err := sql.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }() // 注意这行代码要写在上面err判断的下面
}
```

**思考题**： 为什么上面代码中的`defer db.Close()`语句不应该写在`if err != nil`的前面呢？

### 初始化连接

`Open`函数可能只是验证其参数格式是否正确，实际上并不创建与数据库的连接。如果要检查数据源的名称是否真实有效，应该调用`Ping`方法。

返回的`DB`对象可以安全地被多个`goroutine`并发使用，并且维护其自己的空闲连接池。因此，`Open`函数应该仅被调用一次，很少需要关闭这个`DB`对象。

```go
// 定义一个全局对象db
var db *sql.DB

// 定义一个初始化数据库的函数
func initDB() (err error) {
	// DSN:Data Source Name
	dsn := "user:password@tcp(127.0.0.1:3306)/sql_test?charset=utf8mb4&parseTime=True"
	// 不会校验账号密码是否正确
	// 注意！！！这里不要使用:=，我们是给全局变量赋值，然后在main函数中使用全局变量db
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err
	}
	// 尝试与数据库建立连接（校验dsn是否正确）
	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}

func main() {
	err := initDB() // 调用输出化数据库的函数
	if err != nil {
		fmt.Println("init database failed, err:", err)
		return
	}
}
```

其中`sql.DB`是一个数据库（操作）句柄，代表一个具有零到多个底层连接的连接池。它可以安全地被多个`goroutine`同时使用。`database/sql`包会自动创建和释放连接；它也会维护一个闲置连接的连接池。

### SetMaxOpenConns

```go
func (db *DB) SetMaxOpenConns(n int)
```

`SetMaxOpenConns`设置与数据库建立连接的最大数目。 如果`n`大于`0`且小于最大闲置连接数，会将最大闲置连接数减小到匹配最大开启连接数的限制。 如果`n<=0`，不会限制最大开启连接数，默认为`0`（无限制）。

### SetMaxIdleConns

```go
func (db *DB) SetMaxIdleConns(n int)
```

`SetMaxIdleConns`设置连接池中的最大闲置连接数。 如果`n`大于最大开启连接数，则新的最大闲置连接数会减小到匹配最大开启连接数的限制。 如果`n<=0`，不会保留闲置连接。

## CRUD

### 建库建表

我们先在`MySQL`中创建一个名为`sql_test`的数据库

```sql
CREATE DATABASE sql_test;
```

进入该数据库:

```sql
use sql_test;
```

执行以下命令创建一张用于测试的数据表：

```sql
CREATE TABLE `user` (
    `id` BIGINT(20) NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(20) DEFAULT '',
    `age` INT(11) DEFAULT '0',
    PRIMARY KEY(`id`)
)ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;
```

### 查询

#### 单行查询

单行查询`db.QueryRow()`执行一次查询，并期望返回最多一行结果（即`Row`）。`QueryRow`总是返回非`nil`的值，直到返回值的`Scan`方法被调用时，才会返回被延迟的错误。（如：未找到结果）

```go
func (db *DB) QueryRow(query string, args ...interface{}) *Row
```

具体示例代码：

```go
// 查询单条数据示例
func queryRowDemo() {
	sqlStr := "select id, name, age from user where id=?"
	var u user
	// 非常重要：确保QueryRow之后调用Scan方法，否则持有的数据库链接不会被释放
	err := db.QueryRow(sqlStr, 1).Scan(&u.id, &u.name, &u.age)
	if err != nil {
		fmt.Println("scan failed, err:", err)
		return
	}
	fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
}
```

#### 多行查询

多行查询`db.Query()`执行一次查询，返回多行结果（即`Rows`），一般用于执行`select`命令。参数`args`表示`query`中的占位参数。

```go
func (db *DB) Query(query string, args ...interface{}) (*Rows, error)
```

具体示例代码：

```go
// 查询多条数据示例
func queryMultiRowDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	rows, err := db.Query(sqlStr, 0)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	// 非常重要：关闭rows释放持有的数据库链接
    defer func() { _ = rows.Close() }

	// 循环读取结果集中的数据
	for rows.Next() {
		var u user
		err := rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Println("scan failed, err:", err)
			return
		}
		fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
	}
}
```

### 插入数据

插入、更新和删除操作都使用`Exec`方法。

```go
func (db *DB) Exec(query string, args ...interface{}) (Result, error)
```

`Exec`执行一次命令（包括查询、删除、更新、插入等），返回的`Result`是对已执行的`SQL`命令的总结。参数`args`表示`query`中的占位参数。

具体插入数据示例代码如下：

```go
// 插入数据
func insertRowDemo() {
	sqlStr := "insert into user(name, age) values (?,?)"
	ret, err := db.Exec(sqlStr, "王五", 38)
	if err != nil {
		fmt.Println("insert failed, err:", err)
		return
	}
	theID, err := ret.LastInsertId() // 新插入数据的id
	if err != nil {
		fmt.Println("get lastinsert ID failed, err:", err)
		return
	}
	fmt.Printf("insert success, the id is %d.\n", theID)
}
```

### 更新数据

具体更新数据示例代码如下：

```go
// 更新数据
func updateRowDemo() {
	sqlStr := "update user set age=? where id = ?"
	ret, err := db.Exec(sqlStr, 39, 3)
	if err != nil {
		fmt.Println("update failed, err:", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Println("get RowsAffected failed, err:", err)
		return
	}
	fmt.Println("update success, affected rows:%d", n)
}
```

### 删除数据

具体删除数据的示例代码如下：

```go
// 删除数据
func deleteRowDemo() {
	sqlStr := "delete from user where id = ?"
	ret, err := db.Exec(sqlStr, 3)
	if err != nil {
		fmt.Println("delete failed, err:", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Println("get RowsAffected failed, err:", err)
		return
	}
	fmt.Println("delete success, affected rows:", n)
}
```

## MySQL预处理

### 什么是预处理？

普通`SQL`语句执行过程：

1. 客户端对`SQL`语句进行占位符替换得到完整的`SQL`语句。
2. 客户端发送完整`SQL`语句到`MySQL`服务端
3. `MySQL`服务端执行完整的`SQL`语句并将结果返回给客户端。

预处理执行过程：

1. 把`SQL`语句分成两部分，命令部分与数据部分。
2. 先把命令部分发送给`MySQL`服务端，`MySQL`服务端进行`SQL`预处理。
3. 然后把数据部分发送给`MySQL`服务端，`MySQL`服务端对`SQL`语句进行占位符替换。
4. `MySQL`服务端执行完整的`SQL`语句并将结果返回给客户端。

### 为什么要预处理？

1. 优化`MySQL`服务器重复执行`SQL`的方法，可以提升服务器性能，提前让服务器编译，一次编译多次执行，节省后续编译的成本。
2. 避免`SQL`注入问题。

### Go实现MySQL预处理

`Go`中的

```go
func (db *DB) Prepare(query string) (*Stmt, error)
```

`Prepare`方法会先将`sql`语句发送给`MySQL`服务端，返回一个准备好的状态用于之后的查询和命令。返回值可以同时执行多个查询和命令。

查询操作的预处理示例代码如下：

```go
// 预处理查询示例
func prepareQueryDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Println("prepare failed, err:", err)
		return
	}
	defer stmt.Close()
	rows, err := stmt.Query(0)
	if err != nil {
		fmt.Println("query failed, err:", err)
		return
	}
	defer rows.Close()
	// 循环读取结果集中的数据
	for rows.Next() {
		var u user
		err := rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Println("scan failed, err:", err)
			return
		}
		fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
	}
}
```

插入、更新和删除操作的预处理十分类似，这里以插入操作的预处理为例：

```go
// 预处理插入示例
func prepareInsertDemo() {
	sqlStr := "insert into user(name, age) values (?, ?)"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Println("prepare failed, err:", err)
		return
	}
	defer stmt.Close()
	_, err = stmt.Exec("小王子", 18)
	if err != nil {
		fmt.Println("insert failed, err:", err)
		return
	}
	_, err = stmt.Exec("张三", 18)
	if err != nil {
		fmt.Println("insert failed, err:", err)
		return
	}
	fmt.Println("insert success.")
}
```

## Go实现MySQL事务

### 什么是事务？

事务：一个最小的不可再分的工作单元；通常一个事务对应一个完整的业务(例如银行账户转账业务，该业务就是一个最小的工作单元)，同时这个完整的业务需要执行多次的`DML(insert、update、delete)`语句共同联合完成。`A`转账给`B`，这里面就需要执行两次`update`操作。

在`MySQL`中只有使用了`Innodb`数据库引擎的数据库或表才支持事务。事务处理可以用来维护数据库的完整性，保证成批的`SQL`语句要么全部执行，要么全部不执行。

### 事务的ACID

通常事务必须满足4个条件（`ACID`）：原子性（`Atomicity`，或称不可分割性）、一致性（`Consistency`）、隔离性（`Isolation`，又称独立性）、持久性（`Durability`）。

|  条件  |                             解释                             |
| :----: | :----------------------------------------------------------: |
| 原子性 | 一个事务（`transaction`）中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚（`Rollback`）到事务开始前的状态，就像这个事务从来没有执行过一样。 |
| 一致性 | 在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设规则，这包含资料的精确度、串联性以及后续数据库可以自发性地完成预定的工作。 |
| 隔离性 | 数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。事务隔离分为不同级别，包括读未提交（`read uncommitted`）、读提交（`read committed`）、可重复读（`repeatable read`）和串行化（`erializable`）。 |
| 持久性 | 事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。 |

### 事务相关方法

`Go`语言中使用以下三个方法实现`MySQL`中的事务操作。 开始事务

```go
func (db *DB) Begin() (*Tx, error)
```

提交事务

```go
func (tx *Tx) Commit() error
```

回滚事务

```go
func (tx *Tx) Rollback() error
```

### 事务示例

下面的代码演示了一个简单的事务操作，该事物操作能够确保两次更新操作要么同时成功要么同时失败，不会存在中间状态。

```go
// 事务操作示例
func transactionDemo() {
	tx, err := db.Begin() // 开启事务
	if err != nil {
		if tx != nil {
			tx.Rollback() // 回滚
		}
		fmt.Println("begin trans failed, err:", err)
		return
	}
	sqlStr1 := "Update user set age=30 where id=?"
	_, err = tx.Exec(sqlStr1, 2)
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Println("exec sql1 failed, err:", err)
		return
	}
	sqlStr2 := "Update user set age=40 where id=?"
	_, err = tx.Exec(sqlStr2, 4)
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Println("exec sql2 failed, err:", err)
		return
	}
	err = tx.Commit() // 提交事务
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Println("commit failed, err:", err)
		return
	}
	fmt.Println("exec trans success!")
}
```

# sqlx使用

第三方库`sqlx`能够简化操作，提高开发效率。

## 安装

```go
go get github.com/jmoiron/sqlx
```

## 基本使用

### 连接数据库

```go
var db *sqlx.DB

func initDB() (err error) {
	dsn := "user:password@tcp(127.0.0.1:3306)/sql_test?charset=utf8mb4&parseTime=True"
	// 也可以使用MustConnect连接不成功就panic
	db, err = sqlx.Connect("mysql", dsn)
	if err != nil {
		fmt.Println("connect database failed, err:", err)
		return
	}
	db.SetMaxOpenConns(20)
	db.SetMaxIdleConns(10)
	return
}
```

### 查询

查询单行数据示例代码如下：

```go
// 查询单条数据示例
func queryRowDemo() {
	sqlStr := "select id, name, age from user where id=?"
	var u user
	err := db.Get(&u, sqlStr, 1)
	if err != nil {
		fmt.Println("get failed, err:", err)
		return
	}
	fmt.Printf("id:%d name:%s age:%d\n", u.ID, u.Name, u.Age)
}
```

查询多行数据示例代码如下：

```go
// 查询多条数据示例
func queryMultiRowDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	var users []user
	err := db.Select(&users, sqlStr, 0)
	if err != nil {
		fmt.Println("query failed, err:", err)
		return
	}
	fmt.Printf("users:%#v\n", users)
}
```

### 插入、更新和删除

`sqlx`中的`exec`方法与原生`sql`中的`exec`使用基本一致：

```go
// 插入数据
func insertRowDemo() {
	sqlStr := "insert into user(name, age) values (?,?)"
	ret, err := db.Exec(sqlStr, "Jesse", 19)
	if err != nil {
		fmt.Println("insert failed, err:", err)
		return
	}
	theID, err := ret.LastInsertId() // 新插入数据的id
	if err != nil {
		fmt.Println("get lastinsert ID failed, err:", err)
		return
	}
	fmt.Printf("insert success, the id is %d.\n", theID)
}

// 更新数据
func updateRowDemo() {
	sqlStr := "update user set age=? where id = ?"
	ret, err := db.Exec(sqlStr, 39, 6)
	if err != nil {
		fmt.Println("update failed, err:", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Println("get RowsAffected failed, err:", err)
		return
	}
	fmt.Println("update success, affected rows:", n)
}

// 删除数据
func deleteRowDemo() {
	sqlStr := "delete from user where id = ?"
	ret, err := db.Exec(sqlStr, 6)
	if err != nil {
		fmt.Println("delete failed, err:", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Println("get RowsAffected failed, err:", err)
		return
	}
	fmt.Println("delete success, affected rows:", n)
}
```

### NamedExec

```go
func insertUserDemo()(err error){
	_, err = db.NamedExec(`INSERT INTO users (name, age) VALUES (:name, :age)`,
		map[string]interface{}{
			"name": "Eric",
			"age": 28,
		})
	return
}
```

### NamedQuery

```go
// 使用map做命名查询
rows, err = db.NamedQuery(`SELECT * FROM users WHERE name=:name`, map[string]interface{}{"name": "Eric"})

type User struct {
	Name string `db:"name"`
	Age  int    `db:"age"`
}
u := User{
	Name: "Eric",
}
// 使用结构体命名查询，根据结构体字段的 db tag进行映射
rows, err = db.NamedQuery(`SELECT * FROM users WHERE name=:name`, u)
```

### 事务操作

对于事务操作，我们可以使用`sqlx`中提供的`db.Beginx()`和`tx.Exec()`方法。示例代码如下：

```go
func transactionDemo2()(err error) {
	tx, err := db.Beginx() // 开启事务
	if err != nil {
		fmt.Printf("begin trans failed, err:", err)
		return err
	}
	defer func() {
		if p := recover(); p != nil {
			tx.Rollback()
			panic(p) // re-throw panic after Rollback
		} else if err != nil {
			fmt.Println("rollback")
			tx.Rollback() // err is non-nil; don't change it
		} else {
			err = tx.Commit() // err is nil; if Commit returns error update err
			fmt.Println("commit")
		}
	}()

	sqlStr1 := "Update user set age=20 where id=?"

	rs, err := tx.Exec(sqlStr1, 1)
	if err!= nil{
		return err
	}
	n, err := rs.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("exec sqlStr1 failed")
	}
	sqlStr2 := "Update user set age=50 where i=?"
	rs, err = tx.Exec(sqlStr2, 5)
	if err!=nil{
		return err
	}
	n, err = rs.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("exec sqlStr1 failed")
	}
	return err
}
```

# 注意事项

## SQL中的占位符

不同的数据库中，`SQL`语句使用的占位符语法不尽相同。

|   数据库   |  占位符语法  |
| :--------: | :----------: |
|   MySQL    |     `?`      |
| PostgreSQL | `$1`, `$2`等 |
|   SQLite   |  `?` 和`$1`  |
|   Oracle   |   `:name`    |

## SQL注入

**我们任何时候都不应该自己拼接`SQL`语句！**

这里我们演示一个自行拼接`SQL`语句的示例，编写一个根据`name`字段查询`user`表的函数如下：

```go
// sql注入示例
func sqlInjectDemo(name string) {
	sqlStr := fmt.Sprintf("select id, name, age from user where name='%s'", name)
	fmt.Printf("SQL:%s\n", sqlStr)

	var users []user
	err := db.Select(&users, sqlStr)
	if err != nil {
		fmt.Printf("exec failed, err:%v\n", err)
		return
	}
	for _, u := range users {
		fmt.Printf("user:%#v\n", u)
	}
}
```

此时以下输入字符串都可以引发`SQL`注入问题：

```go
sqlInjectDemo("xxx' or 1=1#")
sqlInjectDemo("xxx' union select * from user #")
sqlInjectDemo("xxx' and (select count(*) from user) <10 #")
```

# 练习题

结合`net/http`和`database/sql`实现一个使用`MySQL`存储用户信息的注册及登陆的简易`web`程序。