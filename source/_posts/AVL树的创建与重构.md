---
title: AVL树的创建与重构
tags: [算法, C++]
categories: [笔记]
date: 2020/8/13 20:46:25
---

## 定义：

#### 二叉搜索树：
他或是一棵空树，或是具有以下性质的树
1. 若它的左子树不空，则左子树上所有结点的值均小于它的根结点的值；
2. 若它的右子树不空，则右子树上所有结点的值均大于它的根结点的值；
3. 它的左、右子树也分别为二叉搜索树。 

#### 平衡二叉树（AVL）：
平衡二叉树是在二叉搜索树的基础上发展而来的，他在节点元素的大小关系上同二叉搜索树。同时左右子树的高度的差值的绝对值小于等于1，并且左右子树都是一棵平衡二叉树。

#### AVL树的基本操作：

> AVL树节点定义如下

```cpp
typedef struct node
{
    int data;
    int height;
    struct node *left, *right;
    node(int data, int height) : data(data), height(height), left(NULL), right(NULL) {}
} node;
```



1. 插入元素（需要调整）
2. 查找元素
3. 删除元素


#### AVL插入元素与调整
---
> 插入元素是一个递归的过程

1. 若节点为空，则根据元素值新建节点。
2. 否则判断插入元素的值，若大于当前节点元素值，则插入到节点的右子树中，若小于则插入到节点的左子树中。（二叉搜索树不存在元素相等的情况）
3. 插入左子树或右子树的过程同步骤2。

---
> 由于原来的树的左右子树高度可能存在差值，插入时可能存在将新节点插入到本就高度更高的一边，故需要对树进行调整，使其符合AVL树的要求。

##### 原左子树重，插入节点还插入到左子树的左子树上，则需要右旋

![image.png](https://b3logfile.com/file/2020/08/image-374b9555.png)


具体操作为：
1. 取**根节点A**的**左子树节点B**作为新的根节点
2. B的右子树作为A的左子树
3. A作为B的右子树
4. 更新各树的高度

上代码：

```cpp
node *SRR(node *tree)
{
    node *newtree = tree->left;
    tree->left = newtree->right;
    newtree->right = tree;
    tree->height = max(getHeight(tree->left), getHeight(tree->right)) + 1;
    newtree->height = max(getHeight(newtree->left), getHeight(newtree->right)) + 1;
    return newtree;
}
```


##### 原右子树重，插入节点还插入到右子树的右子树上，则需要左旋

![image.png](https://b3logfile.com/file/2020/08/image-152116b2.png)


上代码：

```cpp
node *SLR(node *tree)
{
    node *newtree = tree->right;
    tree->right = newtree->left;
    newtree->left = tree;
    tree->height = max(getHeight(tree->left), getHeight(tree->right)) + 1;
    newtree->height = max(getHeight(newtree->left), getHeight(newtree->right)) + 1;
    return newtree;
}

```


##### 原左子树重，插入节点还插入到左子树的右子树上，则需要先左旋再右旋

> 这种情况相较于前面的单旋转更复杂一些

具体操作为：
1. 先对根节点左子树节点进行左旋，完成后根节点的左子树节点更新
2. 之后再对根节点进行右旋

![image.png](https://b3logfile.com/file/2020/08/image-4ba6f934.png)


上代码：
```cpp
node *DLRR(node *tree)
{
    node *temp = tree->left;
    tree->left = SLR(temp);
    return SRR(tree);
}
```


##### 原右子树重，插入节点还插入到右子树的左子树上，则需要先右旋再左旋

![image.png](https://b3logfile.com/file/2020/08/image-87f7bcc0.png)

上代码：
```cpp
node *DRLR(node *tree)
{
    node *temp = tree->right;
    tree->right = SRR(temp);
    return SLR(tree);
}

```




---

插入操作的完整代码：

```cpp
#include <iostream>
#include <algorithm>

using namespace std;

typedef struct node
{
    int data;
    int height;
    struct node *left, *right;
    node(int data, int height) : data(data), height(height), left(NULL), right(NULL) {}
} node;

int getHeight(node *tree)
{
    if (tree == NULL)
        return 0;
    else
        return tree->height;
}

/*
原左子树重，插入节点还插入到左子树的左子树上
则需要右旋
*/
node *SRR(node *tree)
{
    node *newtree = tree->left;
    tree->left = newtree->right;
    newtree->right = tree;
    tree->height = max(getHeight(tree->left), getHeight(tree->right)) + 1;
    newtree->height = max(getHeight(newtree->left), getHeight(newtree->right)) + 1;
    return newtree;
}

/*
原右子树重，插入节点还插入到右子树的右子树上
则需要左旋
*/
node *SLR(node *tree)
{
    node *newtree = tree->right;
    tree->right = newtree->left;
    newtree->left = tree;
    tree->height = max(getHeight(tree->left), getHeight(tree->right)) + 1;
    newtree->height = max(getHeight(newtree->left), getHeight(newtree->right)) + 1;
    return newtree;
}

/*
原左子树重，插入节点还插入到左子树的右子树上
则需要先左旋再右旋
*/
node *DLRR(node *tree)
{
    node *temp = tree->left;
    tree->left = SLR(temp);
    return SRR(tree);
}

/*
原右子树重，插入节点还插入到右子树的左子树上
则需要先右旋再左旋
*/
node *DRLR(node *tree)
{
    node *temp = tree->right;
    tree->right = SRR(temp);
    return SLR(tree);
}

node *insert(node *tree, int x)
{
    if (tree == NULL)
    //空节点
    {
        tree = new node(x, 1);
    }
    else if (x < tree->data)
    //插入左子树
    {
        tree->left = insert(tree->left, x);
        if (getHeight(tree->left) - getHeight(tree->right) > 1)
        //插入后树失衡
        {
            if (x < tree->left->data)
            //插入到左子树的左子树
            {
                tree = SRR(tree);
            }
            else if (x > tree->left->data)
            //插入到左子树的右子树
            {
                tree = DLRR(tree);
            }
        }
    }
    else if (x > tree->data)
    //插入右子树
    {
        tree->right = insert(tree->right, x);
        if (getHeight(tree->right) - getHeight(tree->left) > 1)
        {
            if (x > tree->right->data)
            //插入到右子树的右子树
            {
                tree = SLR(tree);
            }
            else if (x < tree->right->data)
            //插入到右子树的左子树
            {
                tree = DRLR(tree);
            }
        }
    }
    tree->height = max(getHeight(tree->left), getHeight(tree->right)) + 1;
    return tree;
}

```

有时间在更新删除和查找吧.....