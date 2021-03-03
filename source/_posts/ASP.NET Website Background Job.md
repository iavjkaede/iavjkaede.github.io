---
title: ASP.NET 之 IIS托管的后台任务
date: 2019-6-1 12:39
categories: 
	- .NET
tags: 
	- IIS
	- Background Task
	- C#
cover: https://image.zsver.com/2020/05/23/5f1cb7c259843.jpg
---

## 前言

&emsp;&emsp;在 ASP.NET 网站开发中，可能存在这样一个需求，我们希望网站后台能够每隔一段时间自动地执行某些任务，用定时器之类技术实现这个功能并不困难，但是在网站部署到IIS上的情况下，由于IIS应用程序池自动回收的原因，导致任务在IIS回收后中断，无法持续后台运行。

​现在，我们就解决这个问题。

## 环境简介

这里是我当前的开发和部署所使用的环境。

### 系统环境

Windows 10 1903

### 开发环境

工具：Visual Studio 2019

SDK：.Netframework 4.6.2

### 部署环境

IIS Express

## 实现

### 先看看如何利用定时器实现后台任务

`JobManager.cs`

```csharp
using System;

public class JobManager
{
    private static Timer m_timer;
    static JobManager()
    {
         m_timer = new Timer(new TimerCallback(TimerCallback));
    }

    private static void TimerCallback(object target)
    {
        // do work here
    }

    public static void Start()
    {
        // interval 1 min
        m_timer.Change(TimeSpan.Zero, TimeSpan.FromMinutes(1));
    }
}
```

使用

`JobManager.Start();`

有关Timer的用法 可参考MSDN

