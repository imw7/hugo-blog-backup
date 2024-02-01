+++
title='链表'
tags=['链表']
categories=['数据结构']
date="2024-01-11T16:20:08+08:00"
toc=true
draft=false

+++

本文是我在学习数据结构中，对链表部分知识的总结。<!--more-->

# 链表

## 定义

### 概念

链表是一种**物理存储结构上非连续、非顺序的存储结构**，数据元素的逻辑顺序是通过链表中的指针链接次序实现的。

![list_1](https://blog.imw7.com/images/data_structure/list/list_1.png)

### 组成

**由一系列结点（`Node`）通过指针连接而成**，从一个头结点（`Head`）开始，头结点作为链表的入口点，它包含了对第一个结点的引用。最后一个结点的指针指向一个空值（`NULL`），表示链表的结束。

![list_2](https://blog.imw7.com/images/data_structure/list/list_2.png)

#### 专业术语

1. **首结点**：第一个有效结点。
2. **尾结点**：最后一个有效结点。
3. **头结点**：头结点的数据类型和首结点的数据类型一样。第一个有效结点之前的那个结点，头结点并不存放有效数据，加头结点的目的主要是为了方便对链表进行操作。
4. **头指针**：指向头结点的指针变量。
5. **尾指针**：指向尾结点的指针变量。

**只需要`头指针`这一个参数，就能对链表进行处理。因为通过头指针可以推算出链表的其他所有信息。**

### 存储

链表在内存中的存储方式则是**随机存储（见缝插针）**，每一个结点分布在内存的不同位置，依靠指针关联起来。（只要有足够的内存空间，就能为链表分配内存）

![list_3](https://blog.imw7.com/images/data_structure/list/list_3.png)

### 优缺点

相对于顺序存储（例如数组）。

#### 优点

链表的**插入操作更快（`O(1)`），无需预先分配内存空间。**

#### 缺点

失去了**随机读取**的优点（需要从头结点开始依次遍历，直到找到目标结点），**内存消耗较大**（每个结点都需要存储指向下一个结点的指针）。

### 对比

链表是通过结点把离散的数据链接成一个表，通过对结点的插入和删除操作从而实现对数据的存取。而数组是通过开辟一段连续的内存来存储数据，这是数组和链表最大的区别。

数组的每个成员对应链表的结点，成员和结点的数据类型可以是标准的 `C` 类型或者是用户自定义的结构体。

数组有起始地址和结束地址，而链表是一个圈，没有头和尾之分，但是为了方便结点的插入和删除操作会人为的规定一个根结点。

| 不同点                 | 顺序表                           | 链表                           |
| ---------------------- | -------------------------------- | ------------------------------ |
| 存储空间               | 物理上一定连续                   | 逻辑上连续，但物理上不一定连续 |
| 随机访问               | 支持，`O(1)`                     | 不支持，`O(N)`                 |
| 任意位置插入或删除元素 | 可能需要搬移元素，效率低，`O(N)` | 只需修改指针指向               |
| 插入                   | 动态顺序表，空间不够时需要扩容   | 没有容量的概念                 |
| 应用场景               | 元素高效存储+频繁访问            | 任意位置插入和删除频繁         |
| 缓存利用率             | 高                               | 低                             |

### 分类

链表一般有单向链表，双向链表，循环链表，带头链表这四种形式。

常用的两种结构：

![list_4](https://blog.imw7.com/images/data_structure/list/list_4.png)

1. 无头单向非循环链表：结构简单，一般不会单独用来存放数据。实际中更多是作为其他数据结构的子结构，如哈希桶、图的邻接表等等。另外这种结构在笔试面试中出现很多。
2. 带头双向循环链表：结构最复杂，一般用在单独存储数据。实际中使用的链表数据结构都是带头双向循环链表。另外这个结构虽然复杂，但是使用代码实现以后会发现该结构会带来很多优势，实现反而简单。

### 总结

链表有以下特点：

1. `n`个结点离散分配。

2. 彼此通过指针相连。

3. 每个结点只有一个前驱结点，每个结点只有一个后续结点。

4. 首结点没有前驱结点，尾结点没有后续结点。

## 链表的实现

### 链表的创建

`C`语言实现：

```c
// 定义链表结点结构
typedef struct Node {
    int data; // 数据域
    struct Node *pNext; // 指针域，指向下一个结点的指针
}NODE, *PNODE; // NODE 等价于 struct Node, PNODE等价于struct Node *

// 创建链表函数
PNODE create_list() {
    int length; // 用来存放有效结点的个数
    int value; // 用来临时存放用户输入的结点的值
    
    // 分配了一个不存放有效数据的头结点
    PNODE pHead = (PNODE) malloc(sizeof(NODE));
    if (NULL == pHead) {
        printf("内存分配失败，程序终止\n");
        exit(-1);
    }
    PNODE pTail = pHead; // 初始化尾结点，因为pHead一定存在，所以将pHead的值赋给pTail
    pTail->pNext = NULL; // 尾结点的下一个结点一定为空
    
    printf("请输入要生成的链表的结点数: ");
    scanf("%d", &length);
    
    for (int i = 0; i < length; ++i) {
        printf("请输入第%d个结点的值: ", i + 1);
        scanf("%d", &value);
        
        // 为用户输入的新结点动态分配内存
        PNODE pNew = (PNODE) malloc(sizeof(NODE));
        if (NULL == pNew) {
            printf("内存分配失败，程序终止\n");
        	exit(-1);
        }
        pNew->data = value; // 新结点的值为用户输入的值
        pNew->pNext = NULL; // 清空pNew指针域，pNew作为尾结点，它的下一个结点为空
        pTail->pNext = pNew; // pTail的下一个结点指向pNew
        pTail = pNew; // pTail 时刻指向尾指针域
    }
    
    return pHead;
}
```

`Go`语言实现：

```go
// Node 定义链表结点结构
type Node struct {
	data  int   // 数据域
	pNext *Node // 指针域，指向下一个结点的指针
}

// CreateList 创建链表函数
func CreateList() *Node {
	var length int // 用来存放有效结点的个数
	var value int  // 用来临时存放用户输入的结点的值

	// 分配了一个不存放有效数据的头结点
	pHead := new(Node)
	if nil == pHead {
		fmt.Println("内存分配失败，程序终止")
		os.Exit(-1)
	}
	pTail := pHead    // 初始化尾结点，因为pHead一定存在，所以将pHead的值赋给pTail
	pTail.pNext = nil // 尾结点的下一个结点一定为空

	fmt.Print("请输入要生成的链表的结点数: ")
	_, _ = fmt.Scanf("%d", &length)

	for i := 0; i < length; i++ {
		fmt.Printf("请输入第%d个结点的值: ", i+1)
		_, _ = fmt.Scanf("%d", &value)

		// 为用户输入的新结点分配内存
		pNew := new(Node)
		if nil == pNew {
			fmt.Println("内存分配失败，程序终止")
			os.Exit(-1)
		}
		pNew.data = value  // 新结点的值为用户输入的值
		pNew.pNext = nil   // 清空pNew指针域，pNew作为尾结点，它的下一个结点为空
		pTail.pNext = pNew // pTail的下一个结点指向pNew
		pTail = pNew       // pTail时刻指向尾指针域
	}

	return pHead
}
```

### 链表的长度

`C`语言实现：

```c
int get_size(PNODE pHead) {
    PNODE p = pHead->pNext;
    int length = 0;

    while (NULL != p) {
        ++length;
        p = p->pNext;
    }

    return length;
}
```

`Go`语言实现：

```go
// 链表的长度
func getSize(pHead *Node) int {
	p := pHead.pNext
	length := 0

	for nil != p {
		length++
		p = p.pNext
	}

	return length
}
```

### 链表的输出

`C`语言实现：

```c
void traverse_list(PNODE pHead) {
    PNODE p = pHead->pNext; // 定义指针p，指向首结点
    
    while (NULL != p) { // 如果p不为空
        printf("%d ", p->data); // 输出数据
        p = p->pNext; // 指针后移
    }
    printf("\n");
}
```

`Go`语言实现：

```go
// 链表的输出
func traverseList(pHead *Node) {
	p := pHead.pNext // 定义指针p，指向首结点

	for nil != p { // 如果p不为空
		fmt.Printf("%d ", p.data) // 输出数据
		p = p.pNext               // 指针后移
	}
	fmt.Println()
}
```

### 链表的排序

#### 选择排序

`C`语言实现：

```c
// 选择排序
void selection_sort(PNODE pHead) {
    int i, j;
    PNODE p, q;

    int length = get_size(pHead);
    
    // 如果链表长度小于2则返回
    if (length < 2 || NULL == pHead->pNext) {
        return;
    } 
    for (i = 0, p = pHead->pNext; i < length - 1; ++i, p = p->pNext) {
        for (j = i + 1, q = p->pNext; j < length; ++j, q = q->pNext) {
            if (p->data > q->data) {
                int tmp = p->data;
                p->data = q->data;
                q->data = tmp;
            }
        }
    }
}
```

`Go`语言实现：

```go
// 选择排序
func selectionSort(pHead *Node) {

	length := getSize(pHead)

	// 如果链表长度小于2返回
	if length < 2 || nil == pHead.pNext {
		return
	}

	for i := 0; i < length-1; i++ {
		p := pHead.pNext
		for j := i + 1; j < length; j++ {
			q := p.pNext
			if p.data > q.data {
				tmp := p.data
				p.data = q.data
				q.data = tmp

				q = q.pNext
			}
			p = p.pNext
		}
	}
}
```

#### 冒泡排序

```c
// 冒泡排序
void bubble_sort(PNODE pHead) {
    PNODE p, q;
    int i, j;

    int length = get_size(pHead);

    if (length < 2 || NULL == pHead->pNext) {
        return;
    } 
    for (i = 0, p = pHead->pNext; i < length - 1; ++i, p = p->pNext) {
        for (j = 0, q = pHead->pNext; j < length - 1 - i; ++j, q = q->pNext) {
            if (q->data > q->pNext->data) {
                int tmp = q->data;
                q->data = q->pNext->data;
                q->pNext->data = tmp;
            }
        }
    }
}
```

#### 插入排序

`C`语言实现：

```c
// 插入排序
void insert_sort(PNODE pHead) {
    PNODE pre, cur, next; // 三个指针

    cur = pHead->pNext->pNext; // 指向第二个结点
    pHead->pNext->pNext = NULL;

    while (cur) {
        next = cur->pNext; // 保存当前结点的下一个结点
        pre = pHead; // 当前结点的上一个结点
        while (NULL != pre->pNext && pre->pNext->data < cur->data) {
            pre = pre->pNext;
        }
        cur->pNext = pre->pNext;
        pre->pNext = cur;
        cur = next;
    }
}
```

`Go`语言实现：

```go
func insertSort(pHead *Node) {
	cur := pHead.pNext.pNext // 指向第二个结点
	pHead.pNext.pNext = nil

	for nil != cur {
		next := cur.pNext // 保存当前结点的下一个结点
		pre := pHead // 当前结点的上一个结点
		for nil != pre.pNext && pre.pNext.data < cur.data {
			pre = pre.pNext
		}
		cur.pNext = pre.pNext
		pre.pNext = cur
		cur = next
	}
}
```

### 链表的插入

![list_5](https://blog.imw7.com/images/data_structure/list/list_5.png)

同顺序表一样，向链表中增添元素，根据添加位置不同，可分为以下 `3` 种情况：

* 插入到链表的头部，作为首元结点；
* 插入到链表中间的某个位置；
* 插入到链表的最末端，作为链表中最后一个结点；

`C`语言实现：

```c
// 在链表中插入结点
// 在pHead所指向链表的第pos个结点前插入一个新结点，
// 该结点的值是val，并且pos从1开始
bool insert_list(PNODE pHead, int pos, int val) {
    int i = 0;
    PNODE p = pHead;

    while (NULL != p && i < pos - 1) {
        p = p->pNext;
        ++i;
    }

    if (i > pos - 1 || NULL == p) {
        return false;
    }

    PNODE pNew = (PNODE) malloc(sizeof(NODE));
    if (NULL == pNew) {
        printf("动态分配内存失败\n");
        exit(-1);
    }
    pNew->data = val;
    PNODE q = p->pNext;
    p->pNext = pNew;
    pNew->pNext = q;

    return true;
}
```

`Go`语言实现：

```go
// 在链表中插入结点
// 在pHead所指向链表的第pos个结点前插入一个新结点，
// 该结点的值是val，并且pos从1开始
func insertList(pHead *Node, pos, val int) bool {
	i := 0
	p := pHead

	for nil != p && i < pos-1 {
		p = p.pNext
		i++
	}

	if i > pos-1 || nil == p {
		return false
	}

	pNew := new(Node)
	if nil == pNew {
		fmt.Println("动态分配内存失败")
		os.Exit(-1)
	}
	pNew.data = val
	q := p.pNext
	p.pNext = pNew
	pNew.pNext = q

	return true
}
```

### 链表的查询

`C`语言实现：

```c
// 在链表中查询结点
int search_list(PNODE pHead, int val) {
    int cnt = 1; // cnt为链表数据的位置
    PNODE p = pHead->pNext;

    while (NULL != p && p->data != val) {
        cnt++;
        p = p->pNext;
    }
    if (NULL == p || p->data != val) {
        return -1;
    }

    return cnt;
}
```

`Go`语言实现：

```go
// 在链表中查询结点
func searchList(pHead *Node, val int) int {
	cnt := 1 // cnt为链表数据的位置
	p := pHead.pNext

	for nil != p && p.data != val {
		cnt++
		p = p.pNext
	}
	if nil == p || p.data != val {
		return -1
	}

	return cnt
}
```

### 链表的删除

![list_6](https://blog.imw7.com/images/data_structure/list/list_6.png)

`C`语言实现：

```c
// 在链表中删除结点
bool delete_list(PNODE pHead, int pos, int *pVal) {
    int i = 0;
    PNODE p = pHead;

    while (NULL != p->pNext && i < pos - 1) {
        p = p->pNext;
        ++i;
    }

    if (i > pos - 1 || NULL == p->pNext) {
        return false;
    }

    PNODE q = p->pNext;
    *pVal = q->data;

    // 删除p结点后面的结点
    p->pNext = p->pNext->pNext;
    free(q);
    q = NULL;

    return true;
}
```

`Go`语言实现：

```go
// 在链表中删除结点
func deleteList(pHead *Node, pos int, pVal *int) bool {
	i := 0
	p := pHead

	for nil != p.pNext && i < pos-1 {
		p = p.pNext
		i++
	}

	if i > pos-1 || nil == p.pNext {
		return false
	}

	q := p.pNext
	*pVal = q.data

	// 删除p结点后面的结点
	p.pNext = p.pNext.pNext
	q = nil
    
	return true
}
```

