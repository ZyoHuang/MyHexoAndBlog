---
title: 图形学篇：多边形有效边表填充算法
tags: []
id: '1016'
categories:
  - - 图形学
date: 2019-04-30 18:29:57
---

<meta name="referrer" content="no-referrer" />



### 什么是多边形？

*   **多边形是由折线段组成的封闭图形**

### 多边形的表示方法有哪些？\[toc\]

*   **顶点表示法：使用顶点进行描述，但此时多边形仅仅是一些封闭线段，内部是空的，且不能直接进行填充上色**
*   **点阵表示法：使用大量的点进行描述，描述完成之后，得到的就是完整形态的多边形，内部已被填充，可直接针对点来进行上色**
*   **多边形的扫描转换就是从顶点表示法转换到点阵表示法的过程。**

### 基础的填充多边形方式：

*   **检查光栅上的每一个像素是否位于多边形内**

### 光栅究竟是什么？

*   **由大量等宽等间距的平行狭缝构成的光学器件称为光栅，这是专业且准确的方法，然而明显不是给人看的（观众：？？？）**
*   **光栅是连接帧缓冲和具体的电脑屏幕的桥梁（这是很老的大头显示器上的，现在的液晶显示器不存在光栅，它的成像依靠的是电场，液晶，滤光膜等，所以我们暂且把这里说的的光栅理解为像素）**

### 光栅化究竟是什么？

*   **[https://blog.csdn.net/waitforfree/article/details/10066547](https://blog.csdn.net/waitforfree/article/details/10066547)**
*   **光栅化是一切屏幕成像的基础，没有它，就没有图像**
*   **光栅化不依赖于光栅，它依赖于CPU和GPU的交互和运算**

### 有效边表填充算法：

*   **基本原理：按照扫描线从小到大的移动顺序，计算当前扫描线与有效边的交点，然后把这些交点按x的值递增顺序进行排序，配对，以确定填充去间，最后用指定颜色填充区间内的所有像素，即完成填充工作**
*   **优势：通过维护边表和有效边表，避开了扫描线与多边形所有边求交的复杂运算，性能提升巨大**
*   **边界像素处理原则：对于边界像素，采用“`左闭右开`”和“`下闭上开`”的原则**
*   **有效边：多边形与当前扫描线相交的边**
*   **有效边表：把有效边按照与扫描线交点x坐标递增的顺序存放在一个链表中，称为有效边表**
*   **桶表：按照扫描线顺序管理边的数据结构** **算法实现：** **将VC 6.0 调整到ClassView视图**
    

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190419205912252.png) **创建有效边表和桶表类** ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190419205951167.png) ![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190419210009252.png) **将VC 6.0调整到FileView视图，对两个类进行定义**

### AET.h

```
 class CAET  
{
public:
    CAET();
    virtual ~CAET();
public:
    double  x;//当前扫描线与有效边交点的x坐标
    int     yMax;//边的最大y值
    double  k;//斜率的倒数(x的增量)
    CAET   *pNext;
};

```

### Bucket.h

```
 #include "AET.h"
#include "Bucket.h"

class CBucket  
{
public:
    CBucket();
    virtual ~CBucket();
public:
    int     ScanLine;     //扫描线
    CAET    *pET;         //桶上的边表指针
    CBucket *pNext;
};
```

**回到ClassView视图，像之前那样创建CFill类，再回到FileView视图，对CFill类进行定义**

### Fill.h

```
 #include "AET.h"
#include "Bucket.h"

class CFill  
{
public:
    CFill();
    virtual ~CFill();
    void SetPoint(CPoint *p,int);//初始化顶点和顶点数目
    void CreateBucket();//创建桶
    void CreateEdge();//边表
    void AddET(CAET *);//合并ET表
    void ETOrder();//ET表排序
    void Gouraud(CDC *);//填充多边形
    void ClearMemory();//清理内存
    void DeleteAETChain(CAET* pAET);//删除边表
protected:
    int     PNum;//顶点个数
    CPoint    *P;//顶点坐标动态数组
    CAET    *pHeadE,*pCurrentE,*pEdge; //有效边表结点指针
    CBucket *pHeadB,*pCurrentB;        //桶表结点指针
};
```

### Fill.cpp

