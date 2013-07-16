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

1.  HTTP 协议

    HTTP 协议是一个基于请求与响应模式、无状态的应用层协议，HTTP 请求主要包括三部分：请求行、请求报头、请求正文，下面是一个请求示例。

        GET /lcomplete/AspNetServer HTTP/1.1 
        Host: github.com
        Connection: keep-alive
        Cache-Control: max-age=0
        Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
        User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/28.0.1500.72 Safari/537.36
        
        postdata #可选的消息体

    第一行是请求行，该行又分为3个部分，分别是动作、URI 和 HTTP 协议版本，后面的 {key}: {value} 格式的行为报头，如果请求为 post 动作的话，则报头后面的post数据为请求正文，需要注意报头和请求正文之间必需以<CR><LF>（回车+换行）分割。
        
    Web 服务器接收到一个请求后，就会将请求解析成上面3个部分，并开始处理应答，响应也由3个部分组成：状态行、响应报头、响应正文，响应报头和正文同样使用<CR><LF>进行分割，状态行为HTTP协议版本、状态码、状态描述组成，响应报头与请求报头格式相同，只不过请求报头由服务器解释并处理，响应报头由浏览器解释并处理，最后的响应正文便是我们所熟悉的 HTML。
        
    了解了 HTTP 协议的基础知识后，我们可以很容易地构建出一个支持静态文件的 HTTP 服务器，但是如何处理 ASP.NET 动态内容呢，这就要求我们熟悉 ASP.NET 的 HTTP 架构、管道机制、应用程序生命周期和宿主环境。
    
2. ASP.NET 运行时机制

    ASP.NET 被特意设计成避免依赖 IIS，它的底层架构采用了管道机制，管道由一系列处理 HTTP 消息的对象组成，每个 HTTP 请求都要经过这些对象，每个对象都执行一些自己职责之内的任务。
    
    HttpRuntime 类是管道的入口，它负责开始处理请求，管理首先执行 HttpRuntime 类上的静态方法 ProcessRequest ，这个方法接收一个 HttpWorkerRequest 对象参数，该对象包含了当前请求的相关信息，HttpRuntime 类使用这个请求信息构建 HttpContext 对象，其中包含了 HttpRequest 和 HttpResponse 属性，然后根据上下文获取 HttpApplication 对象，之后请求交给 HttpApplication 对象进行处理。
    
    处理请求时，HttpApplication 会执行一系列任务，其中包括为请求调用合适的 IHttpHandler 类的 ProcessRequest 方法，例如，如果请求针对某页，则使用该页的实例处理该请求，另外 HttpApplication 中还维护了 IHttpModule 对象列表，它可以在页面实例处理请求前后进行一些额外的工作。
    
    管道机制是完全自主的，不需要依附于 IIS 上，不过管道并没有接收 HTTP 请求的能力，我们需要自己编写这部分代码，当收到请求时，创建 HttpWorkerRequest 对象并提供给 HttpRuntime.ProcessRequest 方法调用以启动管道。

    要处理 ASP.NET 请求，还需要创建一个应用程序域以托管 HTTP 管道，我们可以使用 ApplicationHost.CreateApplicationHost 方法创建应用程序域，该方法接收3个参数：宿主类型、虚拟路径和物理路径，宿主类型需要跨域应用程序边界，所以需要继承自 MarshalByRefObject 类，并提供与其交互的方法，例如至少要提供一个方法使得可以提交 ASP.NET 请求以进行处理。

    了解了 ASP.NET 的运行机制后，再来看看编写 ASP.NET 服务器需要使用到哪些类，首先我们需要使用 __ApplicationHost__ 创建应用程序域以获得处理 ASP.NET 请求的能力，接收到请求后构造 __HttpWorkerRequest__ （该类是抽象类，需要定义它的子类）对象，交由 __HttpRuntime__ 类进行处理，接下来的事情就由 HTTP 管道处理了。

    好了，预备知识已经讲解完毕，下面让我们进入编码实战。

## 编码实战

