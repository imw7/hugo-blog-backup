+++
title='树'
tags=['树']
categories=['数据结构']
date="2024-01-26T11:37:53+08:00"
toc=true
draft=false

+++

本文介绍常用的数据结构——树。<!--more-->

# 树

## 概述

### 定义

`树` 是一种经常用到的数据结构，用来模拟具有树状结构性质的数据集合。

树里的每一个节点有一个值和一个包含所有子节点的列表。从图的观点来看，树也可视为一个拥有 `N个节点` 和 `N-1条边` 的一个有向无环图。

`二叉树` 是一种更为典型的树状结构。如它名字所描述的那样，二叉树是每个节点最多有 `两个子树` 的树结构，通常子树被称作“左子树”和“右子树”。

### 术语

**节点**：包含一个数据元素及若干指向子树分支的信息。

**节点的度**：一个节点拥有子树的数目称为节点的`度`。

**叶子节点**：也称为**终端节点**，没有子树的节点或度为 `0` 的节点。

**分支节点**：也称为**非终端节点**，度不为`0`的节点。

**父节点**：也称为**双亲节点**，若一个节点含有子节点，则这个节点称为其子节点的`父节点`。

**子节点**：也称为**孩子节点**，一个节点含有的**子树的根节点**称为该节点的`子节点`；

**兄弟节点**：具有**相同父节点**的节点互称为`兄弟节点`；

**树的度**：树中所有**节点的度**的最大值称为`树的度`；

**节点的层次**：从根节点开始，假设**根节点为第`1`层**，根节点的子节点为第`2`层，依此类推，如果某一个节点位于第 `L` 层，则其子节点位于第 `L+1` 层。

**树的深度**：也称为**树的高度**，树中所有节点的层次最大值称为树的深度。

**堂兄弟节点**：双亲在同一层的节点互为`堂兄弟`。

**节点的祖先**：**从根节点到该节点所经分支上的所有节点**。

**子孙**：以某节点为根的子树中任一节点都称为`该节点的子孙`。

**有序树**：如果树中各棵子树的次序是有先后次序，则称该树为有序树。

**无序树**：如果树中各棵子树的次序没有先后次序，则称该树为无序树。

**森林**：由`m (m>=0)`棵互不相交的树构成一片`森林`。若果把一棵非空的树的根节点删除，则该树就变成了一片森林，森林中的树由原来根节点的各棵子树构成。

### 二叉树

#### 特殊类型

1. **满二叉树**：如果一颗二叉树只有度为 `0` 的节点和度为 `2` 的节点，并且度为 `0` 的节点在同一层上，则这棵二叉树为满二叉树。

2. **完全二叉树**：一棵深度为 `k` 的有 `n` 个节点的二叉树，对树中的结点按从上至下、从左到右的顺序进行编号，如果编号为 `i (1≤i≤n)` 的节点与**满二叉树**中编号为 `i` 的结点在二叉树中的位置相同，则这棵二叉树称为**完全二叉树**。

   **完全二叉树**的特点是**叶子节点只可能出现在层序最大的两层上，并且某个节点的左分支下子孙的最大层序与右分支下子孙的最大层序相等或大`1`。**

#### 性质

1. 二叉树的第 `i` 层至多有 ${2^{i-1}}$  $(i≥1)$ 个节点。

2. 深度为 `h` 的二叉树中至多含有 ${2^h-1}$ 个节点。  

3. 若在任意一颗二叉树中，有 ${n_0}$ 个叶子节点，有 ${n_2}$ 个度为 `2` 的节点，则必有 ${n_0=n_2+1}$。

4. 具有`n`个节点的二叉树的深度为 ${1+⌊log_2n⌋(向下取整)}$，满二叉树深度为 ${log_2(1+n)}$。

5. 若对一颗有 `n` 个节点的完全二叉树进行顺序编号`(1≤i≤n)`，那么，对于编号为 `i (i≥1)` 的节点：

   当 `i=1` 时，该节点为根，它无双亲节点。

   当 `i>1` 时，该结点的双亲节点的编号为 $\frac{i}{2}$。

   当 `2i≤n`，则有编号为 `2i` 的左节点，否则没有左节点。

   当 `2i+1≤n`，则有编号为 `2i+1` 的右节点，否则没有右节点。

## 树的存储

### 二叉树的存储

#### 顺序存储

这是最直接的方式，也是最常见的存储方式。顺序存储是将二叉树的节点按照层次并从左到右依次标号，若节点有左右孩子，则其左孩子节点为 `2i`，右孩子节点为 `2i+1`。这种方式的优点是存储空间利用率高，不需要额外的指针空间，访问节点的效率也相对较高。但是，由于不是所有二叉树都是完全二叉树，普通二叉树使用顺序表存储或多或少会存在空间浪费的现象。

#### 链式存储

这种存储方式是通过节点之间的引用关系来存储二叉树。每个节点包含一个数据元素和两个指针，分别指向左子节点和右子节点。根节点没有父节点，其他节点都有一个父节点。链式存储的优点是灵活性高，可以根据实际情况自由调整树的结构，不需要预先分配存储空间。但是，这种方法需要额外的存储空间来存储指针，而且访问节点需要通过指针跳转，效率相对较低。

