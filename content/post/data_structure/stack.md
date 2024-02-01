+++
title='栈'
tags=['栈']
categories=['数据结构']
date="2024-01-13T15:04:08+08:00"
toc=true
draft=false

+++

本文介绍数据结构——栈。<!--more-->

# 栈

## 定义

栈（`stack`）又名堆栈，它是一种运算受限的线性表。限定仅在表尾进行插入和删除操作的线性表。这一端被称为栈顶，相对地，把另一端称为栈底。向一个栈插入新元素又称作进栈、入栈或压栈，它是把新元素放到栈顶元素的上面，使之成为新的栈顶元素；从一个栈删除元素又称作出栈或退栈，它是把栈顶元素删除掉，使其相邻的元素成为新的栈顶元素。即，**先进后出，后进先出**。

![stack_1](https://blog.imw7.com/images/data_structure/stack/stack_1.png)

## 专业术语

* **栈顶**：允许进行插入和删除的一端称为栈顶
* **栈底**：表的另一端称为栈底（**第一个元素进入的位置**）
* **压栈**（入栈、出栈）：在栈顶位置插入元素的操作叫做压栈
* **出栈**（弹栈、退栈）：删除栈顶元素的操作叫做出栈
* **空栈**：不含元素的空表
* **栈溢出**：当栈满的时候，如果再有元素压栈，则发生上溢，当栈空的时候，再出栈则发生下溢

## 栈的相关操作

### 初始化

`C`语言实现：

```c
typedef struct Node {
    int data;
    struct Node *pNext;
} NODE, *PNODE;

typedef struct Stack {
    PNODE pTop;
    PNODE pBottom;
} STACK, *PSTACK;

// PSTACK 等价于 struct STACK *
void init(PSTACK pS) {
    pS->pTop = (PNODE) malloc(sizeof(NODE));
    if(NULL == pS->pTop) {
        printf("动态内存分配失败\n");
        exit(-1);
    } else {
        pS->pBottom = pS->pTop;
        pS->pTop->pNext = NULL; // pS->pBottom->pNext = NULL;
    }
}
```

`Go`语言实现：

```go
type Node struct {
	data  int
	pNext *Node
}

type Stack struct {
	pTop    *Node
	pBottom *Node
}

// Init 初始化
func Init(pS *Stack) {
	pS.pTop = new(Node)
	if nil == pS.pTop {
		fmt.Println("分配内存失败")
		os.Exit(-1)
	}
	pS.pBottom = pS.pTop
	pS.pTop.pNext = nil
}
```

### 压栈

`C`语言实现：

```c
void push(PSTACK pS, int val) {
    PNODE pNew = (PNODE) malloc(sizeof(NODE));
    pNew->data = val;
    pNew->pNext = pS->pTop; // pS->pTop不能改成pS->pBottom
    pS->pTop = pNew;
}
```

`Go`语言实现：

```go
func push(pS *Stack, val int) {
	pNew := new(Node)
	pNew.data = val
	pNew.pNext = pS.pTop
	pS.pTop = pNew
}
```

### 打印

`C`语言实现：

```c
void traverse(PSTACK pS) {
    PNODE p = pS->pTop;
    
    while (p != pS->pBottom) {
        printf("%d ", p->data);
        p = p->pNext;
    }
    printf("\n");
}
```

`Go`语言实现：

```go
func traverse(pS *Stack) {
	p := pS.pTop

	for p != pS.pBottom {
		fmt.Printf("%d ", p.data)
		p = p.pNext
	}
	fmt.Println()
}
```

### 判断栈是否为空

`C`语言实现：

```c
bool empty(PSTACK pS) {
    if (pS->pTop == pS->pBottom) {
        return true;
    }
    return false;
}
```

`Go`语言实现：

```go
func empty(pS *Stack) bool {
	if pS.pTop == pS.pBottom {
		return true
	}
	return false
}
```

### 出栈

`C`语言实现：

```c
// 把pS所指向的栈出栈一次，并把出栈的元素存入pVal形参所指向的变量中，
// 出栈成功返回ture，否则返回false
bool pop(PSTACK pS, int *pVal) {
    if (!empty(pS)) {
        PNODE r = pS->pTop;
        *pVal = r->data;
        pS->pTop = r->pNext;
        free(r);
        r = NULL;
        return true;
    }
    return false;
}
```

`Go`语言实现：

```go
// 把pS所指向的栈出栈一次，并把出栈的元素存入pVal形参所指向的变量中，
// 出栈成功返回ture，否则返回false
func pop(pS *Stack, pVal *int) bool {
	if !empty(pS) {
		r := pS.pTop
		*pVal = r.data
		pS.pTop = r.pNext
		r = nil
		return true
	}
	return false
}
```

### 清空栈中元素

`C`语言实现：

```c
void clearStack(PSTACK pS) {
    if (empty(pS)) {
        return;
    } else {
        PNODE p = pS->pTop;
        PNODE q = NULL;
        
        while (p != pS->pBottom) {
            q = p->pNext;
            free(p);
            p = q;
        }
        pS->pTop = pS->pBottom;
    }
}
```

`Go`语言实现：

```go
func clearStack(pS *Stack) {
	if empty(pS) {
		return
	} else {
		p := pS.pTop
		var q *Node = nil

		for p != pS.pBottom {
			q = p.pNext
			p = q
		}
		pS.pTop = pS.pBottom
	}
}
```

