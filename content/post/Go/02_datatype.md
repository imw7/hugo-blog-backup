---
title: Go语言基础之基本数据类型
tags: [基本数据类型, 基础]
categories: [Go]
date: 2019-03-03 00:00:00
toc: true
---

`Go`语言中有丰富的数据类型，除了基本的`整型`、`浮点型`、`布尔型`、`字符串`外，还有`数组`、`切片`、`结构体`、`函数`、`map`和`通道(channel)`等。`Go` 语言的基本类型和其他语言大同小异。<!--more-->

# 基本数据类型

## 整型

整型分为以下两个大类： 按长度分为：`int8`、`int16`、`int32`、`int64` 对应的无符号整型：`uint8`、`uint16`、`uint32`、`uint64`

其中，`uint8`就是我们熟知的`byte`型，`int16`对应`C`语言中的`short`型，`int64`对应`C`语言中的`long`型。

| 类型     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| `uint8`  | 无符号 `8` 位整型 (`0` 到 `255`)                             |
| `uint16` | 无符号 `16 `位整型 (`0` 到 `65535`)                          |
| `uint32` | 无符号 `32` 位整型 (`0` 到 `4294967295`)                     |
| `uint64` | 无符号 `64` 位整型 (`0` 到 `18446744073709551615`)           |
| `int8`   | 有符号 `8` 位整型 (`-128` 到 `127`)                          |
| `int16`  | 有符号 `16` 位整型 (`-32768` 到 `32767`)                     |
| `int32`  | 有符号 `32` 位整型 (`-2147483648` 到 `2147483647`)           |
| `int64`  | 有符号 `64` 位整型 (`-9223372036854775808` 到 `9223372036854775807`) |

### 特殊整型

| 类型      | 描述                                                       |
| --------- | ---------------------------------------------------------- |
| `uint`    | `32`位操作系统上就是`uint32`，`64`位操作系统上就是`uint64` |
| `int`     | `32`位操作系统上就是`int32`，`64`位操作系统上就是`int64`   |
| `uintptr` | 无符号整型，用于存放一个指针                               |

**注意：** 在使用`int`和 `uint`类型时，不能假定它是32位或64位的整型，而是考虑`int`和`uint`可能在不同平台上的差异。

**注意事项** 获取对象的长度的内建`len()`函数返回的长度可以根据不同平台的字节长度进行变化。实际使用中，切片或 `map` 的元素数量等都可以用`int`来表示。在涉及到二进制传输、读写文件的结构描述时，为了保持文件的结构不会受到不同编译目标平台字节长度的影响，不要使用`int`和 `uint`。

### 数字字面量语法（Number literals syntax）

`Go1.13`版本之后引入了数字字面量语法，这样便于开发者以二进制、八进制或十六进制浮点数的格式定义数字，例如：

`v := 0b00101101`， 代表二进制的 `101101`，相当于十进制的 `45`。 `v := 0o377`，代表八进制的 `377`，相当于十进制的 `255`。 `v := 0x1p-2`，代表十六进制的 `1`除以 `2²`，也就是 `0.25`。 而且还允许我们用 `_` 来分隔数字，比如说：

`v := 123_456` 等于 `123456`。

我们可以借助`fmt`函数来将一个整数以不同进制形式展示。

```go
package main
 
import "fmt"
 
func main(){
	// 十进制
	var a int = 10
	fmt.Printf("%d \n", a)  // 10
	fmt.Printf("%b \n", a)  // 1010  占位符%b表示二进制
 
	// 八进制  以0开头
	var b int = 077
	fmt.Printf("%o \n", b)  // 77
 
	// 十六进制  以0x开头
	var c int = 0xff
	fmt.Printf("%x \n", c)  // ff
	fmt.Printf("%X \n", c)  // FF
}
```

