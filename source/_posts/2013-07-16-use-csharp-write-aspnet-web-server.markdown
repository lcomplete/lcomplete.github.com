---
layout: post
title: "使用 C# 编写简易 ASP.NET Web 服务器"
date: 2013-07-16 05:44
comments: true
categories: C#
---

你是否有过这样的需求——想运行 ASP.NET 程序，又不想安装 IIS 或者 Visual Studio？我想如果你经常编写 ASP.NET 程序的话，应该或多或少都会碰到这种情况。除了使用 IIS 和 VS，我们还有哪些方式可以运行 ASP.NET 程序呢，自己写一个支持 ASP.NET 的 Web 服务器怎么样？NO NO NO，如果你只是想找个这样的工具的话，那完全没必要，我们知道使用 VS 可以运行 ASP.NET 程序，那么我们就可以找出 VS 所调用的程序，将其拷贝到没有 VS 和 IIS 的环境中运行，就能运行 ASP.NET 程序了，安装了 VS 的朋友可以到 C:\Program Files\Common Files\Microsoft Shared\DevServer\ 这个目录里面找找看，这个程序的使用方式如下。
```
WebDev.WebServer.EXE /port:80 /path:"c:\mysite" /vpath:"/"
```
怎么样？不错吧，轻而易举地就解决了文章开头所说的问题了。当然这并不是本篇文章的重点，如果你不满足于只知道这个用法，那可以继续往下阅读，接下来，我们将使用 C# 编写一个支持 ASP.NET 的 Web 服务器，看看这一切究竟是如何运作的。

C# 中有着许多丰富的类库，使用不同的类库，我们可以站在不同的抽象层级去编写一个 Web 服务器，比如在 System.Net 命名空间下提供了一个 HttpListener 类，使用这个类，我们可以很容易地创建一个简单的 Web 服务器，但是这个类隐藏了很多实现的细节，为了避免知其然不知其所以然，我们将使用网络框架最底层的 Socket 类来编写这个程序。

## 预备知识
正式编写这个程序之前，让我们先来了解一些基础知识。编写一个 Web Server，必需要了解 HTTP 协议，它是万维网的基础，位于 TCP/IP 协议栈的应用层。

1. HTTP 协议

HTTP 协议是一个基于请求与响应模式、无状态的应用层协议，HTTP 请求主要包括三部分：请求行、请求报头、请求正文，下面是一个请求示例。
```
GET /lcomplete/AspNetServer HTTP/1.1 

Host: github.com
Connection: keep-alive
Cache-Control: max-age=0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/28.0.1500.72 Safari/537.36

postdata #可选的消息体
```
第一行是请求行，该行又分为3个部分，分别是动作、URI 和 HTTP 协议版本，后面的 {key}: {value} 格式的行为报头，如果请求为 post 动作的话，则报头后面的post数据为请求正文，需要注意请求行、报头、请求正文之间必需以<CR><LF>（回车+换行）分割。

Web 服务器接收到一个请求后，就会将请求解析成上面3个部分，并开始处理应答，响应也由3个部分组成：状态行、响应报头、响应正文，这3部分同样使用<CR><LF>进行分割，状态行为HTTP协议版本、状态码、状态描述组成，响应报头与请求报头格式相同，只不过请求报头由服务器解释并处理，响应报头由浏览器解释并处理，最后的响应正文便是我们所熟悉的 HTML。

了解了 HTTP 协议的基础知识后，我们可以很容易地构建出一个支持静态文件的 HTTP 服务器，但是如何处理 ASP.NET 动态内容呢，这就要求我们熟悉 ASP.NET 的 HTTP 架构、管道机制、应用程序生命周期和宿主环境。

2. ASP.NET 运行时机制

ASP.NET 被特意设计成避免依赖 IIS，它的底层架构采用了管道机制，管道由一系列处理 HTTP 消息的对象组成，每个 HTTP 请求都要经过这些对象，每个对象都执行一些自己职责之内的任务。

