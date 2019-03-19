---
layout: post
title:  "数据结构[1] -- 栈、队列、优先队列、链表和堆"
categories: 数据结构和算法
tags:  数据结构
author: G.Fukang
---

* content
{:toc}
常用的数据结构：栈、队列、优先队列、单向链表和堆



## 栈（Stack）
栈是一种通常用于临时存储数据的数据结构，具有**先入后出**的特点，可以通过数组来实现

```java
/**
 * @Auther gongfukang
 * @Date 2017/12/4 20:14
 */
public class Stack {
    private int maxSize;
    private long[] stackArray;
    private int top;

    public Stack(int s){
        maxSize=s;
        stackArray=new long[maxSize];
        top=-1;
    }

    //入栈
    public void push(long j){
        stackArray[++top]=j;
    }

    //出栈
    public long pop(){
        return stackArray[top--];
    }

    //查看栈顶元素
    public long peek(){
        return stackArray[top];
    }

    public boolean isEmpty(){
        return (top==-1);
    }

    public boolean isFull(){
        return(top==maxSize-1);
    }
}
```

## 队列（Queue）

队列具有**先入先出**的特点，通常也是用数组实现，为了避免队列不满却不能插入新数据项的问题，可以让队头队尾指针回绕到数组开始的位置，这就是**循环队列**。

```java
/**
 * @Auther gongfukang
 * @Date 2017/12/4 20:14
 */
public class Queue {
    private int maxSize;
    private long[] queueArray;
    private int front;
    private int rear;
    private int Items;

    public Queue(int s){
        maxSize=s;
        queueArray=new long[maxSize];
        front=0;
        rear=-1;
        Items=0;
    }

   //入队
    public void insert(long j){
        if(rear==maxSize){
            rear=-1;
        }
        queueArray[++rear]=j;
        Items++;
    }

    //出队
    public long remove(){
        long temp=queueArray[front++];
        if(front==maxSize)
            front=0;
        Items--;
        return temp;
    }

    //查看队头元素
    public long peekFront(){
        return queueArray[front];
    }

    public boolean isEmpty(){
        return (Items==0);
    }

    public boolean isFull(){
        return (Items==maxSize);
    }

    public int size(){
        return Items;
    }
}
```



## 优先队列（PriorityQ）

优先队列是队列的一种，它也像普通队列一样，有一个队头和队尾，但是，在优先级队列中，数据项是有序的，这样`key` 值（最大或者最小）总是在队头，数据的插入也是会按照顺序插入到合适的位置以保证队列的顺序。

```java
/**
 * @Auther gongfukang
 * @Date 2017/12/5 9:07
 */
public class PriorityQ {
    private int maxSize;
    private long[] queArray;
    private int Items;

    public PriorityQ(int s) {
        maxSize = s;
        queArray = new long[maxSize];
        Items = 0;
    }

    public void insert(long item) {
        int j;

        if (Items == 0) {
            queArray[Items++] = item;
        } else {
            for (j = Items - 1; j >= 0; j--) {
                if (item > queArray[j])
                    queArray[j + 1] = queArray[j];
                else
                    break;
            }
            queArray[j+1]=item;
            Items++;
        }
    }

    public long remove(){
        return queArray[--Items];
    }

    public long peekMin(){
        return queArray[Items-1];
    }

    public boolean isEmpty(){
        return (Items==0);
    }

    public boolean isFull(){
        return (Items==maxSize);
    }
}
```



## 单向链表（linkList）

链表是仅次于数组的第二广泛使用的数组结构，她解决了数组的大小固定问题，可以不断扩展。同时链表需要多少内存就可以用多少内存，并且可以扩展到所有内存。

链表的插入和删除操作均为`O(1)` ，查找和删除操作需要`O(N)`

链表的关键是链接点的定义：

```java
/**
 * @Auther gongfukang
 * @Date 2017/12/5 10:47
 * 链结点
 */
public class Link {
    public int iData;
    public double dData;
    public Link next;

    public Link(int id,double dd){
        iData=id;
        dData=dd;
    }

    public void displayLink(){
        System.out.println("{"+iData+","+dData+"}");
    }
}
```

其他操作如下

```java
/**
 * @Auther gongfukang
 * @Date 2017/12/5 10:19
 * 单链表
 */
public class LinkList {
    private Link first;

    public LinkList() {
        first = null;
    }

    public boolean isEmpty() {
        return (first == null);
    }

    public void insertFirst(int id, double dd) {
        Link newLink = new Link(id, dd);
        newLink.next = first;
        first = newLink;
    }

    public Link find(int key) {
        Link current = first;
        while (current.iData != key) {
            if (current.next == null)
                return null;
            else
                current = current.next;
        }
        return current;
    }

    public Link delete(int key) {
        Link current = first;
        Link previous = first;
        while (current.iData != key) {
            if (current.next == null) {
                return null;
            } else{
                previous=current;
                current=previous.next;
            }
        }
        if(current==first){
            first=first.next;
        }else{
            previous.next=current.next;
        }
        return current;
    }

    public void displayList() {
        System.out.println("List (first-->last): ");
        Link current = first;
        while (current != null) {
            current.displayLink();
            current = current.next;
        }
        System.out.println("");
    }
}
```

