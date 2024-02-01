---
title: 密码加密
tags: [密码加密, Bcrypt]
categories: [Go]
date: 2021-12-23
toc: true
---

本文介绍的密码加密方式是`bcrypt`。`bcrypt`是单向`Hash`加密算法，类似`Pbkdf2`算法，不可反向破解生成明文。<!--more-->

## Bcrypt是怎么加密的

`bcrypt`有四个变量：

1. `saltRounds`: 正数，代表`hash`杂凑次数，数值越高越安全，默认10次。
2. `myPassword`: 明文密码字符串。
3. `salt`: 盐，一个`128 bits`随机字符串，22字符。
4. `myHash`: 经过明文密码`password`和盐`salt`进行`hash`，个人的理解是默认10次下 ，循环加盐`hash`10次，得到`myHash`。

每次明文字符串`myPassword`过来，就通过10次循环加盐`salt`加密后得到`myHash`, 然后拼接`BCrypt版本号+salt盐+myHash`等得到最终的`bcrypt`密码 ，存入数据库中。
这样同一个密码，每次登录都可以根据自省业务需要生成不同的`myHash`，`myHash`中包含了版本和`salt`，存入数据库。

即使黑客得到了`bcrypt`密码，他也无法转换明文，因为之前说了`bcrypt`是`单向hash算法`；

## 如何验证密码

`bcrypt`校验时，从`myHash`中取出`salt`，`salt`跟`password`进行`hash`；得到的结果跟保存在`DB`中的`hash`进行比对。

## 使用go实现bcrypt加密密码

pkg/hash/bcrypt.go

```go
package hash

import "golang.org/x/crypto/bcrypt"

type Bcrypt struct {
	cost int
}

// Make 加密算法
func (b *Bcrypt) Make(password []byte) ([]byte, error) {
	return bcrypt.GenerateFromPassword(password, b.cost)
}

// Check 检查方法
func (b *Bcrypt) Check(hashedPassword, password []byte) error {
	return bcrypt.CompareHashAndPassword(hashedPassword, password)
}
```

pkg/hash/bcrypt_test.go

```go
package hash

import (
	"fmt"
	"golang.org/x/crypto/bcrypt"
	"testing"
)

func TestNewHash(t *testing.T) {
	// 实例化结构体
	hash := Bcrypt{
		cost: bcrypt.DefaultCost,
	}
	// 模拟密码
	password := "123456"
	// 加密
	bytes, err := hash.Make([]byte(password))
	if err != nil {
		t.Error(err)
		return
	}
	fmt.Println(string(bytes))

	// 检查明文是否为加密密码的明文
	if err = hash.Check(bytes, []byte(password)); err != nil {
		t.Error(err)
	}
	fmt.Println("密码正确")
}
```

运行结果：

```bash
/usr/local/go/bin/go tool test2json -t /tmp/GoLand/___go_test_gin_demo_pkg_hash.test -test.v -test.paniconexit0
=== RUN   TestNewHash
$2a$10$bg9jZZh8YDB1K/a12k5S/OpSK6zYUNdAuyDMDFT/wKKbPP/i9FFKm
密码正确
--- PASS: TestNewHash (0.11s)
PASS

Process finished with the exit code 0
```

