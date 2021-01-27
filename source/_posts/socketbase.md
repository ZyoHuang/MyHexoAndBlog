---
title: Socket学习笔记
date:
updated:
tags: [网络编程, Socket]
categories:
  - - GamePlay
    - 网络通信
keywords:
top_img:
cover:
aplayer:
---
<meta name="referrer" content="no-referrer" />

### 源代码：
**[https://gitee.com/NKG_admin/SocketTestDemo](https://gitee.com/NKG_admin/SocketTestDemo)**

### Socket 基础知识

- **[https://blog.csdn.net/fighting_xa/article/details/50623571](https://blog.csdn.net/fighting_xa/article/details/50623571)** 
- **[http://liuliliujian.iteye.com/blog/898342](http://liuliliujian.iteye.com/blog/898342)** 
- **[https://baike.baidu.com/item/socket/281150?fr=aladdin](https://baike.baidu.com/item/socket/281150?fr=aladdin)** 
- **看不懂没关系，因为我也看不懂。（旁白：滚！）** 

 **既然谈到Socket，就得牵扯到服务器端，什么是服务器端呢？**

 **个人理解：服务器端就是自己的代码，让它跑在云主机（云服务器上，比如阿里云，亚马逊云这些）。而这些，其实就是在云端买了一个主机，它和你正在使用的电脑一样，有桌面，有系统，有蜘蛛纸牌。。。**

 **不同的是，它有固定的公网IP，而我们电脑公网IP是变化的（所以如果要把自己电脑变成服务器的话需要用花生壳做内网穿透）。**

 **所以我们要把代码里绑定IP和连接IP的地方改成服务器的公网IP就行了。然后只需要把你服务器代码生成的exe文件在上面打开。让它365天没日没夜开着机就行了。**

### **那么Socket到底是个啥？**

 **先看一张图**

 **![](https://note.youdao.com/yws/api/personal/file/WEBab1649ae0df08cba145698384485f922?method=download&shareKey=bdab5ee7732e1bd3642a58d32ee0e6a5)**

 **Socket是一个接口，是应用层与TCP/IP协议族通信的中间软件抽象层，在用户进程与TCP/IP协议之间充当中间人，完成TCP/IP协议的书写，用户只需理解接口即可。 **

 **在设计模式中，Socket其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议。它的出现只是使得程序员更方便地使用TCP/IP协议栈而已。Socket本身并不是协议，它是应用层与TCP/IP协议族通信的中间软件抽象层，是一组调用接口（TCP/IP网络的API函数）。**

### **TCP/IP协议又是个啥？**

 **![](https://note.youdao.com/yws/api/personal/file/WEB94e3ec14ec0ee98285c53f0e57763de4?method=download&shareKey=279a59131490615c34a06426806c16b3)**

 

 **从上图可以看出可以看出TCP/IP协议族包括运输层、网络层、链路层。TCP/IP只是一个协议栈，就像操作系统的运行机制一样，必须要具体实现，同时还要提供对外的操作接口。就像操作系统会提供标准的编程接口，比如Win32编程接口一样，TCP/IP也必须对外提供编程接口，这就是Socket编程接口**

 

 **Socket跟TCP/IP并没有必然的联系。Socket编程接口在设计的时候，就希望也能适应其他的网络协议。所以，Socket的出现只是可以更方便的使用TCP/IP协议栈而已，其对TCP/IP进行了抽象，形成了几个最基本的函数接口。比如create，listen，accept，connect，read和write等等。**

 **socket只是对TCP/IP协议栈操作的抽象，而不是简单的映射关系，这很重要！**

### 通俗点讲：

 **可以这样理解，你可以简单的理解为电话号码。你这边一个电话号码 发送信息，另一个电话号码接收你发送的消息。而中间传输这些信息的过程和技术就是TCP/IP，我们不需要了解，我们只需要知道要拨打的号码（Socket ip地址和端口）即可。你用的这个号码指定发给哪个号码，就只有哪个号码可以接收你发送的消息。这两个电话可收信息，可发信息。就是担任着Socket的角色。两个手机就是你用的电脑了。Socket也一样，指定了[ip](https://www.baidu.com/s?wd=ip&tn=SE_PcZhidaonwhc_ngpagmjz&rsv_dl=gh_pc_zhidao)和端口就变成独一无二的电话号码了。**

### Message协议与解析代码
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
 
namespace ClientTest
{
    class Message
    {
        private byte[] data = new byte[1024];
        private int startIndex = 0;//我们存取了多少个字节的数据在数组里面
        public void AddCount(int count)
        {
            startIndex += count;
        }
        public byte[] Data
        {
            get { return data; }
        }
        public int StartIndex
        {
            get
            {
                return startIndex;
            }
        }
        public int RemindSize
        {
            get
            {
                return data.Length - startIndex;
            }
        }
        //解析数据
        public void ReadMessage()
        {
            while (true)
            {
                if (startIndex <= 4)
                {
                    return;
                }
                int count = BitConverter.ToInt32(data, 0);
                if ((startIndex - 4) >= count)
                {
                    string s = Encoding.UTF8.GetString(data, 4, count);
                    Console.WriteLine("解析出来一条数据：" + s);
                    Array.Copy(data, count + 4, data, 0, startIndex - 4 - count);
                    startIndex -= (count + 4);
                }
                else
                {
                    break;
                }
            }
 
        }
        /// <summary>
        /// 得到数据的约定形式
        /// </summary>
        /// <param name="data"></param>
        /// <returns></returns>
        public static byte[] GetBytes(string data)
        {
            byte[] dataBytes = Encoding.UTF8.GetBytes(data);
            int dataLength = dataBytes.Length;
            byte[] lengthBytes = BitConverter.GetBytes(dataLength);
            byte[] Bytes = lengthBytes.Concat(dataBytes).ToArray();
            return Bytes;
        }
 
 
    }
}

```
### 服务端代码
```csharp
using System;
using System.Net;
using System.Net.Sockets;
using System.Text;
 
namespace Server
{
    class Program
    {
        /// <summary>
        /// 实例化Message
        /// </summary>
        static Message rec_Message = new Message();
        static Socket serverSocket;
        static void Main(string[] args)
        {
            StartServer();
            //暂停
            Console.ReadKey();
        }
        /// <summary>
        /// 开启一个Socket
        /// </summary>
        static void StartServer()
        {
            //实例化一个Socket
            serverSocket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
 
            //设置IP
            IPAddress ipAdress = IPAddress.Parse("127.0.0.1");
 
            //设置网络终结点
            IPEndPoint iPEndPoint = new IPEndPoint(ipAdress, 88);
 
            //绑定ip和端口号
            serverSocket.Bind(iPEndPoint);
 
            //等待队列(开始监听端口号)
            serverSocket.Listen(0);
 
            //异步接受客户端连接
            serverSocket.BeginAccept(AcceptCallBack, serverSocket);
 
        }
 
 
        /// <summary>
        /// 开始发送数据到客户端
        /// </summary>
        /// <param name="toClientsocket">用以连接客户端的Socket</param>
        /// <param name="msg">要传递的数据</param>
        static void BeginSendMessagesToClient(Socket toClientsocket,string msg)
        {
            toClientsocket.Send(Message.GetBytes(msg));
        }
 
        /// <summary>
        /// 开始接收来自客户端的数据
        /// </summary>
        /// <param name="toClientsocket"></param>
        static void BeginReceiveMessages(Socket toClientsocket)
        {
            toClientsocket.BeginReceive(rec_Message.Data, rec_Message.StartIndex, rec_Message.RemindSize, SocketFlags.None, ReceiveCallBack, toClientsocket);
        }
 
        /// <summary>
        /// 当客户端连接到服务器时执行的回调函数
        /// </summary>
        /// <param name="ar"></param>
        static void AcceptCallBack(IAsyncResult ar)
        {
            //这里获取到的是向客户端收发消息的Socket
            Socket toClientsocket = serverSocket.EndAccept(ar);
            Console.WriteLine("客户端{0}连接进来了。",toClientsocket.RemoteEndPoint.ToString());
            //要向客户端发送的消息    
            string msg = "Hello client!你好。";
            //开始发送消息
            BeginSendMessagesToClient(toClientsocket, msg);
            //开始接收客户端传来的消息
            BeginReceiveMessages(toClientsocket);
            ////继续等待下一个客户端的链接
            serverSocket.BeginAccept(AcceptCallBack, serverSocket);
        }
 
        /// <summary>
        /// 接收到来自客户端消息的回调函数
        /// </summary>
        /// <param name="ar"></param>
        static void ReceiveCallBack(IAsyncResult ar)
        {
            Socket toClientsocket = null;
            try
            {
                toClientsocket = ar.AsyncState as Socket;
                int count = toClientsocket.EndReceive(ar);
                if (count == 0)
                {
                    toClientsocket.Close();
                    return;
                }
                Console.WriteLine("从客户端：{0} 接收到数据,解析中。。。",toClientsocket.RemoteEndPoint);
                rec_Message.AddCount(count);
                //打印来自客户端的消息
                rec_Message.ReadMessage();
                BeginSendMessagesToClient(toClientsocket, "客户端"+toClientsocket.RemoteEndPoint+"我收到了你消息。");
                //继续监听来自客户端的消息
                toClientsocket.BeginReceive(rec_Message.Data, rec_Message.StartIndex, rec_Message.RemindSize, SocketFlags.None, ReceiveCallBack, toClientsocket);
            }
            catch (Exception e)
            {
                Console.WriteLine(e);
                if (toClientsocket != null)
                {
                    toClientsocket.Close();
                }
            }
            finally
            {
 
            }
 
        }
 
 
    }
}

```

### 客户端代码

```csharp
using System;
using System.Text;
using System.Net.Sockets;
using System.Net;
 
namespace ClientTest
{
    class Program
    {
        /// <summary>
        /// 实例化Message
        /// </summary>
        static Message rec_Message = new Message();
        //声明客户端
        static Socket clientSocket;
        static void Main(string[] args)
        {
            StartClient();
            while (true)
            {
                Console.WriteLine("请输入想要向服务器发送的字符串：");
                string data = Console.ReadLine();
                BeginSendMessagesToServer(data);
            }
        }
        /// <summary>
        /// 开启客户端并连接到服务器端
        /// </summary>
        static void StartClient()
        {
            clientSocket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
            try
            {
                clientSocket.Connect(new IPEndPoint(IPAddress.Parse("127.0.0.1"), 88));
                Console.WriteLine("连接服务器成功");
            }
            catch (Exception e)
            {
                Console.WriteLine(e);
                Console.WriteLine("连接服务器失败");
            }
            BeginSendMessagesToServer("Hello,服务器端。");
            BeginReceiveMessages();
        }
 
        /// <summary>
        /// 开始发送数据到客户端
        /// </summary>
        /// <param name="msg">要传递的数据</param>
        static void BeginSendMessagesToServer(string msg)
        {
            try
            {
                clientSocket.Send(Message.GetBytes(msg));
                Console.WriteLine("{0} 发送成功!",msg);
            }
            catch(Exception e)
            {
                Console.WriteLine(e);
            }
        }
 
        /// <summary>
        /// 开始接收来自客户端的数据
        /// </summary>
        /// <param name="toClientsocket"></param>
        static void BeginReceiveMessages()
        {
            clientSocket.BeginReceive(rec_Message.Data, rec_Message.StartIndex, rec_Message.RemindSize, SocketFlags.None, ReceiveCallBack, null);
        }
 
        /// <summary>
        /// 接收到来自服务端消息的回调函数
        /// </summary>
        /// <param name="ar"></param>
        static void ReceiveCallBack(IAsyncResult ar)
        {
            try
            {
                int count = clientSocket.EndReceive(ar);
                Console.WriteLine("从客户端接收到数据,解析中。。。");
                rec_Message.AddCount(count);
                //打印来自客户端的消息
                rec_Message.ReadMessage();
 
                //继续监听来自服务端的消息
                clientSocket.BeginReceive(rec_Message.Data, rec_Message.StartIndex, rec_Message.RemindSize, SocketFlags.None, ReceiveCallBack, null);
            }
            catch (Exception e)
            {
                Console.WriteLine(e);
            }
 
        }
    }
}

```