``` go
package main

import "fmt"

// 整型

func main() {
	// 十进制
	var i1 = 101
	fmt.Printf("%d\n", i1)
	// 101
	fmt.Printf("%b\n", i1) // 把十进制转换为二进制
	// 1100101
	fmt.Printf("%o\n", i1) // 把十进制转换为八进制
	// 145
	fmt.Printf("%x\n", i1) // 把十进制转换为十六进制
	// 65

	// 八进制
	i2 := 077
	fmt.Printf("%d\n", i2)
	// 63

	// 十六进制
	i3 := 0x1234567
	fmt.Printf("%d\n", i3)
	// 19088743

	// 查看变量的类型
	fmt.Printf("%T\n", i3)
	// int

	// 声明int8类型的变量
	i4 := int8(9) // 明确指定int8类型，否则就是默认为int类型
	fmt.Printf("%T\n", i4)
	// int8
}
```

## 浮点型

`Go`语言支持两种浮点型数：`float32`和`float64`。这两种浮点型数据格式遵循`IEEE 754`标准： `float32` 的浮点数的最大范围约为 `3.4e38`，可以使用常量定义：`math.MaxFloat32`。 `float64` 的浮点数的最大范围约为 `1.8e308`，可以使用一个常量定义：`math.MaxFloat64`。

打印浮点数时，可以使用`fmt`包配合动词`%f`，代码如下：

```go
package main

import (
    "fmt"
    "math"
)

func main() {
    fmt.Printf("%f\n", math.Pi)
    fmt.Printf("%.2f\n", math.Pi)
}
```

```go
package main

import "fmt"

// 浮点数

func main() {
	// math.MaxFloat32 // float32最大值
	f1 := 1.23456
	fmt.Printf("%T\n", f1) // 默认Go语言中的小数都是float64类型
	f2 := float32(1.23456)
	fmt.Printf("%T\n", f2) // 显示声明float32类型
	// f1 = f2 // float32类型的值不能直接赋值给float64类型的变量
}
```

## 复数

`complex64` 和 `complex128`

```go
var c1 complex64
c1 = 1 + 2i
var c2 complex128
c2 = 2 + 3i
fmt.Println(c1)
fmt.Println(c2)
```

复数有实部和虚部，`complex64` 的实部和虚部为`32`位，`complex128`的实部和虚部为`64`位。

## 布尔值

`Go`语言中以`bool`类型进行声明布尔型数据，布尔型数据只有`true（真）`和`false（假）`两个值。

**注意：**

1. 布尔类型变量的默认值为`false`。
2. `Go` 语言中不允许将整型强制转换为布尔型。
3. 布尔型无法参与数值运算，也无法与其他类型进行转换。

```go
package main

import "fmt"

func main() {
	b1 := true
	var b2 bool // 默认是false
	fmt.Printf("%T\n", b1)
	// bool
	fmt.Printf("%T value:%v\n", b2, b2)
	// bool value:false
}
```

```go
package main

import "fmt"

// fmt占位符
func main()  {
	var n = 100
	// 查看类型
	fmt.Printf("%T\n", n)
	// int
	fmt.Printf("%v\n", n)
	// 100
	fmt.Printf("%b\n", n)
	// 1100100
	fmt.Printf("%d\n", n)
	// 100
	fmt.Printf("%o\n", n)
	// 144
	fmt.Printf("%x\n", n)
	// 64
	var s = "Hello go!"
	fmt.Printf("字符串：%s\n", s)
	// 字符串：Hello go!
	fmt.Printf("字符串：%v\n", s)
	// 字符串：Hello go!
	fmt.Printf("字符串：%#v\n", s)
	// 字符串："Hello go!"
}
```

## 字符串

`Go`语言中的字符串以原生数据类型出现，使用字符串就像使用其他原生数据类型（`int`、`bool`、`float32`、`float64` 等）一样。 `Go` 语言里的字符串的内部实现使用`UTF-8`编码。 字符串的值为`双引号(")`中的内容，可以在`Go`语言的源码中直接添加非`ASCII`码字符，例如：

