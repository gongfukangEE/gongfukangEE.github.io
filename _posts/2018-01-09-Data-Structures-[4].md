---
layout: post
title:  "数据结构[4] -- 带权图"
categories: 数据结构和算法
tags:  数据结构 算法
author: G.Fukang
---

* content
{:toc}
带权图的操作集：最小生成树（MSTW）和最短路径（Dijkstra）



## 带权图 `GraphW` 

## 最小生成树（MSTW） 

有权图的最小生成树是用**优先级队列**来实现的，**优先级队列** 可以基于**堆**来实现，也可以基于**数组**实现。

![](http://ww1.sinaimg.cn/mw690/005WLTaUly1fmu0v6hhlcj30cq08dmz6.jpg)

### 方法

1. 从一个顶点开始，放到树的集合中，循环执行下面步骤
2. 找到从最新的顶点到其他顶点的的所有边，这些顶点不在树的集合中，将这些边放到优先级队列
3. 找到权值最小的边，把它和它所到达的顶点都放到树的集合中

### 顶点分类
- 已经被连接的顶点，它们是最小生成树的顶点
- 还没有被链接的顶点，但是知道把他们连接到至少一个已知顶点的权重，
  这些叫做边缘顶点
- 还不知道任何信息的顶点

### 边的剪除

在算法中，要确保优先级队列不能有连接在已在树中的的顶点的边，每次向树中增加顶点后，都要遍历优先级队列查找并删除这样的边，也就是优先级队列中应该只包含一条到达某个第二类顶点的边         
![](http://ww1.sinaimg.cn/mw690/005WLTaUly1fmu0w0c4mmj310o08uacj.jpg)   

`putInPQ`：保证优先级队列中应该只包含一条到达某个特定目标顶点的边     

1. 调用`find()`方法，寻找优先队列中是否存在可以到达指定节点的边，
  如果存在，返回其所在队列的位置，不存在则返回`-1`
2. 存在：如果`oldDist`大于`newDist`,则在队列中删除`removeN`前者所在`tempEdge`，插入`insert`后者`theEdge`；
  如果`oldDist`小于或等于`newDist`，则队列不作更改
3. 不存在：直接`insert`插入`Edge`  

### 算法具体实现

1. 当前顶点`currentVert`放在树中
2. 连接这个顶点的边放到优先级队列中`putInPQ`,但是只要一下任意一个条件满足，这条边就不可以放入队列中        
- 源点和终点相同
- 终点在树中
- 源点和终点之间没有边（邻接矩阵中对应的值为`INFINITY`）

3.将权值最小的边从优先级队列中删除`removeMin`，把这条边和该边的终点加入树,并显示源点和终点。

### 代码片段

```java
public void mstw(){
        currentVert=0;

        while (nTree<nVerts){
            vertexList[currentVert].isInTree=true;
            nTree++;

            for(int j=0;j<nVerts;j++){
                if(j==currentVert)
                    continue;
                if(vertexList[j].isInTree)
                    continue;
                int distance=adjMat[currentVert][j];
                if(distance==INFINITY)
                    continue;
                putInPQ(j,distance);
            }
            if(thePQ.size()==0){
                System.out.println("GRAPH NOT CONNECTED");
                return;
            }

            Edge theEdge=thePQ.removeMin();
            int sourceVert=theEdge.srcVert;
            currentVert=theEdge.destVert;

            System.out.print(vertexList[sourceVert].label);
            System.out.print("->");
            System.out.print(vertexList[currentVert].label);
            System.out.print("  ");
        }

        for(int j=0;j<nVerts;j++)
            vertexList[j].isInTree=false;
    }

    public void putInPQ(int newVert,int newDist){
        int queueIndex=thePQ.find(newVert);
        if(queueIndex!=-1){
            Edge tempEdge=thePQ.peekN(queueIndex);
            int oldDist=tempEdge.distance;
            if(oldDist>newDist){
                thePQ.removeN(queueIndex);
                Edge theEdge=new Edge(currentVert,newVert,newDist);
                thePQ.insert(theEdge);
            }
        }else{
            Edge theEdge=new Edge(currentVert,newVert,newDist);
            thePQ.insert(theEdge);
        }
    }
```



## 最短路径（Dijkstra）

**最短路径**：寻找带权图中给定某两点间的最短路径（代码基于有向带权图）。

![](http://ww1.sinaimg.cn/mw690/005WLTaUly1fmu9utdv5pj30eq0c2dj2.jpg)  

### 方法

- 每次将一个新顶点放入树，用这个新顶点不断修正记录，在记录中只保留从源顶点到某个已知顶点的最低费用
- 不断将新的顶点放入树，条件是**从源点到那个城市的路线费用最低**

### 顶点分类

- 已经被连接的顶点，它们是已在树中
- 还没有被链接的顶点，但是知道把他们连接到至少一个已知顶点的权重，这些叫做边缘顶点
- 还不知道任何信息的顶点  

### 算法步骤

1. 将源顶点放入树中，并记录此顶点到其他可直达顶点的权值，不可直达为`INFINITY`
2. 取权值最小的顶点，并分别记录源顶点到其他顶点（**源顶点直达+源顶点经此顶点到达**）的**最小**权值
3. 确定第二个顶点，将其放入树中
4. 循环2到3
5. 到达目标顶点，结束 

![](http://ww1.sinaimg.cn/mw690/005WLTaUly1fmub7ezpayj30zo07c40p.jpg)

### 算法解析

- `sPath[]`数组存储从源点开始到其他顶点（终点）的最短距离。

- 不仅记录从源顶点到每个终点的最短路径，还应该记录走过的路径，即记录终点的父节点`DisPar`。
   - **C->E[140]**：`C`是起始顶点，`E`终止顶点，`[150]`表示从源节点到`E`的最短路径的权值和，`C->E`表示路径

- `path()`选择源顶点放入树中，并记录`sPath`，开始循环
  1. 选择`sPath[]`数组的最小距离
  2. 把对应的顶点放入树中，这个顶点变成新顶点`currentVert`
  3. 根据`currentVert`变化，更新`adjust_sPath()`所有的`sPath[]`数组内容

- `adjust_sPath()`当例程被调用的时候，`currentVert`被放入树中，`startToCurrent`是`sPath[]`数组中的当前项。
  `adjust_sPath()`方法现在检测`sPath[]`数组的每一项，它使用循环计数器`column`依次指向每一个顶点，对于`sPath[]`数组的每一项，如果顶点不在树中，就做以下三件事：
  - 把当前顶点的距离`startToCurrent`加到从源顶点到当前顶点的距离上`startToFringe`
  - 把`startToFringe`与`sPath[]`数组当前项做做比较
  - 如果`startToFringe`更小，就替换当前项

### 代码片段

  ```java
  public void path(){
          int startTree=0;
          vertexList[startTree].isInTree=true;
          nTree=1;

          for(int j=0;j<nVerts;j++){
              int tempDist=adjMat[startTree][j];
              sPath[j]=new DisPar(startTree,tempDist);
          }

          while (nTree<nVerts){
              int indexMin=getMin();
              int minDist=sPath[indexMin].distance;

              if(minDist==INFINITY){
                  System.out.println("There are unreachable vertices");
                  break;
              }else{
                  currentVert=indexMin;
                  startToCurreent=sPath[indexMin].distance;
              }

              vertexList[currentVert].isInTree=true;
              nTree++;
              adjust_sPath();
          }

          displayPaths();

          nTree=0;
          for(int j=0;j<nVerts;j++)
              vertexList[j].isInTree=false;
      }

      public void adjust_sPath(){
          int column=1;
          while (column<nVerts){
              if(vertexList[column].isInTree){
                  column++;
                  continue;
              }

              int currentToFringe=adjMat[currentVert][column];
              int startToFringe=startToCurreent+currentToFringe;
              int sPathDist=sPath[column].distance;

              if(startToFringe<sPathDist){
                  sPath[column].parentVert=currentVert;
                  sPath[column].distance=startToFringe;
              }
              column++;
          }
      }
  ```

  

## Github

完整代码，我托管在[Github](https://github.com/gongfukangEE/Data-Structures-Java/tree/master/src/GraphW)上，如果对你有帮助，请给我点个star以示肯定和鼓励。
