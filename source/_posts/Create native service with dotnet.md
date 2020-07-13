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


服务程序，一般代表一些需要长期稳定运行，没有界面交互的程序，周而复始，生生不息。那么,怎么用.netcore 来编织这么一个程序呢？

## 利器

以下是在下所使用的工具

- VisualStduio 2019
- .netcore 3.1

## 起始

### 道生一 | 使用Woker Service 模板创建服务

启动`VisualStudio 2019`，选择`创建新项目`,在上方的搜索栏搜索`service` ,如下图。

![创建Worker Service](https://image.zsver.com/2020/06/01/45ed7d8f41ad2.png)

或者使用`dotnet new worker`

这里在下创建一个名为`MyService`的项目

```bash
dotnet new worker -n MyService
```

窥探一番项目文件，可见里面有两个主要的文件

- #### Program.cs

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

- #### Worker.cs

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

这两个文件都是相当简洁明了的。
`Program.cs` 创建`Host`并对Worker进行托管。  
那么Worker是什么?  
 &emsp; 从`Worker.cs`中可见Worker只是继承自`BackgroundService`的子类。  
`BackgroundService`又是何方神圣呢？  
  &emsp;窥一眼`MyService`的项目依赖，依赖包里有`Microsoft.Extensions.Hosting`这一项，`BackgroundService`正是来自于此。  
这么一看，不过如此，如果阁下不使用VS 模板来创建自己的服务，想必也是易如反掌。

### 一生二 | 编写属于自己的Worker

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

运行之后，就可以看到内容输出了。这点很重要，为什么还会有控制台窗口，而不是一个无界面的服务？
![运行情况](https://image.zsver.com/2020/06/01/1a5bde9b72a3e.gif)
嗯，现在它还是一个控制台程序。  
那么，如和部署成服务呢？  

### 三生万物 | 部署Windows 服务

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
这里需要使用另一把武器 `sc`  
打开cmd窗口（使用Powershell 也可以，但是低版本的Powershell 可能会出错）

```powershell
sc create MyService binpath=E:\full\path\to\yourbinaryfile.exe
# binpath 一定是绝对路径，否则运行服务会提示系统找不到指定文件

# 另外
sc start MyService
sc stop MyService
sc delete MyService

```

程序在控制台输出，所以安装成服务并不能看到效果。谁管呢，总而言之，使用.netcore创建服务程序很简单，这种方式用在.netframework上又当如何呢？阁下可自行尝试，此处不再多言。  

2020年7月13日 补充
在.netframework 上使用也是没问题的，但是.netcore 可以跨平台食用
  
&emsp;*更多创建服务的方式，可以点击下面的引用链接*

## 终

### Reference

[Creating Windows Services In .NET Core – Part 3 – The “.NET Core Worker” Way](https://dotnetcoretutorials.com/2019/12/07/creating-windows-services-in-net-core-part-3-the-net-core-worker-way/)