```go
s1 := "hello"
s2 := "你好"
```

`Go`语言中单引号包裹的是字符！

```go
// 字符串
s := "Hello，世界"
// 单独的字母、汉字、符号表示一个字符
c1 := 'h'
c2 := '1'
c3 := '中'
// 字节：1字节=8Bit（8个二进制位）
// 1个字符'A'=1个字节
// 1个utf-8编码的汉字'中'=一般占3个字节
```

### 字符串转义符

`Go` 语言的字符串常见转义符包含回车、换行、单双引号、制表符等，如下表所示。

| 转义符 | 含义                               |
| ------ | ---------------------------------- |
| `\r`   | 回车符（返回行首）                 |
| `\n`   | 换行符（直接跳到下一行的同列位置） |
| `\t`   | 制表符                             |
| `\'`   | 单引号                             |
| `\"`   | 双引号                             |
| `\\`   | 反斜杠                             |

举个例子，我们要打印一个`Windows`平台下的一个文件路径：

```go
package main

import "fmt"

func main() {
    fmt.Println("str := \"c:\\Code\\lesson1\\go.exe\"")
}
```

### 多行字符串

`Go`语言中要定义一个多行字符串时，就必须使用`反引号`字符：

```go
s1 := `第一行
第二行
第三行
`
fmt.Println(s1)
```

反引号间换行将被作为字符串中的换行，但是所有的转义字符均无效，文本将会原样输出。

### 字符串的常用操作

| 方法                                     | 介绍           |
| ---------------------------------------- | -------------- |
| `len(str)`                               | 求长度         |
| `+` 或 `fmt.Sprintf`                     | 拼接字符串     |
| `strings.Split`                          | 分割           |
| `strings.contains`                       | 判断是否包含   |
| `strings.HasPrefix`, `strings.HasSuffix` | 前缀/后缀判断  |
| `strings.Index()`, `strings.LastIndex()` | 子串出现的位置 |
| `strings.Join(a[]string, sep string)`    | join操作       |

```go
package main

import (
	"fmt"
	"strings"
)

// 字符串

func main() {
	// \ 本来是具有特殊含义的，我应该告诉程序我写的\就是一个单纯的\
	path := "\"E:\\Go\\src\\github.com\\hugew7\\day01\""
	fmt.Println(path)
	// "E:\Go\src\github.com\hugew7\day01"

	s := "I'm ok"
	fmt.Println(s)
	// I'm ok

	// 多行的字符串
	s2 := `
		世情薄
		人情恶
		雨送黄昏花易落
	`
	fmt.Println(s2) // 原样输出

	s3 := `E:\Go\src\github.com\hugew7\day01`
	fmt.Println(s3)
	// E:\Go\src\github.com\hugew7\day01

	// 字符串相关操作
	fmt.Println(len(s3))
	// 40

	// 字符串拼接
	name := "小明"
	word := "大帅比"

	ss := name + word
	fmt.Println(ss)
	// 小明大帅比

	ss1 := fmt.Sprintf("%s%s", name, word)
	// fmt.Printf("%s%s", name, word)
	fmt.Println(ss1)
	// 小明大帅比

	// 分割
	ret := strings.Split(s3, "\\")
	fmt.Println(ret)
	// [E: Go src github.com hugew7 day01]

	// 包含
	fmt.Println(strings.Contains(ss, "小花"))
	// false
	fmt.Println(strings.Contains(ss, "小明"))
	// true

	// 前缀
	fmt.Println(strings.HasPrefix(ss, "小明"))
	// true

	// 后缀
	fmt.Println(strings.HasSuffix(ss, "小明"))
	// false

	s4 := "abcdeb"
	fmt.Println(strings.Index(s4, "c"))
	// 2
	fmt.Println(strings.LastIndex(s4, "b"))
	// 5

	// 拼接
	fmt.Println(strings.Join(ret, "+"))
	// E:+Go+src+github.com+hugew7+day01
}
```

