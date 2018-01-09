---
layout: post
title:  "数据结构[2] -- 二叉搜索树（BST）"
categories: 数据结构和算法
tags:  数据结构
author: G.Fukang
---

* content
{:toc}
二叉搜索树操作集：建立树、查找节点、删除指定节点，插入节点、先序&后序&中序&层序遍历树、求数的高度、查找最大最小值。





## 二叉搜索树操作集

**二叉搜索树**：一个节点的左子节点的关键字的值小于这个节点，右子节点的关键字的值大于或者等于这个父节点

## 查找节点

在查找过程中，用变量`current`来保存正在查看的节点，参数`key`是要查找的值，查找从`root`开始，因此开始把`current`设为根。之后，在`while`循环中，将要查找的值，`key`与`iData`做比较。小于，则`current`设为左节点，大于则设为右节点。

## 插入节点

`current`在查找过程中会变成`null`才能发现它查找的上一个节点是不是空节点，因此，引入一个新的变量`parent(current父节点)`来存储遇到的最后一个不是`null`的节点。

1. 查找`null`节点
2. 空树`root=newNode`
3. 插入叶节点，即`parent`没有左子节点 `parent.leftChild=newNode`
4. 插入非叶节点，即`parent`没有右子节点`parent.rightChild=newNode

## 二叉树高度
- `PostOrderGetHeight` Height=MAX(Hl,Hr)+1

## 查找最大最小值 
- 最大元素一定在树的最右端分支的端节点 
- 最小元素一定在树的最左端分支的端节点

## 遍历树
- 中序遍历 `inOrder`    `left->root->right`

- 前序遍历 `preOrder`   `root->left->right`

- 后序遍历 `postOrder`  `left->right->root`

- 层序遍历 `levelTravel`

  - `root`入队
  - 如果队列不为空，则进入循环    


  - 队首元素出队，并输出
  - 如果该队首元素有左孩子，则将其左孩子入队
  - 如果该队首元素有右孩子，则将其右孩子入队

## 删除节点

- 没有子节点     
  找到节点后，首先判断是不是`root`,如果是的话，就将它置为`null`,否则，就把父节点`leftChild`或者`rightChild`置为`null`
- 有一个子节点
  - 要删除的节点的子节点有左子节点或者右子节点，并且每种情况中的要删除的节点也可能是自己父节点的左子节点或者右子节点
  - 被删除的节点是根，他没有父节点，需要被合适的子树所代替
- 有两个子节点        
  删除有两个字节点的节点，用它的中序后继来代替该节点 `getSuccessor()`。      
  **寻找后继节点**：比要删除节点大的下一个节点 `right->left->left->...->left->left=null` 
  - 后继节点是`delNode`节点的右子节点       
    把`current`从它的父节点的`rightChild`or`leftChild`删掉，然后指向后继节点     
    把`current`的左子节点插到后继节点的`leftChild`
  - 后继节点是`delNode`节点的左子节点       
    后继节点的`leftChild`置为后继节点的右子节点       
    后继节点的`rightChild`置为要删除节点的右子节点     
    把`current`从它父节点的`rightChild`字段移除，把这个字段置为`successor`       
    把`current`的左子节点从`current`移除，`successor`的`leftChild`的字段置为`successor`    

## Github

完整代码，我托管在[Github](https://github.com/gongfukangEE/Data-Structures-Java/tree/master/src/BinaryTree)上，如果对你有帮助，请给我点个star以示肯定和鼓励。