```
 // Fill.cpp: implementation of the CFill class.
//
//////////////////////////////////////////////////////////////////////

#include "stdafx.h"
#include "Study3.h"
#include "Fill.h"
#include "AET.h"
#include "Bucket.h"

#ifdef _DEBUG
#undef THIS_FILE
static char THIS_FILE[]=__FILE__;
#define new DEBUG_NEW
#endif

//////////////////////////////////////////////////////////////////////
// Construction/Destruction
//////////////////////////////////////////////////////////////////////
CFill::CFill()
{
    PNum=0;
    P=NULL;
    pEdge=NULL;
    pHeadB=NULL;
    pHeadE=NULL;
    pCurrentB = NULL;
    pCurrentE = NULL;
}

CFill::~CFill()
{
    if(P!=NULL)
    {
        delete[] P;
        P=NULL;
    }
    ClearMemory();
}

void CFill::SetPoint(CPoint *p,int m)////初始化顶点和顶点数目
{
    P=new CPoint[m];//创建一维动态数组，转储CTestView类中P数组中的顶点
    for(int i=0;i<m;i++)
    {
        P[i]=p[i];  
    }
    PNum=m;//记录顶点数量
}

void CFill::CreateBucket()//创建桶表
{
    int yMin,yMax;
    yMin = yMax = P[0].y;
    for(int i = 0;i<PNum;i++)
    {
        if(P[i].y<yMin)
        {
            yMin = P[i].y;

        }
        if(P[i].y>yMax)
        {
            yMax = P[i].y;
        }
    }

    for(int y = yMin;y<=yMax;y++)
    {
        if(yMin == y)
        {
            pHeadB = new CBucket;
            pCurrentB = pHeadB;
            pCurrentB->ScanLine = yMin;
            pCurrentB->pET = NULL;
            pCurrentB->pNext = NULL;
        }
        else
        {
            pCurrentB->pNext = new CBucket;
            pCurrentB=pCurrentB->pNext;
            pCurrentB->ScanLine = y;
            pCurrentB->pET=NULL;
            pCurrentB->pNext=NULL;
        }
    }
}

void CFill::CreateEdge()//创建边表，即将各边链入到相应的桶节点
{
    for(int i = 0;i<PNum;i++)
    {
        pCurrentB=pHeadB;
        int j = (i+1)%PNum;
        if(P[i].y<P[j].y)
        {
            pEdge = new CAET;
            pEdge->x=P[i].x;
            pEdge->yMax=P[j].y;
            pEdge->k = (double)(P[j].x-P[i].x)/((double)(P[j].y-P[i].y));

            pEdge->pNext = NULL;
            while(pCurrentB->ScanLine!=P[i].y)
            {
                pCurrentB=pCurrentB->pNext;
            }
        }

        if(P[j].y<P[i].y)
        {
            pEdge=new CAET;
            pEdge->x=P[j].x;
            pEdge->yMax=P[i].y;
            pEdge->k = (double)(P[i].x-P[j].x)/((double)(P[i].y-P[j].y));
            pEdge->pNext = NULL;
            while(pCurrentB->ScanLine!=P[j].y)
            {
                pCurrentB=pCurrentB->pNext;
            }
        }

        if(P[j].y!=P[i].y)
        {
            pCurrentE=pCurrentB->pET;
            if(pCurrentE==NULL)
            {
                pCurrentE=pEdge;
                pCurrentB->pET=pCurrentE;
            }
            else
            {
                while(NULL!=pCurrentE->pNext)
                {
                    pCurrentE=pCurrentE->pNext;
                }
                pCurrentE->pNext=pEdge;
            }
        }
    }
}
void CFill::AddET(CAET *pNewEdge)//合并ET表
{
    CAET *pCE=pHeadE;//边表头结点
    if(pCE==NULL)//若边表为空，则pNewEdge作为边表头结点
    {
        pHeadE=pNewEdge;
        pCE=pHeadE;
    }
    else//将pNewEdge链接到边表末尾（未排序）
    {
        while(pCE->pNext!=NULL)
        {
            pCE=pCE->pNext;
        }
        pCE->pNext=pNewEdge;
    }
}

void CFill::ETOrder()//边表的冒泡排序算法
{
    CAET *pT1,*pT2;
    int Count=1;
    pT1=pHeadE;
    if(pT1==NULL)//没有边，不需要排序
    {
        return;
    }
    if(pT1->pNext==NULL)//如果该ET表没有再连ET表
    {
        return;//只有一条边，不需要排序
    }
    while(pT1->pNext!=NULL)//统计边结点的个数
    {
        Count++;
        pT1=pT1->pNext;
    }
    for(int i=0;i<Count-1;i++)//冒泡排序
    {
        CAET **pPre=&pHeadE;//pPre记录当面两个节点的前面一个节点，第一次为头节点
        pT1=pHeadE;
        for (int j=0;j<Count-1-i;j++)
        {
            pT2=pT1->pNext;

            if ((pT1->x>pT2->x)((pT1->x==pT2->x)&&(pT1->k>pT2->k)))//满足条件，则交换当前两个边结点的位置
            {
                pT1->pNext=pT2->pNext;
                pT2->pNext=pT1;
                *pPre=pT2;
                pPre=&(pT2->pNext);//调整位置为下次遍历准备
            }
            else//不交换当前两个边结点的位置，更新pPre和pT1
            {
                pPre=&(pT1->pNext);
                pT1=pT1->pNext;
            }
        }
    }
}

void CFill::Gouraud(CDC *pDC)//填充多边形
{
    CAET *pT1=NULL,*pT2=NULL;
    pHeadE=NULL;
    for(pCurrentB=pHeadB;pCurrentB!=NULL;pCurrentB=pCurrentB->pNext)
    {
        for(pCurrentE=pCurrentB->pET;pCurrentE!=NULL;pCurrentE=pCurrentE->pNext)
        {
            pEdge=new CAET;
            pEdge->x=pCurrentE->x;
            pEdge->yMax=pCurrentE->yMax;
            pEdge->k=pCurrentE->k;
            pEdge->pNext=NULL;
            AddET(pEdge);
        }

        ETOrder();
        pT1=pHeadE;
        if(pT1==NULL)
        {
            return ;
        }
        while(pCurrentB->ScanLine>=pT1->yMax)
        {
            CAET *pAETTEmp = pT1;
            pT1=pT1->pNext;
            delete pAETTEmp;
            pHeadE=pT1;
            if(pHeadE==NULL)
            {
                return;
            }

        }
        if(pT1->pNext!=NULL)
        {
            pT2=pT1;
            pT1=pT2->pNext;
        }

        while(pT1!=NULL)
        {
            if(pCurrentB->ScanLine>=pT1->yMax)
            {
                CAET *pAETTemp=pT1;
                pT2->pNext=pT1->pNext;
                pT1=pT2->pNext;
                delete pAETTemp;
            }
            else
            {
                pT2=pT1;
                pT1=pT2->pNext;
            }
        }
        BOOL In = FALSE;
        int xb,xe;
        for(pT1=pHeadE;pT1!=NULL;pT1=pT1->pNext)
        {
            if(FALSE==In)
            {
                xb = (int)pT1->x;
                In=TRUE;
            }
            else
            {
                xe = (int)pT1->x;
                for(int x = xb;x<xe;x++)
                {
                    pDC->SetPixel(x,pCurrentB->ScanLine,RGB(0,0,255));
                }
                In = FALSE;

            }
        }
        for(pT1=pHeadE;pT1!=NULL;pT1=pT1->pNext)
        {
            pT1->x=pT1->x+pT1->k;
        }
    }
}


void CFill::ClearMemory()//安全删除所有桶与桶上连接的边
{
    DeleteAETChain(pHeadE);//删除边表
    CBucket *pBucket=pHeadB;
    while (pBucket!=NULL)//针对每一个桶
    {
        CBucket *pBucketTemp=pBucket->pNext;
        DeleteAETChain(pBucket->pET);//删除桶上面的边
        delete pBucket;
        pBucket=pBucketTemp;
    }
    pHeadB=NULL;
    pHeadE=NULL;
}

void CFill::DeleteAETChain(CAET *pAET)//删除边表
{
    while (pAET!=NULL)
    {
        CAET *pAETTemp=pAET->pNext;
        delete pAET;
        pAET=pAETTemp;
    }
}
```

