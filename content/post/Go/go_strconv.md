---
title: Go语言标准库之strconv
tags: [strconv, 标准库]
categories: [Go]
date: 2019-06-29 00:00:00
toc: true
---

`Go`语言中`strconv`包实现了基本数据类型和其字符串表示的相互转换。<!--more-->

# strconv包

`strconv`包实现了基本数据类型与其字符串表示的转换，主要有以下常用函数： `Atoi()`、`Itia()`、`parse`系列、`format`系列、`append`系列。

更多函数请查看[官方文档](https://golang.org/pkg/strconv/)。

## string与int类型转换

这一组函数是我们平时编程中用的最多的。

### Atoi()

`Atoi()`函数用于将字符串类型的整数转换为`int`类型，函数签名如下。

```go
func Atoi(s string) (i int, err error)
```

如果传入的字符串参数无法转换为`int`类型，就会返回错误。

```go
s1 := "100"
i1, err := strconv.Atoi(s1)
if err != nil {
	fmt.Println("can't convert to int")
} else {
	fmt.Printf("type:%T value:%#v\n", i1, i1) //type:int value:100
}
```

### Itoa()

`Itoa()`函数用于将int类型数据转换为对应的字符串表示，具体的函数签名如下。

```go
func Itoa(i int) string
```

示例代码如下：

```go
i2 := 200
s2 := strconv.Itoa(i2)
fmt.Printf("type:%T value:%#v\n", s2, s2) //type:string value:"200"
```

### a的典故

【扩展阅读】这是`C`语言遗留下的典故。`C`语言中没有`string`类型而是用字符数组(`array`)表示字符串，所以`Itoa`对很多`C`系的程序员很好理解。

## Parse系列函数

`Parse`类函数用于转换字符串为给定类型的值：`ParseBool()`、`ParseFloat()`、`ParseInt()`、`ParseUint()`。

### ParseBool()

```go
func ParseBool(str string) (value bool, err error)
```

返回字符串表示的`bool`值。它接受`1`、`0`、`t`、`f`、`T`、`F`、`true`、`false`、`True`、`False`、`TRUE`、`FALSE`；否则返回错误。

### ParseInt()

```go
func ParseInt(s string, base int, bitSize int) (i int64, err error)
```

返回字符串表示的整数值，接受正负号。

`base`指定进制（`2`到`36`），如果`base`为`0`，则会从字符串前置判断，”`0x`”是`16`进制，”`0`”是`8`进制，否则是`10`进制；

`bitSize`指定结果必须能无溢出赋值的整数类型，`0`、`8`、`16`、`32`、`64` 分别代表 `int`、`int8`、`int16`、`int32`、`int64`；

返回的`err`是`*NumErr`类型的，如果语法有误，`err.Error = ErrSyntax`；如果结果超出类型范围`err.Error = ErrRange`。

### ParseUnit()

```go
func ParseUint(s string, base int, bitSize int) (n uint64, err error)
```

`ParseUint`类似`ParseInt`但不接受正负号，用于无符号整型。

### ParseFloat()

```go
func ParseFloat(s string, bitSize int) (f float64, err error)
```

解析一个表示浮点数的字符串并返回其值。

如果`s`合乎语法规则，函数会返回最为接近`s`表示值的一个浮点数（使用`IEEE754`规范舍入）。

`bitSize`指定了期望的接收类型，`32`是`float32`（返回值可以不改变精确值的赋值给`float32`），`64`是`float64`；

返回值`err`是`*NumErr`类型的，语法有误的，`err.Error=ErrSyntax`；结果超出表示范围的，返回值`f`为`±Inf`，`err.Error= ErrRange`。

### 代码示例

```go
b, err := strconv.ParseBool("true")
f, err := strconv.ParseFloat("3.1415", 64)
i, err := strconv.ParseInt("-2", 10, 64)
u, err := strconv.ParseUint("2", 10, 64)
```

这些函数都有两个返回值，第一个返回值是转换后的值，第二个返回值为转化失败的错误信息。

## Format系列函数

`Format`系列函数实现了将给定类型数据格式化为`string`类型数据的功能。

### FormatBool()

```go
func FormatBool(b bool) string
```

根据b的值返回”`true`”或”`false`”。

### FormatInt()

```go
func FormatInt(i int64, base int) string
```

返回`i`的`base`进制的字符串表示。`base` 必须在`2`到`36`之间，结果中会使用小写字母’`a`’到’`z`’表示大于`10`的数字。

### FormatUint()

```go
func FormatUint(i uint64, base int) string
```

是`FormatInt`的无符号整数版本。

### FormatFloat()

```go
func FormatFloat(f float64, fmt byte, prec, bitSize int) string
```

函数将浮点数表示为字符串并返回。

`bitSize`表示`f`的来源类型（`32：float32、64：float64`），会据此进行舍入。

`fmt`表示格式：’`f`’（`-ddd.dddd`）、’`b`’（`-ddddp±ddd`，指数为二进制）、’`e`’（`-d.dddde±dd`，十进制指数）、’`E`’（`-d.ddddE±dd`，十进制指数）、’`g`’（指数很大时用’`e`’格式，否则’`f`’格式）、’`G`’（指数很大时用’`E`’格式，否则’`f`’格式）。

`prec`控制精度（排除指数部分）：对’`f`’、’`e`’、’`E`’，它表示小数点后的数字个数；对’`g`’、’`G`’，它控制总的数字个数。如果`prec` 为`-1`，则代表使用最少数量的、但又必需的数字来表示`f`。

### 代码示例

```go
s1 := strconv.FormatBool(true)
s2 := strconv.FormatFloat(3.1415, 'E', -1, 64)
s3 := strconv.FormatInt(-2, 16)
s4 := strconv.FormatUint(2, 16)
```

## 其他

### isPrint()

```go
func IsPrint(r rune) bool
```

返回一个字符是否是可打印的，和`unicode.IsPrint`一样，`r`必须是：字母（广义）、数字、标点、符号、`ASCII`空格。

### CanBackquote()

```go
func CanBackquote(s string) bool
```

返回字符串`s`是否可以不被修改的表示为一个单行的、没有空格和`tab`之外控制字符的反引号字符串。

### 其他

除上文列出的函数外，`strconv`包中还有`Append`系列、`Quote`系列等函数。具体用法可查看[官方文档](https://golang.org/pkg/strconv/)。