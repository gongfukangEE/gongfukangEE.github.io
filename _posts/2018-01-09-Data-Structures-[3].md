---
layout: post
title:  "数据结构[3] -- 无权图"
categories: 数据结构和算法
tags:  数据结构 算法 DFS BFS MST 图
author: G.Fukang
---

* content
{:toc}
无权图的操作集：图的建立、深度优先搜索（DFS）、广度优先搜索（BFS）和最小生成树（MST）







## 无权图  `Graph` [wiki](https://www.wikiwand.com/zh-hans/%E5%9B%BE_(%E6%95%B0%E5%AD%A6))

## 表示图

- 邻接矩阵 `adjMat`      
  邻接矩阵是一个**二维数组**，数据项表示两点之间是否存在边，如果图有N个顶点`Vertex`，邻接矩阵就是`N*N`的数组。
- 邻接表 `verterxList`       
  邻接表是一个链表数组（或者是链表的链表），每个单独的链表表示了有哪些顶点与当前顶点邻接。

## 邻接矩阵建立图

- 插入边`addEdge`
```java
public void addEdge(int start,int end){
        adjMat[start][end]=1;
        adjMat[end][start]=1;
    }
```
- 插入顶点`addVertex`
```java
public void addVertex(char lab){
        verterxList[nVerts++]=new Verterx(lab);
    }
```
## 深度优先搜索（DFS）

![](http://ww1.sinaimg.cn/mw690/005WLTaUly1fmt5vthwzrj309o08pgms.jpg) 

### 方法      
**在搜索到尽头的时候，深度优先搜索用栈记住下一步的走向，图中数字显示了顶点被访问的顺序。**  
找到一个起点，首先访问该顶点，然后把顶点放入栈`Stack`，以便于记住它，最后标记该点，这样就不再访问它。
### 规则
1. 如果可能，访问一个邻接的未访问的顶点，标记它，然后把它放入栈中
2. 当不能执行规则1的时候，如果栈不空，则从栈中弹出一个顶点
3. 当不能执行规则1和规则2的时候，就完成了整个搜索

### 代码片段

```java
public void dsf(){
        vertexList[0].wasVisited=true;
        displayVertex(0);
        theSack.push(0);

        while (!theSack.isEmpty()){
            int v=getAdjUnvisitedVertex(theSack.peek());
            if(v==-1)
                theSack.pop();
            else
            {
                vertexList[v].wasVisited=true;
                displayVertex(v);
                theSack.push(v);
            }
        }

        for(int j=0;j<nVerts;j++)
            vertexList[j].wasVisited=false;
    }

    public int getAdjUnvisitedVertex(int v){
        for(int j=0;j<nVerts;j++)
            if(adjMat[v][j]==1&& vertexList[j].wasVisited==false)
                return j;
        return -1;
    }
```



## 广度优先搜索（BFS）

![](http://ww1.sinaimg.cn/mw690/005WLTaUly1fmt5u6jee1j30dl08vabd.jpg)  



**广度优先搜索借助于队列实现，先找到与起始点相距一条边的所有顶点，然后是与起始点相距两条边的顶点，以此类推，图中数字显示了顶点被访问的顺序。**

### 方法
找到一个起点，首先访问所有与它邻接的顶点，并在访问的同时把他们插到队列`Queue`里，已经没有为访问且与前一个顶点相连接的顶点时，就从队列中取出
### 规则
1. 访问下一个未来访问的邻接点（如果存在），这个顶点必须是当前顶点的邻接点，标记它，并把它插到队列里
2. 如果因为已经没有未访问的顶点而不能执行规则1，那么从队列头取出一个顶点（如果存在），并使其成为当前顶点
3. 如果因为队列为空而不能执行规则2，则搜索结束

### 代码片段

```java
public void dsf(){
        vertexList[0].wasVisited=true;
        displayVertex(0);
        theSack.push(0);

        while (!theSack.isEmpty()){
            int v=getAdjUnvisitedVertex(theSack.peek());
            if(v==-1)
                theSack.pop();
            else
            {
                vertexList[v].wasVisited=true;
                displayVertex(v);
                theSack.push(v);
            }
        }

        for(int j=0;j<nVerts;j++)
            vertexList[j].wasVisited=false;
    }

    public int getAdjUnvisitedVertex(int v){
        for(int j=0;j<nVerts;j++)
            if(adjMat[v][j]==1&& vertexList[j].wasVisited==false)
                return j;
        return -1;
    }
```



## 最小生成树（MST）

**最少的边连接所有的顶点** （基于**BFS**或者**DFS**），最小生成树比较 容易从深度优先搜索得到，这是因为`DFS`访问所有的顶点，但是只访问一次，因此深度优先搜索算法走过整个图的路径必定是最小生成树。

### 代码片段

```java
public void mst() {
        vertexList[0].wasVisited = true;
        theSack.push(0);

        while (!theSack.isEmpty()) {
            int currentVertex = theSack.peek();
            int v = getAdjUnvisitedVertex(currentVertex);

            if (v == -1)
                theSack.pop();
            else {
                vertexList[v].wasVisited = true;
                theSack.push(v);

                displayVertex(currentVertex);
                displayVertex(v);
                System.out.print(" ");
            }
        }

        for (int j = 0; j < nVerts; j++)
            vertexList[j].wasVisited = false;
    }
```

## Github

完整代码，我托管在[Github](https://github.com/gongfukangEE/Data-Structures-Java/tree/master/src/Graph)上，如果对你有帮助，请给我点个star以示肯定和鼓励。