**一切准备就绪，进入CXXXView.cpp（“XXX”为你项目名称）** **给OnDraw函数添加以下语句**

```
    CRect rect;
    GetClientRect(&rect);
    pDC->SetMapMode(MM_ANISOTROPIC);
    pDC->SetWindowExt(rect.Width(),rect.Height());
    pDC->SetViewportExt(rect.Width(),-rect.Height());
    pDC->SetViewportOrg(rect.Width()/2,rect.Height()/2);
    rect.OffsetRect(-rect.Width()/2,-rect.Height()/2);

    //声明Fill类
    CFill *cFill = new CFill;

    //声明多边形的七个顶点
    CPoint points[7] = {CPoint(50,70),CPoint(-150,270),CPoint(-250,20),CPoint(-150,-280),CPoint(0,-80),CPoint(100,-280),CPoint(300,120)};

    //设置顶点
    cFill->SetPoint(points,7);

    //创建桶表
    cFill->CreateBucket();

    //创建边表
    cFill->CreateEdge();

    //填充多边形
    cFill->Gouraud(pDC);
```

### 运行结果

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/20190419211002177.png)

### 完整程序地址：[https://gitee.com/NKG\_admin/graphics\_project\_library.git](https://gitee.com/NKG_admin/graphics_project_library.git "https://gitee.com/NKG_admin/graphics_project_library.git")

**大家对于填充算法要多看看，很重要**