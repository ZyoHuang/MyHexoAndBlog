---
title: 数据结构篇：邻接表
tags: []
id: '1083'
categories:
  - - C#
  - - c
    - 数据结构
date: 2019-04-30 20:56:43
---

<meta name="referrer" content="no-referrer" />



**每一个顶点后面就是一条链表，每个顶点都存在数组里。** 以这张图为例 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20181117121212269.png) 结构如下 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20181117122022198.png) 运行截图 ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20181121213442224.png) 结构体定义

```
 //边表结点
typedef struct EdgeNode {
    //顶点对应的下标
    int adjvex;
    //指向下一个邻接点
    struct EdgeNode *next;
} edgeNode;

//顶点表结点
typedef struct VertexNode {
    //顶点数据
    char data;
    //边表头指针
    edgeNode *firstedge;
} VertexNode, AdjList[100];

//集合
typedef struct {
    AdjList adjList;
    //顶点数和边数
    int numVertexes, numEdges;
} GraphAdjList;
```

### 完整程序

```
 #include <iostream>

using namespace std;
//边表结点
typedef struct EdgeNode {
    //顶点对应的下标
    int adjvex;
    //指向下一个邻接点
    struct EdgeNode *next;
} edgeNode;

//顶点表结点
typedef struct VertexNode {
    //顶点数据
    char data;
    //边表头指针
    edgeNode *firstedge;
} VertexNode, AdjList[100];

//集合
typedef struct {
    AdjList adjList;
    //顶点数和边数
    int numVertexes, numEdges;
} GraphAdjList;

class AdjacencyList {
public:

    void CreateALGraph(GraphAdjList *G);

    void ShowALGraph(GraphAdjList *G);

    void Test();
};

void AdjacencyList::CreateALGraph(GraphAdjList *G) {
    int i, j, k;
    edgeNode *e;
    cout << "输入顶点数和边数" << endl;
    //输入顶点数和边数
    cin >> G->numVertexes >> G->numEdges;
    //读入顶点信息，建立顶点表
    for (i = 0; i < G->numVertexes; i++)
    {
        //输入顶点信息
        cin >> G->adjList[i].data;
        //将边表置为空表
        G->adjList[i].firstedge = NULL;
    }
    //建立边表（头插法）
    for (k = 0; k < G->numEdges; k++)
    {
        cout << "输入边（vi,vj）上的顶点序号" << endl;
        cin >> i >> j;
        e = new EdgeNode;
        e->adjvex = j;
        e->next = G->adjList[i].firstedge;
        G->adjList[i].firstedge = e;

        e = new EdgeNode;

        e->adjvex = i;
        e->next = G->adjList[j].firstedge;
        G->adjList[j].firstedge = e;
    }
}

void AdjacencyList::Test() {
    cout << "ALL IS OK." << endl;
}

void AdjacencyList::ShowALGraph(GraphAdjList *G) {
    for (int i = 0; i < G->numVertexes; i++)
    {
        cout << "顶点" << i << ": " << G->adjList[i].data<<"--firstedge--";
        edgeNode *p = new edgeNode;
        p = G->adjList[i].firstedge;
        while (p)
        {
            cout<<p->adjvex<<"--Next--";
            p=p->next;
        }
        cout<<"--NULL"<<endl;
    }

}

int main() {
    AdjacencyList adjacencyList;
    GraphAdjList *GA = new GraphAdjList;
    adjacencyList.Test();
    adjacencyList.CreateALGraph(GA);
    adjacencyList.ShowALGraph(GA);
    return 0;
}

```