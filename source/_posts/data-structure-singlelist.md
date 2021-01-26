---
title: 数据结构篇：单链表
tags: []
id: '1088'
categories:
  - - C#
  - - c
    - 数据结构
date: 2019-04-30 20:59:38
---

<meta name="referrer" content="no-referrer" />



### 本程序以类似应用程序的对话框形式进行单链表的操作。希望对大家的学习有所帮助。

![](https://myfirstblog.oss-cn-hangzhou.aliyuncs.com/2019/04/QQ截图20190430205840.png)

```
 #include <iostream>
using namespace std;
typedef int ElemType;

#define EMPTYLIST -1
#define NOEMPTYLIST -2
#define SUCCESS -3
#define FAIL -4
#define ERROR -5
#define DELETESUCCESS 1
#define DELETEFAIL 0

struct Node {
    ElemType data;
    Node *next;
};
class LinkList {
    private :
        Node *Head;
    public :
        LinkList();
        ~LinkList();
        int h_CreateList(int n);//头插法建立单链表
        int t_CreateList(int n);//尾插法建立单链表
        int GetListLength();//得到链表长度
        void ShowList();//显示链表
        int ListInsert(int pos,int e);//向链表插入元素
        int p_ListDelete(int pos);//按位置删除节点
        int e_ListDelete(int e);//按值删除节点
        int Deletelist();//删除整条链表
        int e_ListFind(int e);//按数值查找 位置
        int p_ListFind(int pos);//按位置查找 数值
        int CombineList(LinkList *main,LinkList *beCombined);//求两个链表的并集
        int TestEmpty();//检查链表是否为空
        int TestSuccess();//检查操作是否成功
        int FindRepeat(int e); //检查链表中是否已经存在此数值
        int SplitOddAndEven(LinkList *main,LinkList *beCombined);//分离一个链表，偶数在main，奇数在beCombined

};
LinkList :: LinkList() {
    Head=new Node;
    Head->data=0;
    Head->next=NULL;
}
LinkList::~LinkList() {
    delete Head;
}
int LinkList:: h_CreateList(int n) {
    if(TestEmpty()==NOEMPTYLIST) { //如果不为空，直接返回
        return  NOEMPTYLIST;;
    }
    Node *p,*s;
    p= Head;
    if(n<=0) {
        cout<<"输入有误，请重新输入。"<<endl;
        return FAIL;
    }
    for(int i=1; i<=n; i++) {
        s=new Node;
        cin>>s->data;
        s->next=p->next;//第一次运行此行代码就相当于指定了尾节点为空
        p->next=s;
    }
    return SUCCESS;
}
int LinkList:: t_CreateList(int n) {
    if(TestEmpty()==NOEMPTYLIST) { //如果不为空，直接返回
        return  NOEMPTYLIST;
    }
    Node *p,*s;
    p= Head;
    if(n<=0) {
        cout<<"输入有误，请重新输入。"<<endl;
        return FAIL;
    }
    for(int i=1; i<=n; i++) {
        s=new Node;
        cin>>s->data;
        s->next=p->next;
        p->next=s;
        p=s;
    }
    return SUCCESS;
}
int LinkList :: GetListLength() {
    Node *p;
    int i=0;
    p=Head;
    while(p->next) {
        i++;
        p=p->next;
    }
    return i;
}
void LinkList:: ShowList() {
    Node *p=Head;
    while(p->next) {
        cout<<(p->next)->data<<" ";
        p=p->next;
    }
    cout<<endl;
}
int LinkList::ListInsert(int pos,int e) {
    if(TestEmpty()==EMPTYLIST) { //如果为空，直接返回
        return EMPTYLIST;
    }
    Node *p,*s;
    int i=1;
    p=Head;
    while(i<pos&&p) {
        i++;
        p=p->next;
    }
    if(i!=posp==NULL) {
        cout<<"输入的位置有误，请重新输入。"<<endl;
        return FAIL;
    }
    s=new Node;
    s->data=e;
    s->next=p->next;
    p->next=s;
    return SUCCESS;
}

int LinkList:: p_ListFind(int pos) {
    if(TestEmpty()==EMPTYLIST) { //如果为空，直接返回
        return EMPTYLIST;
    }
    Node *p;
    int i=1;
    p=Head->next;
    while(p&&i<pos) {
        i++;
        p=p->next;
    }
    if(p==NULLi>pos) {
        return FAIL;
    } else {

        return p->data;
    }
}

int LinkList:: e_ListFind(int e) {
    if(TestEmpty()==EMPTYLIST) { //如果为空，直接返回
        return EMPTYLIST;
    }
    Node *p;
    int pos=1;
    p=Head->next;
    while(p&&p->data!=e) { //注意，如果p->data!=e在前，将导致空指针，因为p已为空
        pos++;
        p=p->next;
    }

    if(p==NULL) {
        return FAIL;
    } else {

        return pos;
    }
}
int LinkList:: p_ListDelete(int pos) {
    if(TestEmpty()==EMPTYLIST) { //如果为空，直接返回
        return EMPTYLIST;
    }
    Node *p,*s;
    int i=1;
    p=Head;
    while(p->next&&i<pos) {
        i++;
        p=p->next;
    }
    if(p->next==NULLi>pos) {
        return FAIL;
    } else {
        s=p->next;
        p->next=s->next;
        delete s;
        return SUCCESS;
    }
}
int LinkList:: e_ListDelete(int e) {
    if(TestEmpty()==EMPTYLIST) { //如果为空，直接返回
        return DELETEFAIL;
    }
    int i=1,pos=1,count=0;
    while(e_ListFind(e)!=EMPTYLIST&&e_ListFind(e)!=FAIL) {
        pos=e_ListFind(e);
        p_ListDelete(pos);
        count++;
    }
    if(count!=0)
        return DELETESUCCESS;
    else
        return DELETEFAIL;

}
int LinkList:: Deletelist() {
    Node *p;
    p=Head;
    if(TestEmpty()==EMPTYLIST) { //如果为空，直接返回
        return EMPTYLIST;
    }
    while (p->next) {
        Node *s;
        s=p->next;
        p->next=s->next;
        delete s;
        p=Head;
    }
    cout<<"删除整条链表成功."<<endl;
    return SUCCESS;
}
int LinkList:: CombineList(LinkList *main,LinkList *beCombined) {
    int count,replace,_count,i=1;
    if(main->TestEmpty()==EMPTYLIST) {
        cout<<"请先创建第一个链表："<<endl;
        return FAIL;
    }
    cout<<"请输入想要创建进行并集操作表的数据个数："<<endl;
    cin>>count;
    cout<<"请依次输入"<<count<<"个数据:"<<endl;
    beCombined->t_CreateList(count);
    for (i=1; i<=beCombined->GetListLength(); i++) {
        replace=beCombined->p_ListFind(i);
        _count=main->GetListLength();
        main->ListInsert(++_count,replace);
    }
    for (i=1; i<=main->GetListLength(); i++) {
        main->FindRepeat(main->p_ListFind(i));
    }
    cout<<"并集操作完成。"<<endl;
    return SUCCESS;
}
int LinkList::FindRepeat(int e) {
    Node *p;
    int pos=1,count=1;
    p=Head->next;
    while(p) {
        if(p->data!=e) {
            p=p->next;
            pos++;
        } else if(p->data==e) {
            count++;
            pos++;
            p=p->next;
        }
    }
    if(pos>GetListLength()) {
        return ERROR;
    } else if(count==2) {
        p_ListDelete(pos);
        return SUCCESS;
    }
}
int LinkList:: SplitOddAndEven(LinkList *main,LinkList *beCombined) {

    int count,replace,result,rem=1,i=1;
    if(main->TestEmpty()==EMPTYLIST) {
        cout<<"请先创建第一个链表："<<endl;
        return FAIL;
    }
    if(beCombined->TestEmpty()==EMPTYLIST) {
        Node *p,*s;
        p=beCombined->Head;
        s=new Node;
        s->data=1;
        s->next=p->next;
        p->next=s;
    }
    count=beCombined->GetListLength();
    for (i=1; i<=main->GetListLength(); i=rem) {
        replace=main->p_ListFind(i);
        if(replace%2!=0) {
            main->e_ListDelete(replace);
            beCombined->ListInsert(++count,replace);
        } else {
            rem++;
        }
    }
    for (int i=1;i<=main->GetListLength();i++)
    {
        main->FindRepeat(main->p_ListFind(i));
    }
    beCombined->p_ListDelete(1);
    cout<<"ha链表为："<<endl;
    main->ShowList();
    cout<<"hb链表为："<<endl;
    beCombined->ShowList();
    cout<<"分离操作完成。"<<endl;
    return SUCCESS;

}
void  Menu(LinkList *linkList) {
    int input,result,pos,e,n;
    LinkList beCombined[1];
    cout <<"1.头插法创建单链表          2.尾插法创建单链表   3.获取单链表的长度   \n";
    cout <<"4.判断单链表是否为空        5.按位置获取数值     6.按数值获取位置     \n";
    cout <<"7.在指定位置插入指定元素    8.按位置删除节点     9.按值删除节点       \n" ;
    cout <<"10.删除所有元素             11.拼接两条链表      12.查重              \n";
    cout <<"13.查看链表                 14.分离操作 \n" ;
    cout<<"请输入相应的功能序号："<<endl;
    cin>>input;
    switch (input) {
        case 1:
            cout<<"您想创建含有几个数据元素的单链表？请输入："<<endl;
            cin>>n;
            cout<<"请依次输入"<<n<<"个数据:"<<endl;
            result=linkList->h_CreateList(n);
            while (result==FAIL) {
                cout<<"您想创建含有几个数据元素的单链表？请输入："<<endl;
                cin>>n;
                cout<<"请依次输入"<<n<<"个数据:"<<endl;
                result=linkList->h_CreateList(n);
            }
            if(result==NOEMPTYLIST) {
                cout<<"链表不为空。"<<endl;
            }
            if(result==SUCCESS) {
                cout<<"单链表创建成功。"<<endl;
            }
            Menu(linkList);
            break;
        case 2:
            cout<<"您想创建含有几个数据元素的单链表？请输入："<<endl;
            cin>>n;
            cout<<"请依次输入"<<n<<"个数据:"<<endl;
            result=linkList->t_CreateList(n);
            while (result==FAIL) {
                cout<<"您想创建含有几个数据元素的单链表？请输入："<<endl;
                cin>>n;
                cout<<"请依次输入"<<n<<"个数据:"<<endl;
                result=linkList->t_CreateList(n);
            }
            if(result==NOEMPTYLIST) {
                cout<<"链表不为空。"<<endl;
            }
            if(result==SUCCESS) {
                cout<<"单链表创建成功。"<<endl;
            }
            Menu(linkList);
            break;
        case 3:
            cout<<linkList->GetListLength()<<endl;
            Menu(linkList);
            break;
        case 4:
            result=linkList->TestEmpty();
            if (result==EMPTYLIST) {
                cout<<"链表为空，自动返回主菜单。"<<endl;
            } else
                cout<<"链表不为空，自动返回主菜单。"<<endl;
            Menu(linkList);
            break;
        case 5:
            cout<<"请输入想要寻找的位置："<<endl;
            cin>> pos;
            result=linkList->p_ListFind(pos);
            if (result==FAILresult==EMPTYLIST) {
                cout<<"没有找到您想要的位置，自动返回主菜单"<<endl;
            } else {
                cout<<pos<<"位置的值为："<<result<<endl;
            }
            Menu(linkList);
            break;
        case 6:
            cout<<"请输入想要寻找的数值："<<endl;
            cin>> e;
            result=linkList->e_ListFind(e);
            if (result==FAILresult==EMPTYLIST) {
                cout<<"没有找到您想要的数值，自动返回主菜单"<<endl;
            } else {
                cout<<e<<"所在位置为："<<result<<endl;
            }
            Menu(linkList);
            break;
        case 7:
            cout<<"请输入想要插入的位置："<<endl;
            cin>> pos;
            cout<<"请输入想要插入的数据："<<endl;
            cin>>e;
            result=linkList->ListInsert(pos,e);
            while (result==FAIL) {
                cout<<"请输入想要插入的位置："<<endl;
                cin>> pos;
                cout<<"请输入想要插入的数据："<<endl;
                cin>>e;
                result=linkList->ListInsert(pos,e);
            }
            if(result==SUCCESS) {
                cout<<"数据插入成功。"<<endl;
            }
            Menu(linkList);
            break;
        case 8:
            cout<<"请输入想要删除的位置："<<endl;
            cin>> pos;
            result=linkList->p_ListDelete(pos);
            if (result==FAILresult==EMPTYLIST) {
                cout<<"删除失败，自动返回主菜单"<<endl;
            } else {
                cout<<"第"<<pos<<"节点删除成功。自动返回主菜单。"<<endl;
            }
            Menu(linkList);
            break;
        case 9:
            cout<<"请输入想要删除的数值："<<endl;
            cin>> e;
            result=linkList->e_ListDelete(e);
            if (result==DELETEFAIL) {
                cout<<"删除失败，自动返回主菜单"<<endl;
            } else if(result==DELETESUCCESS) {
                cout<<"值为"<<e<<"节点删除成功。自动返回主菜单。"<<endl;
            }
            Menu(linkList);
            break;
        case 10:
            result=linkList->Deletelist();
            if (result==FAILresult==EMPTYLIST) {
                cout<<"删除失败，自动返回主菜单"<<endl;
            } else {
                cout<<"删除成功，自动返回主菜单"<<endl;
            }
            Menu(linkList);
            break;
        case 11:
            result=linkList->CombineList(linkList,beCombined);
            if (result==FAILresult==EMPTYLIST) {
                cout<<"并集失败，自动返回主菜单"<<endl;
            } else {
                cout<<"并集成功，自动返回主菜单"<<endl;
            }

            Menu(linkList);
            break;
        case 13:
            linkList->ShowList();
            Menu(linkList);
            break;
        case 12:
            cout<<"请输入想要查重的数值："<<endl;
            cin>> e;
            result=linkList->FindRepeat(e);
            if (result==FAILresult==EMPTYLIST) {
                cout<<"查重失败，自动返回主菜单"<<endl;
            } else {
                cout<<"查重成功，自动返回主菜单"<<endl;
            }
            Menu(linkList);
            break;
        case 14:
            result=linkList->SplitOddAndEven(linkList,beCombined);
            if (result==FAILresult==EMPTYLIST) {
                cout<<"分离失败，自动返回主菜单"<<endl;
            } else {
                cout<<"分离成功，自动返回主菜单"<<endl;
            }
            Menu(linkList);
            break;

    }

}
int LinkList::TestEmpty() {
    if(Head->next==NULL) {
        return  EMPTYLIST;
    } else {
        return  NOEMPTYLIST;
    }
}
int main() {
    LinkList linkList[1];
    Menu(linkList);
    return 0;
}
```