## byte和rune类型

组成每个字符串的元素叫做“字符”，可以通过遍历或者单个获取字符串元素获得字符。 字符用单引号 `'` 包裹起来，如：

```go
var a := '中'
var b := 'x'
```

`Go` 语言的字符有以下两种：

1. `uint8`类型，或者叫 `byte` 型，代表了`ASCII码`的一个字符。
2. `rune`类型，代表一个 `UTF-8字符`。

当需要处理中文、日文或者其他复合字符时，则需要用到`rune`类型。`rune`类型实际是一个`int32`。

`Go` 使用了特殊的 `rune `类型来处理 `Unicode`，让基于 `Unicode` 的文本处理更为方便，也可以使用 `byte` 型进行默认字符串处理，性能和扩展性都有照顾。

```go
// 遍历字符串
func traversalString() {
	s := "hello北京"
	for i := 0; i < len(s); i++ { //byte
		fmt.Printf("%v(%c) ", s[i], s[i])
	}
	fmt.Println()
	for _, r := range s { //rune
		fmt.Printf("%v(%c) ", r, r)
	}
	fmt.Println()
}
```

输出：

```bash
104(h) 101(e) 108(l) 108(l) 111(o) 229(å) 140(Œ) 151(—) 228(ä) 186(º) 172(¬) 
104(h) 101(e) 108(l) 108(l) 111(o) 21271(北) 20140(京) 
```

因为`UTF8`编码下一个中文汉字由`3~4`个字节组成，所以我们不能简单的按照字节去遍历一个包含中文的字符串，否则就会出现上面输出中第一行的结果。

字符串底层是一个`byte`数组，所以可以和 `[]byte` 类型相互转换。字符串是不能修改的 字符串是由`byte`字节组成，所以字符串的长度是 `byte` 字节的长度。 `rune` 类型用来表示 `utf8` 字符，一个 `rune` 字符由一个或多个 `byte` 组成。

### 修改字符串

要修改字符串，需要先将其转换成 `[]rune`或`[]byte`，完成后再转换为 `string`。无论哪种转换，都会重新分配内存，并复制字节数组。

```go
func changeString() {
	s1 := "big"
	// 强制类型转换
	byteS1 := []byte(s1)
	byteS1[0] = 'p'
	fmt.Println(string(byteS1))

	s2 := "白萝卜"
	runeS2 := []rune(s2)
	runeS2[0] = '红'
	fmt.Println(string(runeS2))
}
```

## 类型转换

`Go`语言中只有强制类型转换，没有隐式类型转换。该语法只能在两个类型之间支持相互转换的时候使用。

强制类型转换的基本语法如下：

```bash
T(表达式)
```

其中，`T`表示要转换的类型。表达式包括变量、复杂算子和函数返回值等.

比如计算直角三角形的斜边长时使用 `math` 包的 `Sqrt()` 函数，该函数接收的是 `float64` 类型的参数，而变量 `a` 和 `b` 都是 `int` 类型的，这个时候就需要将 `a` 和 `b` 强制类型转换为 `float64` 类型。

```go
func sqrtDemo() {
	var a, b = 3, 4
	var c int
	// math.Sqrt()接收的参数是float64类型，需要强制转换
	c = int(math.Sqrt(float64(a*a + b*b)))
	fmt.Println(c)
}
```

