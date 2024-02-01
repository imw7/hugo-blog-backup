+++
title='队列'
tags=['队列']
categories=['数据结构']
date="2024-01-18T12:34:18+08:00"
toc=true
draft=false

+++

本文介绍常用的数据结构——队列。<!--more-->

# 队列

## 定义

队列是一种特殊的线性表，特殊之处在于它只允许在表的前端（`front`）进行删除操作，而在表的后端（`rear`）进行插入操作，和栈一样，队列是一种操作受限制的线性表。进行插入操作的端称为**队尾**，进行删除操作的端称为**队头**。队列中没有元素时，称为**空队列**。

队列的数据元素又称为队列元素。在队列中插入一个队列元素称为**入队**，从队列中删除一个元素称为**出队**。因为队列只允许在一端插入，在另一端删除，所以只有最早进入队列的元素才能最先从队列中删除，故队列又称为**先进先出**（`FIFO - first in first out`）线性表。

## 存储结构

### 顺序队列

顺序队列，也叫数组队列。建立顺序队列结构必须为其静态分配或动态申请一片连续的存储空间，并设置两个指针进行管理。一个是队头指针`front`，它指向队头元素；另一个是队尾指针`rear`，它指向下一个入队元素的存储位置，如图所示。

![queue_1](https://blog.imw7.com/images/data_structure/queue/queue_1.png)

每次在队尾插入一个元素时，`rear`增`1`；每次在队头删除一个元素时，`front`增`1`。随着插入和删除操作的进行，队列元素的个数不断变化，队列所占的存储空间也在为队列结构所分配的连续空间中移动。当`front=rear`时，队列中没有任何元素，称为空队列。当`rear`增加到指向分配的连续空间之外时，队列无法再插入新元素，但这时往往还有大量可用空间未被占用，这些空间是已经出队的队列元素曾经占用过的存储单元。

#### 顺序队列中的溢出现象

1. **“下溢”现象**：当队列为空时，做出队运算产生的溢出现象。“下溢”是正常现象，常用作程序控制转移的条件。
2. **“真上溢”现象**：当队列满时，做进栈运算产生空间溢出的现象。“真上溢”是一种出错状态，应设法避免。
3. **“假上溢”现象**：由于入队和出队操作中，头尾指针只增加不减小，致使被删元素的空间永远无法重新利用。当队列中实际的元素个数远远小于向量空间的规模时，也可能由于尾指针已超越向量空间的上界而不能做入队操作。该现象称为"假上溢"现象。

### 单链队列

使用链表作为基本数据结构，所以不存在伪溢出的问题，队列长度也没有限制，但插入和读取的时间代价较高。

![queue_2](https://blog.imw7.com/images/data_structure/queue/queue_2.png)

### 循环队列

在实际使用队列时，为了使队列空间能重复使用，往往对队列的使用方法稍加改进：无论插入或删除，一旦`rear`指针增`1`或`front`指针增`1`时超出了所分配的队列空间，就让它指向这片连续空间的起始位置。自己则从`MaxSize-1`增`1`变到`0`，可用取余运算方法管理的队列也就称为循环队列。除了一些简单应用之外，真正实用的队列是循环队列。

在循环队列中，当队列为空时，有`front = rear`，而当所有队列空间全占满时，也有`front = rear`。为了区别这两种情况，规定循环队列最多只能有`MaxSize-1`个队列元素，当循环队列中只剩一个空存储单元时，队列就已经满了。因此，队列判空的条件是 `front = rear`，而队列判满的条件是 `front = (rear+1) % MaxSize`。

![queue_3](https://blog.imw7.com/images/data_structure/queue/queue_3.png)

循环队列需要两个参数来确定，`front`和`rear`。两个参数不同场合有不同的含义：

* 队列初始化：`front` 和 `rear` 的值都是零。
* 队列非空：`front`代表的是队列的第一个元素，`rear`代表的是队列的最后一个有效元素的下一个元素。

* 队列空：`front` 和 `rear` 的值相等，但不一定是零。

## 队列操作

### 初始化

在进行队列的操作前，应该先将其初始化，即初始化一个空队列。

#### 顺序队列和循环队列

顺序队列结构体实现：

`C`语言实现：

```c
#define MAXSIZE 6 // 顺序队列的最大存储容量
// 顺序队列的结构体
typedef struct Queue {
    int *pBase; // 头指针
    int front; // 队头下标
    int rear; // 队尾下标
} QUEUE, *PQUEUE;
```

`Go`语言实现：

```go
var MAXSIZE = 6 // 顺序队列的最大存储容量

// Queue 顺序队列的结构体
type Queue struct {
	pBase []int // 队列元素
	front int   // 队头下标
	rear  int   // 队尾下标
}
```

初始化时，将队列的队头下标和队尾下标置为 `0` 即可。

`C`语言实现：

```c
void init(PQUEUE pQ) {
    pQ->pBase = (int *) malloc(sizeof(int) * MAXSIZE);
    pQ->front = 0;
    pQ->rear = 0;
}
```

`Go`语言实现：

```go
func Init(pQ *Queue) {
	pQ.pBase = make([]int, MAXSIZE)
	pQ.front = 0
	pQ.rear = 0
}
```

#### 单链队列

单链队列结构体实现。

`C`语言实现：

```c
// 单链表的结点的结构体
typedef struct QueueNode {
    int data; // 数据域
    struct QueueNode *pNext; // 指针域
} QueueNode, *PLNODE;

// 链队列的结构体
typedef struct {
    PLNODE front; // 队头指针
    PLNODE rear; // 队尾指针
} LQUEUE, *PLQUEUE;
```

`Go`语言实现：

```go
type QueueNode struct {
	data  int        // 数据域
	pNext *QueueNode // 指针域
}

type Queue struct {
	front *QueueNode // 队头指针
	rear  *QueueNode // 队尾指针
}
```

创造出头结点，然后将队头指针和队尾指针均指向头结点即可。

`C`语言实现：

```c
// 初始化链队列：将队头指针和队尾指针指向头结点
void init(PLQUEUE pQ) {
    // 创造头结点
    PLNODE pHead = (PLNODE) malloc(sizeof(PLNODE));
    // 队头队尾指针指向头结点
    pQ->front = pHead;
    pQ->rear = pHead;
}
```

`Go`语言实现：

```go
// Init 初始化链队列：将队头指针和队尾指针指向头结点
func Init(pQ *Queue) {
	// 创造头结点
	pHead := new(QueueNode)
	// 队头队尾指针指向头结点
	pQ.front = pHead
	pQ.rear = pHead
}
```

### 入队操作

入队操作只允许元素从队尾进。

#### 顺序队列和循环队列

顺序队列的队尾下标为队尾元素的下一个元素，所以直接将待入队元素放入队尾下标处，然后队尾下标“`+1`”。（注意：循环队列中的`+1`要对`MAXSIZE`取模）

`C`语言实现：

```c
/**
 * 入队操作
 * pQ: 指向队列的指针
 * elem: 入队的数据
 * return: true成功，false失败
 */
bool en_queue(PQUEUE pQ, int elem) {
    // 判断队列是否已满
    if ((pQ->rear + 1) % MAXSIZE == pQ->front) {
        printf("队列已满，%d无法继续入队\n", elem);
        return false;
    }
    // 元素入队
    pQ->pBase[pQ->rear] = elem;
    // 队尾下标加1
    pQ->rear = (pQ->rear + 1) % MAXSIZE;
    
    printf("%d入队成功\n", elem);

    return true;
}
```

`Go`语言实现：

```go
/**
 * 入队操作
 * pQ: 指向队列的指针
 * elem: 入队的数据
 * return: true成功，false失败
 */
func enQueue(pQ *Queue, elem int) bool {
	// 判断队列是否已满
	if (pQ.rear+1)%MAXSIZE == pQ.front {
		fmt.Printf("队列已满，%d无法继续入队\n", elem)
		return false
	}
	// 元素入队
	pQ.pBase[pQ.rear] = elem
	// 队尾下标加1
	pQ.rear = (pQ.rear + 1) % MAXSIZE

	fmt.Printf("%d入队成功\n", elem)

	return true
}
```

#### 链队列

链队列的入队操作本质是单链表的尾插法。

`C`语言实现：

```c
/**
 * 入队操作
 * pQ: 指向队列的指针
 * elem: 入队的数据
 */
void en_queue(PLQUEUE pQ, int elem) {
    // 创造新结点
    PLNODE pNew = (PLNODE) malloc(sizeof(PLNODE));
    pNew->data = elem;
    // 入队（尾插法）
    pQ->rear->pNext = pNew;
    pQ->rear = pNew;
}
```

`Go`语言实现：

```go
/**
 * 入队操作
 * pQ: 指向队列的指针
 * elem: 入队的数据
 */
func enQueue(pQ *Queue, elem int) {
	// 创造新结点
	pNew := new(QueueNode)
	pNew.data = elem
	// 入队（尾插法）
	pQ.rear.pNext = pNew
	pQ.rear = pNew
}
```

### 出队操作

出队操作只允许元素从队头出。

#### 顺序队列

将队头下标处的元素出队，然后将队头下标“`+1`”（对 `MAXSIZE` 取模）。

`C`语言实现：

```c
/**
 * 出队操作
 * pQ: 指向队列的指针
 * elem: 指向保存出队数据的变量
 * return: true成功，false失败
 */
bool de_queue(PQUEUE pQ, int *pElem) {
    // 判断队列是否为空
    if (pQ->front == pQ->rear) {
        printf("队列空，无元素可出\n");
        return false;
    }
    // 元素出队
    *pElem = pQ->pBase[pQ->front];
    pQ->front = (pQ->front + 1) % MAXSIZE;
    return true;
}
```

`Go`语言实现：

```go
/**
 * 出队操作
 * pQ: 指向队列的指针
 * elem: 指向保存出队数据的变量
 * return: true成功，false失败
 */
func deQueue(pQ *Queue, pElem *int) bool {
	// 判断队列是否为空
	if pQ.front == pQ.rear {
		fmt.Println("队列空，无元素可出")
		return false
	}
	// 元素出队
	*pElem = pQ.pBase[pQ.front]
	pQ.front = (pQ.front + 1) % MAXSIZE
	return true
}
```

#### 链队列

链队列的出队操作本质上是单链表的头删法。注意，如果出队的是队列中最后一个元素，需要在出队后，将队尾指针重新指向头结点，重新形成空队列。

`C`语言实现：

```c
/**
 * 出队操作
 * pQ: 指向队列的指针
 * pElem: 指向保存变量的指针
 * return: true成功，false失败
 */
bool de_queue(PLQUEUE pQ, int *pElem) {
    // 判断队列是否为空
    if (pQ->front == pQ->rear) {
        printf("队列空，无元素可出\n");
        return false;
    }
    PLNODE pFront = pQ->front->pNext; // 队头元素
    // 保存数据
    *pElem = pFront->data;
    // 队头元素出队（头删法）
    pQ->front->pNext = pFront->pNext;
    // 如果元素出完，队尾指针重新指向头结点
    if (pFront == pQ->rear) {
        pQ->rear = pQ->front;
    }
    free(pFront);
    return true;
}
```

`Go`语言实现：

```go
/**
 * 出队操作
 * pQ: 指向队列的指针
 * pElem: 指向保存变量的指针
 * return: true成功，false失败
 */
func deQueue(pQ *Queue, pElem *int) bool {
	// 判断队列是否为空
	if pQ.front == pQ.rear {
		fmt.Println("队列空，无元素可出")
		return false
	}
	pFront := pQ.front.pNext // 队头元素
	// 保存数据
	*pElem = pFront.data
	// 队头元素出队（头删法）
	pQ.front.pNext = pFront.pNext
	// 如果元素出完，队尾指针重新指向头结点
	if pFront == pQ.rear {
		pQ.rear = pQ.front
	}
	return true
}
```

### 遍历操作

#### 顺序队列

`C`语言实现：

```c
void traverse_queue(PQUEUE pQ) {
    int i = pQ->front;
    while (i % MAXSIZE != pQ->rear) {
        printf("%d ", pQ->pBase[i]);
        i = (i + 1) % MAXSIZE;
    }
    printf("\n");
}
```

循环队列的长度为：`(rear - front + MAXSIZE) % MAXSIZE`（空队列也满足）。

`Go`语言实现：

```go
func traverseQueue(pQ *Queue) {
	i := pQ.front
	for i%MAXSIZE != pQ.rear {
		fmt.Printf("%d ", pQ.pBase[i])
		i = (i + 1) % MAXSIZE
	}
	fmt.Println()
}
```

#### 链队列

`C`语言实现：

```c
void traverse_queue(PLQUEUE pQ) {
    PLNODE pNew = pQ->front->pNext; // 指向队头元素
    while (NULL != pNew) {
        printf("%d ", pNew->data);
        pNew = pNew->pNext;
    }
    printf("\n");
}
```

`Go`语言实现：

```go
func traverseQueue(pQ *Queue) {
	pNew := pQ.front.pNext // 指向队头元素
	for nil != pNew {
		fmt.Printf("%d ", pNew.data)
		pNew = pNew.pNext
	}
	fmt.Println()
}
```

### 主函数

```c
int main() {
    QUEUE Q;
    // LQUEUE Q;
    int elem;

    init(&Q);

    en_queue(&Q, 1);
    en_queue(&Q, 2);
    en_queue(&Q, 3);
    en_queue(&Q, 4);
    en_queue(&Q, 5);
    en_queue(&Q, 6);
    en_queue(&Q, 7);
    en_queue(&Q, 8);

    printf("入队后，队列为: \n");
    traverse_queue(&Q);
    
    while (Q.front != Q.rear) {
        if (de_queue(&Q, &elem)) {
            printf("出队成功，队列出队的元素是: %d\n", elem);
            printf("出队后，队列为:\n");
            traverse_queue(&Q);
        }
    }

    return 0;
}
```

以上就是队列的基本原理及操作。

## 队列的具体应用

所有和时间有关的操作都有队列的影子。