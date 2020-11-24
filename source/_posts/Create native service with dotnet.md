---
title: 基于netcore 编写本地服务
categories: 
    - .NET
tags: 
    - C#
    - Service
date: 2020年5月27日
cover: https://image.zsver.com/2020/05/23/ff71252584474.jpg
---

## 前言

服务程序，一般是指一些需要长期稳定运行，没有界面交互的程序，今天就来看一看怎么用C#在 netcore 平台来编写这类程序。

## 环境

以下是编写服务程序使用的工具和环境

- VisualStduio 2019
- .netcore 3.1

## 开始

### 1. 使用Woker Service 模板创建服务

启动`VisualStudio 2019`，选择`创建新项目`,在上方的搜索栏搜索`service` ,如下图。

![创建Worker Service](https://image.zsver.com/2020/06/01/45ed7d8f41ad2.png)

或者使用 `dotnet new worker` 命令

这里我创建一个名为`MyService`的项目

```bash
dotnet new worker -n MyService
```

浏览一下项目文件，可以看见里面有两个主要的文件

- ####  1. Program.cs

```csharp
  public static void Main(string[] args)
  {
      CreateHostBuilder(args).Build().Run();
  }

  public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureServices((hostContext, services) =>
                {
                    services.AddHostedService<Worker>();
                });
```

- ####  2. Worker.cs

```csharp
 public class Worker : BackgroundService
 {
    private readonly ILogger<Worker> _logger;

    public Worker(ILogger<Worker> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Worker running at: {time}", DateTimeOffset.Now);
            await Task.Delay(1000, stoppingToken);
        }
    }
 }
```

这两个文件中的代码都是简洁明了的。
`Program.cs` 创建`Host`并对Worker进行托管。  

那么我们先看一下Worker是什么?  
 &emsp; 从`Worker.cs`中可见Worker只是继承自`BackgroundService`的子类。  
`BackgroundService`又是何方神圣呢？  
  &emsp;看一下`MyService`的项目依赖，依赖包里有`Microsoft.Extensions.Hosting`这一项，`BackgroundService` 就是源自于此。

### 2. 编写属于自己的Worker

新建 `MyWorker.cs` 文件，内容如下

```csharp
 class MyWorker : BackgroundService
    {
        private readonly string[] _lines =
        {
            "沉舟侧畔千帆过，",
            "病树前头万木春。",
            "-------------",
            "长风破浪会有时，",
            "直挂云帆济沧海。",
            "-------------",
            "谁无暴风劲雨时，",
            "守得云开见月明。",
            "-------------"
        };


        protected async override Task ExecuteAsync(CancellationToken stoppingToken)
        {
            while(!stoppingToken.IsCancellationRequested)
            {

                foreach (var line in _lines)
                {
                    await Task.Delay(1000);

                    foreach (var ch in line)
                    {
                        Console.Write(ch);
                        await Task.Delay(100);
                    }

                    Console.Write("\r");
                }


            }
        }
    }
```

除此之外，还需要将Worker 注入。
`services.AddHostedService<MyWorker>();`
注释掉 `services.AddHostedService<Worker>();`,而只关注`MyWorker`

运行之后，就可以看到内容输出了。但是，为什么还会有控制台窗口，而不是一个无界面的服务呢？
![运行情况](https://image.zsver.com/2020/06/01/1a5bde9b72a3e.gif)
嗯mm，因为现在它还是一个控制台程序。  
那么，如和部署成服务呢？  

### 3. 部署Windows 服务

首先，要做一些改动  

1. 为`MyService`项目添加nuget包 `Microsoft.Extensions.Hosting.WindowsServices`

2. 修改 `Program.cs` 中的 `CreateHostBuilder` 函数，在最后加上 `UseWindowsService()`  
如下  

```csharp
  static IHostBuilder CreateHostBuilder(string[] args)
            => Host.CreateDefaultBuilder(args).ConfigureServices(services =>
            {
                services.AddHostedService<MyWorker>();
            }).UseWindowsService();
```

接着  
这里需要使用另一个工具 `sc`  
打开cmd窗口（使用Powershell 也可以，但是低版本的Powershell 可能会出错）

```powershell
sc create MyService binpath=E:\full\path\to\yourbinaryfile.exe
# binpath 一定是绝对路径，否则运行服务会提示系统找不到指定文件

# 另外
sc start MyService
sc stop MyService
sc delete MyService

```

程序在控制台输出，所以安装成服务并不能看到效果。不过，总而言之，使用.netcore创建服务程序很是简单。你可能会想用这种方式用在.netframework上是不是也可以。可以自己尝试。  

---

## 更新

2020年7月13日 补充
在.netframework 上使用也是没问题的，但是.netcore 可以跨平台食用
  
&emsp;*更多创建服务的方式，可以点击下面的引用链接*  

## 引用

[Creating Windows Services In .NET Core – Part 3 – The “.NET Core Worker” Way](https://dotnetcoretutorials.com/2019/12/07/creating-windows-services-in-net-core-part-3-the-net-core-worker-way/)