在实际应用中，选择哪种存储方式需要根据具体情况来决定。如果树的结构会经常变化，或者树的深度不确定，那么链式存储更加合适。如果树的结构比较稳定，且深度较小，那么顺序存储可能更为适用。

### 一般树的存储

#### 双亲表示法

利用结构体数组存储：数组下标即为该节点编号，结构体内部有两部分，一部分是数据域，一部分存储其父节点编号，如果没有父节点则为`-1`。

![tree_1](https://blog.imw7.com/images/data_structure/tree/tree_1.png)

缺点是只能记录无序树。

#### 孩子表示法

利用结构体数组存储：每个节点由两部分组成，一部分是数据域，一部分存储其子节点的地址，所有孩子像链表一样连起来。后面的子节点是无序的。

![tree_2](https://blog.imw7.com/images/data_structure/tree/tree_2.png)

缺点是难以寻找父节点。

#### 双亲孩子表示法

利用结构体数组存储：每个节点对应的数组下标就是各自的编号。每个节点由三部分组成：数据域，双亲节点编号，孩子节点地址。

![tree_3](https://blog.imw7.com/images/data_structure/tree/tree_3.png)

优点是即方便寻找双亲节点也方便找孩子节点。

#### 二叉树表示法（最常用）

利用结构体数组存储：思路是先转化为二叉树，再按照二叉树的方法进行存储。每个节点有三部分组成，一个数据域，一个指针域指向其最左侧第一个孩子，另一个指针域指向其第一个兄弟。这一步可以把一个普通树转化为二叉树。

![tree_4](https://blog.imw7.com/images/data_structure/tree/tree_4.png)

普通树转化为二叉树，一定没有右子树。

### 森林的存储

先把森林转化为二叉树，再存储二叉树。把根节点互相看做兄弟，接下来转化方法就跟普通树一样。

## 二叉树的遍历

二叉树常见的遍历方法有前序遍历、中序遍历和后续遍历。

以下面的二叉树为例，依次介绍各种遍历方法。

![b_tree](https://blog.imw7.com/images/data_structure/tree/b_tree.png)

首先需要用程序语言创建二叉树。

`Go`语言实现：

```go
// 二叉树结构体
type TreeNode struct {
    Val   interface{} // 节点的值，写成interface{}类型，是为了接收各种类型的值
	Left  *TreeNode   // 当前节点的左子树
	Right *TreeNode   // 当前节点的右子树
}

// 创建二叉树，返回根节点
func createTree() *TreeNode {
	a := new(TreeNode)
	b := new(TreeNode)
	c := new(TreeNode)
	d := new(TreeNode)
	e := new(TreeNode)
	f := new(TreeNode)
	g := new(TreeNode)
	h := new(TreeNode)
	i := new(TreeNode)

	a.Val = 'A'
	a.Left = nil
	a.Right = nil

	b.Val = 'B'
	b.Left = a
	b.Right = d

	c.Val = 'C'
	c.Left = nil
	c.Right = nil

	d.Val = 'D'
	d.Left = c
	d.Right = e

	e.Val = 'E'
	e.Left = nil
	e.Right = nil

	f.Val = 'F'
	f.Left = b
	f.Right = g

	g.Val = 'G'
	g.Left = nil
	g.Right = i

	h.Val = 'H'
	h.Left = nil
	h.Right = nil

	i.Val = 'I'
	i.Left = h
	i.Right = nil

	return f
}
```


`C`语言实现：

```c
struct TreeNode {
    char val;
    struct TreeNode *left;
    struct TreeNode *right;
};

struct TreeNode *createTree() {
    struct TreeNode *a = (struct TreeNode *) malloc(sizeof(struct TreeNode));
    struct TreeNode *b = (struct TreeNode *) malloc(sizeof(struct TreeNode));
    struct TreeNode *c = (struct TreeNode *) malloc(sizeof(struct TreeNode));
    struct TreeNode *d = (struct TreeNode *) malloc(sizeof(struct TreeNode));
    struct TreeNode *e = (struct TreeNode *) malloc(sizeof(struct TreeNode));
    struct TreeNode *f = (struct TreeNode *) malloc(sizeof(struct TreeNode));
    struct TreeNode *g = (struct TreeNode *) malloc(sizeof(struct TreeNode));
    struct TreeNode *h = (struct TreeNode *) malloc(sizeof(struct TreeNode));
    struct TreeNode *i = (struct TreeNode *) malloc(sizeof(struct TreeNode));

    a->val = 'A';
    a->left = NULL;
    b->right = NULL;

    b->val = 'B';
    b->left = a;
    b->right = d;

    c->val = 'C';
    c->left = NULL;
    c->right = NULL;

    d->val = 'D';
    d->left = c;
    d->right = e;

    e->val = 'E';
    e->left = NULL;
    e->right = NULL;

    f->val = 'F';
    f->left = b;
    f->right = g;

    g->val = 'G';
    g->left = NULL;
    g->right = i;

    h->val = 'H';
    h->left = NULL;
    h->right = NULL;

    i->val = 'I';
    i->left = h;
    i->right = NULL;

    return f;
}
```

### 前序遍历

前序遍历首先访问根节点，然后遍历左子树，最后遍历右子树。

![preorder](https://blog.imw7.com/images/data_structure/tree/preorder.gif)



#### 递归

`Go`语言实现：

```go
func preorderTraversal(root *TreeNode) (vals []interface{}) {
	var preorder func(*TreeNode)
	preorder = func(node *TreeNode) {
		if nil == node {
			return
		}
		vals = append(vals, node.Val)
		preorder(node.Left)
		preorder(node.Right)
	}
	preorder(root)
	return
}
```

`C`语言实现：

```c
void preorderTraversal(struct TreeNode *root) {
    if (NULL != root) {
        printf("%c ", root->val);
        if (NULL != root->left) {
            preorderTraversal(root->left);
        }
        if (NULL != root->right) {
            preorderTraversal(root->right);
        }
    }
}
```

#### 迭代

我们也可以用迭代的方式实现递归函数，两种方式是等价的，区别在于递归的时候隐式地维护了一个栈，而我们在迭代的时候需要显式地将这个栈模拟出来，其余的实现与细节都相同。

`Go`语言实现：

```go
func preorderTraversal(root *TreeNode) (vals []interface{}) {
    var stack []*TreeNode
    node := root
    for nil != node || len(stack) > 0 {
        for nil != node {
            vals = append(vals, node.Val)
            stack = append(stack, node)
            node = node.Left
        }
        node = stack[len(stack)-1].Right
        stack = stack[:len(stack)-1]
    }
    return
}
```

### 中序遍历

中序遍历是先遍历左子树，然后访问根节点，然后遍历右子树。

![inorder](https://blog.imw7.com/images/data_structure/tree/inorder.gif)

#### 递归

`Go`语言实现：

```go
func inorderTraversal(root *TreeNode) (vals []interface{}) {
	var inorder func(*TreeNode)
	inorder = func(node *TreeNode) {
		if nil == node {
			return
		}
		inorder(node.Left)
		vals = append(vals, node.Val)
		inorder(node.Right)
	}
	inorder(root)
	return
}
```

`C`语言实现：

```c
void inorderTraversal(struct TreeNode *root) {
    if (NULL != root) {
        if (NULL != root->left) {
            inorderTraversal(root->left);
        }
        printf("%c ", root->val);
        if (NULL != root->right) {
            inorderTraversal(root->right);
        }
    }
}
```

#### 迭代

`Go`语言实现：

```go
func inorderTraversal(root *TreeNode) (vals []interface{}) {
	var stack []*TreeNode
	for root != nil || len(stack) > 0 {
		for root != nil {
			stack = append(stack, root)
			root = root.Left
		}
		root = stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		vals = append(vals, root.Val)
		root = root.Right
	}
	return
}
```

通常来说，对于二叉搜索树，我们可以通过中序遍历得到一个递增的有序序列。 

### 后序遍历

后序遍历是先遍历左子树，然后遍历右子树，最后访问树的根节点。

![postorder](https://blog.imw7.com/images/data_structure/tree/postorder.gif)

#### 递归

`Go`语言实现：

```go
func postorderTraversal(root *TreeNode) (vals []interface{}) {
	var postorder func(*TreeNode)
	postorder = func(node *TreeNode) {
		if nil == node {
			return
		}
		postorder(node.Left)
		postorder(node.Right)
		vals = append(vals, node.Val)
	}
	postorder(root)
	return
}
```

`C`语言实现：

```c
void postorderTraversal(struct TreeNode *root) {
    if (NULL != root) {
        if (NULL != root->left) {
            postorderTraversal(root->left);
        }
        if (NULL != root->right) {
            postorderTraversal(root->right);
        }
        printf("%c ", root->val);
    }
}
```

#### 迭代

`Go`语言实现：

```go
func postorderTraversal2(root *TreeNode) (vals []interface{}) {
	var stk []*TreeNode
	var prev *TreeNode
	for nil != root || len(stk) > 0 {
		for nil != root {
			stk = append(stk, root)
			root = root.Left
		}
		root = stk[len(stk)-1]
		stk = stk[:len(stk)-1]
		if nil == root.Right || root.Right == prev {
			vals = append(vals, root.Val)
			prev = root
			root = nil
		} else {
			stk = append(stk, root)
			root = root.Right
		}
	}
	return
}
```

值得注意的是，当你删除树中的节点时，删除过程将按照后序遍历的顺序进行。 也就是说，当你删除一个节点时，你将首先删除它的左节点和它的右边的节点，然后再删除节点本身。

### 已知两种遍历序列求原始二叉树

只有通过 `前序遍历`和 `中续遍历`，或者 `中序遍历` 和 `后续遍历`，我们才可以唯一的确定一个二叉树。

来看一个例子：

**已知一个二叉树的两种遍历方式：`前序：ABCDEFGH`，`中序：BDCEAFHG`。求后续遍历？**

![tree_5](https://blog.imw7.com/images/data_structure/tree/tree_5.png)