还记得文章开头的命令吗？运行一个网站需要提供3个必要的东西，端口、网站物理路径、网站虚拟路径，在程序开始运行时需要得到这3个参数。
``` csharp Program.cs
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

//需要拷贝执行文件 才能创建ASP.NET应用程序域
private static void InitHostFile(string dir)
{
    string path = Path.Combine(dir, "bin");
    if (!Directory.Exists(path))
        Directory.CreateDirectory(path);
    string source = Assembly.GetExecutingAssembly().Location;
    string target = path + "/" + Assembly.GetExecutingAssembly().GetName().Name + ".exe";
    if(File.Exists(target))
        File.Delete(target);
    File.Copy(source, target);
}
```
为了便于测试，我将这3个参数都写死了，端口默认使用45758，物理路径使用当前程序所在目录，虚拟路径使用根目录，这两个路径信息保存在 host 对象中。由于 Application.CreateApplicationHost 方法期望在 GAC 或指定的物理路径中的 bin 目录中找到宿主类型所在的程序集，所以在创建应用程序域之前先将当前程序拷贝到了物理路径的 bin 目录中，创建完应用程序域后初始化 WebServer 对象，调用该对象的 Start 方法以启动服务器。在 WebServer 中保留了 host 的引用，当处理 ASP.NET 请求时会使用到，我们先看一下启动服务器的方法。

``` csharp WebServer.cs
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
``` csharp HttpProcessor.cs
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

为了与 ASP.NET 的应用程序域交互，我们需要将请求信息提交给宿主对象 host 进行处理，下面是我们实现的宿主类。

``` csharp
public class SimpleHost : MarshalByRefObject
{
    public string PhysicalDir { get; private set; }

    public string VituralDir { get; private set; }

    public void Config(string vitrualDir, string physicalDir)
    {
        VituralDir = vitrualDir;
        PhysicalDir = physicalDir;
    }

    public void ProcessRequest(HttpProcessor processor, RequestInfo requestInfo)
    {
        WorkerRequest workerRequest = new WorkerRequest(this, processor, requestInfo);
        HttpRuntime.ProcessRequest(workerRequest);
    }
}
```

在 ProcessRequest 方法中，创建了 HttpWorkerRequest 的子类 WorkerRequest 对象，并提交给 HttpRuntime 进行处理。WorkerRequest 类中实现了 HttpWorkerRequest 中的抽象方法，其中包括 GetRawUrl 、GetHttpVerbName 等等这一类获取请求相关信息的方法，HTTP 管道调用这些方法以获取请求数据，同时它还包含类似 FlushResponse 这类输出响应的方法，HTTP 管道最终会调用这类方法向客户端发送数据，下面是 FlushResponse 方法的实现，在该方法中我们使用 HttpProcessor 对象向 socket 客户端发送响应数据。

``` csharp
public override void FlushResponse(bool finalFlush)
{
    if (!_isHeaderSent)
    {
        _processor.SendHeaders(_statusCode, _responseHeaders, -1, finalFlush);
        _isHeaderSent = true;
    }

    for (int i = 0; i < _responseBodyBytes.Count; i++)
    {
        byte[] data = _responseBodyBytes[i];
        _processor.SendResponse(data);
    }

    _responseBodyBytes = new List<byte[]>();
    if (finalFlush)
        _processor.Close();
}
```
 
到这一步，我们已经可以运行 ASP.NET 程序了，但是只实现抽象方法还不能提供足够的信息给 HTTP 管道，例如 HTTP 管道无法得知 POST 数据和 Cookie 数据，要提供这些信息我们还需要重写一些虚拟方法，如 GetKnownRequestHeader 、GetPreloadedEntityBody 等等，实现一些必要的方法之后，ASP.NET 程序就能够良好地运行了。

## 总结

编写支持 ASP.NET 的 Web 服务器，并不是一件难事，这得益于 ASP.NET 优雅的设计，只要向运行时提供必要的信息，HTTP 管道就能够正确地进行处理。

文中只贴了一小部分代码，你可以通过 [https://github.com/lcomplete/AspNetServer](https://github.com/lcomplete/AspNetServer) 该地址查看所有代码。