HttpRuntime 类是管道的入口，它负责开始处理请求，管理首先执行 HttpRuntime 类上的静态方法 ProcessRequest ，这个方法接收一个 HttpWorkerRequest 对象参数，该对象包含了当前请求的相关信息，HttpRuntime 类使用这个请求信息构建 HttpContext 对象，其中包含了 HttpRequest 和 HttpResponse 属性，然后根据上下文获取 HttpApplication 对象，之后请求交给 HttpApplication 对象进行处理。

HttpApplication 对象中维护了 IHttpModule 模块对象列表，它可以在页面对象执行前后进行一些额外的工作，TODO。

下面就开始编码实战吧。

## 编码实战
还记得文章开头的命令吗？运行一个网站需要提供3个必要的东西，端口、网站物理路径、网站虚拟路径，在程序开始运行时需要得到这3个参数。
``` csharp
static void Main(string[] args)
{
    int port;
    string dir = Directory.GetCurrentDirectory();
    if(args.Length==0 || !int.TryParse(args[0],out port))
    {
        port = 45758; //端口
    }

    InitHostFile(dir);
    SimpleHost host= (SimpleHost) ApplicationHost.CreateApplicationHost(typeof (SimpleHost), "/", dir);
    host.Config("/", dir); //配置虚拟路径和物理路径

    WebServer server = new WebServer(host, port);
    server.Start();
}
```
为了便于测试，我将这3个参数都写死了，端口默认使用45758，物理路径使用程序所在目录，虚拟路径使用根目录，这两个路径信息保存在 host 对象中，通过 host 和 port 初始化 WebServer 对象，调用 server 的 Start 方法启动服务器。
``` csharp
public void Start()
{
    _serverSocket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
    _serverSocket.ExclusiveAddressUse = true;
    _serverSocket.Bind(new IPEndPoint(IPAddress.Any, Port));
    _serverSocket.Listen(1000);
    IsRuning = true;

    Console.WriteLine("Serving HTTP on 0.0.0.0 port " + Port + " ...");

    new Thread(OnStart).Start();
}

private void OnStart(object state)
{
    while (IsRuning)
    {
        try
        {
            Socket socket = _serverSocket.Accept();
            ThreadPool.QueueUserWorkItem(AcceptSocket, socket);
        }
        catch (Exception ex)
        {
            Console.WriteLine(ex);
            Thread.Sleep(100);
        }
    }
}

private void AcceptSocket(object state)
{
    if (IsRuning)
    {
        Socket socket = state as Socket;
        HttpProcessor processor = new HttpProcessor(_host, socket);
        processor.ProcessRequest();
    }
}
```
在 Start 方法中，创建了一个全局的 socket 对象，使其监听指定端口，并新开了一个线程用于处理客户端请求，当接收到客户端请求后，将其交给 HttpProcessor 对象处理。
``` csharp
public void ProcessRequest()
{
    try
    {
        RequestInfo requestInfo = ParseRequest();
        if (requestInfo != null)
        {
            string staticContentType = GetStaticContentType(requestInfo);
            if (!string.IsNullOrEmpty(staticContentType))
            {
                WriteFileResponse(requestInfo.FilePath, staticContentType);
            }
            else if (requestInfo.FilePath.EndsWith("/"))
            {
                WriteDirResponse(requestInfo.FilePath);
            }
            else
            {
                _host.ProcessRequest(this, requestInfo);
            }
        }
        else
        {
            SendErrorResponse(400);
        }
    }
    finally
    {
        Close();//确保连接关闭
    }
}
```
处理的步骤如下：

1. 解析请求数据，从建立的 socket 连接处获取请求数据，将其解析为RequestInfo对象。
2. 判断请求是否有效，无效则响应 400 错误，有效则进行下一步处理。
3. 判断请求的是否为静态内容，是则输出文件响应。
4. 判断请求是否为目录，是则输出目录下的子文件夹和文件的链接，与 IIS 目录服务类似。
5. 不为静态内容和目录时，则交给 host 对象处理（使用ASP.NET HTTP 运行时进行处理）。
6. 处理完后确保连接关闭。

其中输出响应是构造状态行、响应报头和响应正文，接着通过 socket 发送给客户端的过程。相信看到这里，大家已经对整个交互过程有了一个了解，剩下的最后一个问题就是如何处理动态内容。