```go
package main

import "fmt"

// byte和rune类型

// Go语言中为了处理非ASCII码类型的字符 定义了新的rune类型

func main() {
	s := "Hello go 你好世界"
	// len()求的是byte字节的数量
	n := len(s)    // 求字符串s的长度，把长度保存到变量n中
	fmt.Println(n) // 21

	// for i := 0; i < len(s); i++ {
	// 	// fmt.Println(s[i])
	// 	fmt.Printf("%c\n", s[i]) // %c：字符
	// }

	for _, c := range s { // 从字符串中拿出具体的字符
		fmt.Printf("%c\n", c) // %c：字符
	}

	// "Hello" => 'H' 'e' 'l' 'l' 'o'
	// 字符串修改
	s2 := "白萝卜"      // '白' '萝' '卜'
	s3 := []rune(s2) // 把字符串强制转换成了一个rune切片
	s3[0] = '红'
	fmt.Println(string(s3)) // 把rune切片强制转换成字符串
	// 红萝卜

	c1 := "红"
	c2 := '红' // rune(int32)
	fmt.Printf("c1:%T c2:%T\n", c1, c2)
	// c1:string c2:int32
	c3 := "H" // string
	c4 := 'H' // int32
	fmt.Printf("c3:%T c4:%T\n", c3, c4)
	// c3:string c4:int32
	fmt.Printf("%d\n", c4)
	// 72

	// 类型转换
	n1 := 10 // int
	var f float64
	f = float64(n1)
	fmt.Println(f)
	// 10
	fmt.Printf("%T\n", f)
	// float64
}
```

# 练习题

1.编写代码分别定义一个整型、浮点型、布尔型、字符串型变量，使用`fmt.Printf()`搭配`%T`分别打印出上述变量的值和类型。

```go
func dataType() {
	n1 := 27
	n2 := 3.1416926535
	n3 := true
	n4 := "hello go world"
	fmt.Printf("n1 = {值: %d, 类型: %T}\n", n1, n1)
	fmt.Printf("n2 = {值: %f, 类型: %T}\n", n2, n2)
	fmt.Printf("n3 = {值: %v, 类型: %T}\n", n3, n3)
	fmt.Printf("n4 = {值: %s, 类型: %T}\n", n4, n4)
}
```

输出结果：

```bash
n1 = {值: 27, 类型: int}
n2 = {值: 3.141693, 类型: float64}
n3 = {值: true, 类型: bool}
n4 = {值: hello go world, 类型: string}
```

2.编写代码统计出字符串`"Hello，世界한국어にほんご！"`中汉字的数量。

```go
func chineseCharacterCounts() {
    // 判断字符串中汉字的数量
	// 难点是判断一个字符是汉字
	sentence := "Hello，世界こんにちは！"
    // 1>依次拿到字符串中的字符
	count := 0
	for _, c := range sentence {
        // 2>判断当前这个字符是不是汉字
		if unicode.Is(unicode.Han, c) { // 平假名：unicode.Hiragana 片假名：unicode.Katakana
            // 3>把汉字出现的次数累加得到最终结果
			count++
		}
	}
	fmt.Printf("%#v中汉字的个数为：%d", sentence, count)
}
```

输出结果：

```bash
"Hello，世界こんにちは！"中汉字的个数为：2
```

3.打印99乘法表。

```go
func multiTable() {
	for x := 1; x < 10; x++ {
		for y := 1; y <= x; y++ {
			fmt.Printf("%d*%d=%d\t", y, x, x*y)
		}
		fmt.Println()
	}
}
```

输出结果：

```bash
1*1=1   
1*2=2   2*2=4   
1*3=3   2*3=6   3*3=9   
1*4=4   2*4=8   3*4=12  4*4=16  
1*5=5   2*5=10  3*5=15  4*5=20  5*5=25  
1*6=6   2*6=12  3*6=18  4*6=24  5*6=30  6*6=36  
1*7=7   2*7=14  3*7=21  4*7=28  5*7=35  6*7=42  7*7=49  
1*8=8   2*8=16  3*8=24  4*8=32  5*8=40  6*8=48  7*8=56  8*8=64  
1*9=9   2*9=18  3*9=27  4*9=36  5*9=45  6*9=54  7*9=63  8*9=72  9*9=81 
```