## 堆（Heap）

堆是实现优先队列的特殊二叉树，操作效率为`O(log N)`

**堆特点**

- 完全二叉树
- 数组实现
- 堆中的每一个节点都满足堆的条件，也就是说每一个节点的关键字都大于(或等于)这个节点的子节点的关键字 

**堆操作**：

**移除**：删除堆`key`最大的节点

- 移走`root`
- 把最后一个节点移动到`root`位置
- 一直向下筛选`trickleDown`这个节点，直到它在一个大于它的节点之下，小于它的节点之上为止 

**向下筛选`trickleDown`**：

- `index<currentSize/2`判断是否有`leftChild`
- 从`leftChild`和`rightChild`中找到一个大值的节点
- 将`Child`变量指向较大的那个

**插入 `insert`**
- 插入数组最后一个空着的单元中，数组容量大+1
- 向上筛选`trickleUp`（目标节点不断和父节点交换）

定义堆单元：

```java
/**
 * @Auther gongfukang
 * @Date 2017/12/24 10:05
 */
public class Node {
    private int iData;

    public Node(int key){
        iData=key;
    }

    public int getKey(){
        return iData;
    }

    public void setKey(int id){
        iData=id;
    }
}
```

堆操作集：

```java
/**
 * @Auther gongfukang
 * @Date 2017/12/24 10:07
 * 大顶堆
 */
public class Heap {
    private Node[] heapArray;
    private int maxSize;
    private int currentSize;

    public Heap(int mx){
        maxSize=mx;
        currentSize=0;
        heapArray=new Node[maxSize];
    }

    public boolean isEmpty(){
        return currentSize==0;
    }

    public boolean insert(int key){
        if(currentSize==maxSize)
            return false;
        Node newNode=new Node(key);
        heapArray[currentSize]=newNode;
        trickleUp(currentSize++);
        return true;
    }

    public void trickleUp(int index){
        int parent=(index-1)/2;
        Node bottom=heapArray[index];

        while (index>0&&heapArray[parent].getKey()<bottom.getKey()){
            heapArray[index]=heapArray[parent];
            index=parent;
            parent=(parent-1)/2;
        }
        heapArray[index]=bottom;
    }

    public Node remove(){
        Node root=heapArray[0];
        heapArray[0]=heapArray[--currentSize];
        trickleDown(0);
        return root;
    }

    public void trickleDown(int index){
        int largeChild;
        Node top=heapArray[index];

        while (index<currentSize/2){
            int leftChild=2*index+1;
            int rightChild=leftChild+1;

            if(rightChild<currentSize&&heapArray[leftChild].getKey()<heapArray[rightChild].getKey())
                largeChild=rightChild;
            else
                largeChild=leftChild;

            if(top.getKey()>=heapArray[largeChild].getKey())
                break;

            heapArray[index]=heapArray[largeChild];
            index=largeChild;
        }
        heapArray[index]=top;
    }

    public boolean change(int index,int newValue){
        if(index<0||index>=currentSize)
            return false;
        int oldValue=heapArray[index].getKey();
        heapArray[index].setKey(newValue);

        if(oldValue<newValue)
            trickleUp(index);
        else
            trickleDown(index);
        return true;
    }

    public void displayHeap(){
        System.out.print("heapArrary: ");
        for(int m=0;m<currentSize;m++){
            if(heapArray[m]!=null)
                System.out.print(heapArray[m].getKey()+" ");
            else
                System.out.print("-- ");
        }
        System.out.println();

        int nBlacks=32;
        int itemsPerRow=1;
        int column=0;
        int j=0;
        String dots="***********************************";

        while (currentSize>0){
            if(column==0)
                for(int k=0;k<nBlacks;k++)
                    System.out.print(' ');
            System.out.print(heapArray[j].getKey());
            if(++j==currentSize)
                break;

            if(++column==itemsPerRow){
                nBlacks/=2;
                itemsPerRow*=2;
                column=0;
                System.out.println();
            }else
                for(int k=0;k<nBlacks*2-2;k++)
                    System.out.print(' ');
        }
        System.out.println("\n"+dots+dots);
    }
}
```

## GitHub

以上数据结构的完整代码，我均托管在[GitHub](https://github.com/gongfukangEE/Data-Structures-Java)，如果对你有帮助，请给我点个star以示肯定和鼓励。