[Timer Class](https://docs.microsoft.com/zh-cn/dotnet/api/system.threading.timer?f1url=https%3A%2F%2Fmsdn.microsoft.com%2Fquery%2Fdev15.query%3FappId%3DDev15IDEF1%26l%3DZH-CN%26k%3Dk(System.Threading.Timer);k(TargetFrameworkMoniker-.NETFramework,Version%3Dv4.6.2);k(DevLang-csharp)%26rd%3Dtrue&view=netframework-4.8)

**前言提到过，单单这样实现的任务并不能保证后台执行，IIS 应用程序池自动回收的时候会毫不留情的将其摧毁。**

下面有两种异曲同工的解决方式。

---

### 1.Application_End 中唤醒 IIS

​&emsp;&emsp;IIS的应用程序池进行回收的时候会调用*Application_End* 方法，有了这个条件，我们便可以在*Application_End*方法中做些文章了。
怎么做呢，我们只简单的访问一下自己的网就好了。  
IIS 想休息，因为觉得没人拜访了，那我们就登门拜访。

​关键代码如下所示

`Global.asax.cs`

```csharp
protected async void Application_End(object sender,EventArgs e)
{
    // 从这里唤醒你的网站，所以我们需要得到网站的地址
    // TODO: 网址不要使用硬编码表示
    string url = "http://www.yoursite.com";
    try{
          using(HttpClient client = new HttpClient())
          {
             var response = await client.GetAsync(url);
          }
    }catch(Exception)
    {
        // TODO: 处理可能出现的异常
    }
}
```

可能还需要写个日志来记录情况或者用*HttpWebRequest*来发起请求，都取决与聪明的你。

来看下一种方式。

---

### 2.IRegisteredObject 唤醒IIS

#### IRegisteredObject 接口

```csharp
public interface IRegisteredObject
{
    void Stop(bool immediate);
}
```

​它是在`System.Web.Hosting` 下的一个接口，它是做什么用的呢？

搬运一下MSDN的描述 ：

---

*可以通过调用创建已注册对象的实例[ApplicationManager.CreateObject](https://docs.microsoft.com/zh-cn/dotnet/api/system.web.hosting.applicationmanager.createobject?view=netframework-4.8)的应用程序管理器中的方法。 应用程序管理器将新创建的对象返回给调用方，然后在对象调用特定于类型的方法。 在启动期间，已注册的对象应调用[HostingEnvironment.RegisterObject](https://docs.microsoft.com/zh-cn/dotnet/api/system.web.hosting.hostingenvironment.registerobject?view=netframework-4.8)方法以完成注册的对象。*

*当应用程序管理器需要停止已注册的对象时，它将调用[Stop](https://docs.microsoft.com/zh-cn/dotnet/api/system.web.hosting.iregisteredobject.stop?view=netframework-4.8)方法*

简而言之，就是**应用程序池将要回收的时候，会调用Stop方法**，那么我们可以在*Stop*方法中实现与上面同样的逻辑。

---

#### IRegisteredObject 使用

首先要有一个类继承自 `IRegisteredObject`

```csharp
public class JobHost:IRegisteredObject
{
    public JobHost()
    {
        HostingEnvironment.RegisterObject(this);
    }
    public void Stop(bool immediate)
    {
        // 在这里唤醒IIS
        HostingEnvironment.UnregisterObject(this);
    }
}
```

在使用 *JobHost* 之前需要调用

`HostingEnvironment.RegisterObject(this);` 进行注册

在 *Stop* 方法被调用时调用

`HostingEnvironment.UnregisterObject(this);` 进行反注册

---

### 完整代码

通过第二种方式实现后台任务的完整代码

#### JobHost.cs

```csharp
class JobHost : IRegisteredObject
{
    private readonly object m_lock = new object ();
    private bool m_shuttingDown;

    public JobHost ()
    {
        // 注册对象
        HostingEnvironment.RegisterObject (this);
    }

    // IIS进行回收时调用
    public void Stop (bool immediate)
    {
        lock (m_lock)
        {
            m_shuttingDown = true;
        }
        HostingEnvironment.UnregisterObject (this);

        // 程序池被回收的时候重新唤醒iis
        JobManager.Wakeup ();
    }

}
```

##### JobManager.cs

```csharp
public class JobManager
{
    private static Timer m_timer;
    private static readonly JobHost m_jobHost = new JobHost();

    static JobManager()
    {
         m_timer = new Timer(new TimerCallback(TimerCallback));
    }

    private static void TimerCallback(object target)
    {
        m_jobHost.DoWork(()=>{
            // do work here
        });
    }

    public static void Start()
    {
        // interval 1 min
        m_timer.Change(TimeSpan.Zero, TimeSpan.FromMinutes(1));
    }

    public static void Wakeup()
    {
        string url = "www.yousite.com";
        try
        {
            using(HttpClient client = new HttpClient())
            {
               var response = await client.GetAsync(url);
            }

        }catch(Exception)
        {
            //TODO 处理可能出现的异常
        }
    }

}
```

我这里说的可能不太详尽，如果想要参考更多细节，这里贴出参考的blog

[The Dangers of Implementing Recurring Background Tasks In ASP.NET](https://haacked.com/archive/2011/10/16/the-dangers-of-implementing-recurring-background-tasks-in-asp-net.aspx/)

---

## 最后

​这个方法很简单，IIS总觉得当没人访问站点的时候，它就应该减少该站点的资源占用。
    抱歉
   “我不要你觉得，我要我觉得 。”

   没错，不要偷懒。
**当然，这是一个投机取巧的方式，用它来完成一些不那么重要和准确的任务是可以的，如果这些任务特别重要，应该考虑使用更稳定和适合的方式，比如，Windows 服务**

## 参考

[IIS 自动回收导致后台定时器失效的问题解决](https://www.2cto.com/kf/201112/115331.html)

[The Dangers of Implementing Recurring Background Tasks In ASP.NET](https://haacked.com/archive/2011/10/16/the-dangers-of-implementing-recurring-background-tasks-in-asp-net.aspx/)

[Timer Class](https://docs.microsoft.com/zh-cn/dotnet/api/system.threading.timer?f1url=https%3A%2F%2Fmsdn.microsoft.com%2Fquery%2Fdev15.query%3FappId%3DDev15IDEF1%26l%3DZH-CN%26k%3Dk(System.Threading.Timer);k(TargetFrameworkMoniker-.NETFramework,Version%3Dv4.6.2);k(DevLang-csharp)%26rd%3Dtrue&view=netframework-4.8)
