+++
title='泛型'
tags=['泛型', '基础']
categories=['Go']
date="2023-12-09T11:12:58+08:00"
toc=true
draft=false

+++

千呼万唤始出来，`Go1.18`版本终于加入了对泛型的支持。这也是自`Go`语言开源以来最大的变化。让我们来一探究竟这个新特性。<!--more-->

## 什么是泛型

泛型是一种与所使用的特定类型无关的编写代码的方式。使用泛型可以编写出适用于一组类型中的任何一种函数和类型，比如`int`、`float32`可以使用一个函数或类型来定义，而不用重复定义了。

泛型给`Go`语言带来了3个新的重要特性：

1. 函数和类型的类型参数。
2. 将接口定义为类型集，包括没有方法的类型。
3. 类型推断，多数情况它允许在调用函数时省略类型参数。

## 泛型语法

### 类型参数

#### 类型形参和类型实参

我们之前已经知道函数定义时可以指定形参，函数调用时需要传入实参。

![parameter-argument](https://blog.imw7.com/images/Go/generics/parameter-argument.png)

现在，`Go`语言允许函数和类型添加类型参数。类型参数列表看起来像普通的参数列表，只是它使用方括号（`[]`）而不是圆括号（`()`）。

![type-parameter-argument](https://blog.imw7.com/images/Go/generics/type-parameter-argument.png)

借助泛型，我们可以声明一个适用于**一组类型**的`Min`函数。

```go
func Min[T int | float64](x, y T) T {
    if x < y {
        return x
    }
    return y
}
```

#### 类型实例化

这次定义的`Min`函数就同时支持`int`和`float64`两种类型了，也就是说当调用`Min`函数时，我们既可以传入`int`类型的参数。

```go
m1 := Min[int](1, 2) // 1
```

也可以传入`float64`类型的参数。

```go
m2 := Min[float64](-0.1, -0.2) // -0.2
```

向`Min`函数提供类型参数（在本例中为`int`和`float64`）称为实例化（`instantiation`）。

类型实例化分两步进行：

1. 首先，编译器在整个泛型函数或类型中将所有类型形参（`type parameters`）替换为它们各自的类型实参（`type arguments`）。
2. 其次，编译器验证每个类型参数是否满足相应的约束。

在成功实例化之后，我们将得到一个非泛型函数，它可以像任何其他函数一样被调用。例如：

```go
fmin := Min[float64]   // 类型实例化，编译器生成T=float64的Min函数
m3 := fmin(3.14, 2.71) // 2.71
```

`Min[float64]`得到的是类似我们之前定义的`minFloat64`函数——`fmin`，我们可以在函数调用中使用它。

#### 类型参数的使用

除了函数中支持使用类型参数列表外，类型也可以使用类型参数列表。

```go
type Slice[T int | string] []T

type Map[K int | string, V float32 | float64] map[K]V

type Tree[T interface{}] struct {
    left, right *Tree[T]
    value		T
}
```

在上述泛型类型中，`T`、`K`、`V`都属于类型形参，类型形参后面是类型约束，类型实参需要满足对应的类型约束。

泛型类型可以有方法，例如为上面的`Tree`实现一个查找元素的`Lookup`方法。

```go
func (t *Tree[T]) Lookup(x T) *Tree[T] {...}
```

要使用泛型类型，必须进行实例化。`Tree[string]`是使用类型实参`string`实例化`Tree`的示例。

```go
var stringTree Tree[string]
```

#### 类型约束

普通函数中的每个参数都有一个类型; 该类型定义一系列值的集合。例如，我们上面定义的非泛型函数`minFloat64`那样，声明了参数的类型为`float64`，那么在函数调用时允许传入的实际参数就必须是可以用`float64`类型表示的浮点数值。

类似于参数列表中每个参数都有对应的参数类型，类型参数列表中每个类型参数都有一个**类型约束**。类型约束定义了一个类型集——只有在这个类型集中的类型才能用作类型实参。

`Go`语言中的类型约束是接口类型。

就以上面提到的`Min`函数为例，我们来看一下类型约束常见的两种方式。

类型约束接口可以直接在类型参数列表中使用。

```go
// 类型约束字面量，通常外层interface{}可省略
func Min[T interface{ int | float64 }](a, b T) T {
	if a <= b {
		return a
	}
	return b
}
```

作为类型约束使用的接口类型可以事先定义并支持复用。

```go
// 事先定义好的类型约束类型
type Value interface {
	int | float64
}
func min[T Value](a, b T) T {
	if a <= b {
		return a
	}
	return b
}
```

在使用类型约束时，如果省略了外层的`interface{}`会引起歧义，那么就不能省略。例如：

```go
type IntPtrSlice [T *int] []T  // T*int ?

type IntPtrSlice[T *int,] []T  // 只有一个类型约束时可以添加`,`
type IntPtrSlice[T interface{ *int }] []T // 使用interface{}包裹
```

### 类型集

**`Go1.18`开始接口类型的定义也发生了改变，由过去的接口类型定义方法集（`method set`）变成了接口类型定义类型集（`type set`）**。也就是说，接口类型现在可以用作值的类型，也可以用作类型约束。

![type-set](https://blog.imw7.com/images/Go/generics/type-set.png)

把接口类型当做类型集相较于方法集有一个优势：我们可以显式地向集合添加类型，从而以新的方式控制类型集。

`Go`语言扩展了接口类型的语法，让我们能够向接口中添加类型。例如：

```go
type V interface {
    int | string | bool
}
```

上面的代码就定义了一个包括 `int`、`string` 和 `bool` 类型的类型集。

![type-set-2](https://blog.imw7.com/images/Go/generics/type-sets-2.png)

从 `Go1.18` 开始，一个接口不仅可以嵌入其他接口，还可以嵌入任何类型、类型的联合或共享相同底层类型的无限类型集合。当用作类型约束时，由接口定义的类型集精确地指定允许作为相应类型参数的类型。

* `|`符号

  `T1 | T2` 表示类型约束为 `T1` 和 `T2` 这两个类型的并集，例如下面的 `Integer` 类型表示由 `Signed` 和 `Unsigned` 组成。

  ```go
  type Integer interface {
      Signed | Unsigned
  }
  ```

* `~`符合

  `~T` 表示所有底层类型是 `T` 的类型，例如 `~string` 表示所有底层类型是 `string` 的类型集合。

  ```go
  type MyString string // MyString的底层类型是string
  ```

  **注意**：`~`符号后面只能是基本类型。

接口作为类型集是一种强大的新机制，是使类型约束能够生效的关键。目前，使用新语法表的接口只能用作类型约束。

#### any接口

空接口在类型参数列表中很常见，在 `Go1.18` 引入了一个新的预声明标识符，作为空接口类型的别名。

```go
// src/builtin/builtin.go

type any = interface{}
```

由此，我们可以使用如下代码：

```go
func foo[S ~[]E, E any]() {
    // ...
}
```

#### constrains

[`constraints包`](http://pkg.go.dev/golang.org/x/exp/constraints)提供了一些常用类型。

### 类型推断

最后一个新的主要语言特征是类型推断。从某些方面来说，这是语言中最复杂的变化，但它很重要，因为它能让人们在编写调用泛型函数的代码时更自然。

#### 函数参数类型推断

对于类型参数，需要传递类型参数，这可能导致代码冗长。回到我们通用的 `Min` 函数。

```go
func Min[T int | float64](x, y T) T {
    if x < y {
        return x
    }
    return y
}
```

类型形参 `T` 用于指定 `x` 和 `y` 的类型。我们可以使用显式类型实参调用它：

```go
var x, y, m float64
m = Min[float64](x, y) // 显式指定类型实参
```

在许多情况下，编译器可以从普通参数推断 `T` 的类型实参。这使得代码更短，同时保持清晰。

```go
var x, y, m float64
m = Min(x, y) // 无需指定类型实参
```

这种从实参的类型推断出函数的类型实参的推断称为函数实参类型推断。函数实参类型推断只适用于函数参数中使用的类型参数，而不适用于仅在函数结果中或仅在函数体中使用的类型参数。例如，它不适用于像 `MakeT [ T any ]() T` 这样的函数，因为它只使用 `T` 表示结果。

#### 约束类型推断

`Go`语言支持另一种类型推断，即*约束类型推断*。接下来我们从下面这个缩放整数的例子开始：

```go
// Scale 返回切片中每个元素都乘c的副本切片
func Scale[E constraints.Integer](s []E, c E) []E {
    r := make([]E, len(s))
    for i, v := range s {
        r[i] = v * c
    }
    return r
}
```

这是一个泛型函数适用于任何整数类型的切片。

现在假设我们有一个多维坐标的 `Point` 类型，其中每个 `Point` 只是一个给出点坐标的整数列表。这种类型通常会实现一些业务方法，这里假设它有一个 `String` 方法。

```go
type Point []int32

func (p Point) String() string {
    b, _ := json.Marshal(p)
    return string(b)
}
```

由于一个 `Point` 其实就是一个整数切片，我们可以使用前面编写的 `Scale` 函数：

```go
func ScaleAndPoint(p Point) {
    r := Scale(p, 2)
    fmt.Println(r.String()) // 编译失败
}
```

不幸的是，这代码会编译失败，输出 `r.String undefined (type []int32 has no field or method String)` 的错误。

问题是 `Scale` 函数返回类型为 `[]E` 的值，其中 `E` 是参数切片的元素类型。当我们使用 `Point` 类型的值调用 `Scale` （其基本类型为 `[]int32`）时，我们返回的是 `[]int32` 类型的值，而不是 `Point` 类型。这源于泛型代码的编写方式，但这不是我们想要的。

为了解决这个问题，我们必须更改 `Scale` 函数，以便为切片类型使用类型参数。

```go
func Scale[S ~[]E, E constraints.Integer](s S, c E) S {
    r := make(S, len(s))
    for i, v := range s {
        r[i] = v * c
    }
    return r
}
```

我们引入了一个新的类型参数 `S`，它是切片参数的类型。我们对它进行了约束，使得基础类是 `S` 而不是 `[]E`，函数返回的结果类型现在是 `S`。由于 `E` 被约束为整数，因此效果与之前相同：第一个参数必须是某个整数类型的切片。对函数体的唯一更改是，现在我们在调用 `make` 时传递 `S`，而不是 `[]E`。

现在这个 `Scale` 函数，不仅支持传入普通整数切片参数，也支持传入 `Point` 类型参数。

这里需要思考的是，为什么不传递显式类型参数就就可以写入 `Scale` 调用？也就是说，为什么我们可以写 `Scale(p, 2)`，没有类型参数，而不是必须写 `Scale[Point, int32](p, 2)`？

新 `Scale` 函数有两个类型参数——`S` 和 `E`。在不传递任何类型参数的 `Scale(p, 2)` 调用中，如上所述，函数参数类型推断让编译器推断 `S` 的类型参数是 `Point`。但是这个函数也有一个类型参数 `E`，它是乘法因子 `c` 的类型。相应的函数参数是 `2`，因为 `2` 是一个非类型化的常量，函数参数类型推断不能推断出 `E` 的正确类型（最好的情况是它可以推断出 `2` 的默认类型是 `int`，而这时错误的，因为`Point`的基础类型是 `[]int32`）。相反，编译器推断 `E` 的类型参数是切片的元素类型的过程称为**约束类型推断**。

约束类型推断从类型参数约束推断类型参数。当一个类型参数具有根据另一个类型参数定义的约束时使用。当其中一个类型参数的类型参数已知时，约束用于推断另一个类型参数的类型参数。

通常情况是，当一个约束对某种类型使用 ***`~type`*** 形式时，该类型是使用其他类型参数编写的。我们在 `Scale` 的例子中看到了这一点。`S` 是 `~[]E`，后面跟着一个用另一个类型参数写的类型 `[]E`。如果我们知道了 `S` 的类型实参，我们就可以推断出 `E` 的类型实参。 `S` 是一个切片类型，而 `E` 是该切片的元素类型。

## 总结

总之，如果你发现自己多次编写完全相同的代码，而这些代码之间的唯一区别就是使用的类型不同，这个时候你就应该考虑是否可以使用类型参数。

泛型和接口类型之间并不是替代关系，而是相辅相成的关系。泛型的引入是为了配合接口的使用，让我们能够编写更加类型安全的 `Go` 代码，并能有效地减少重复代码。