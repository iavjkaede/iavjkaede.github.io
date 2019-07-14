---
title: ASP.NET 之 IIS托管的后台任务
date: 2019-6-1 12:39
categories: 
	- Web
tags: 
	- IIS
	- Background Task
	- C#
cover: https://cloud.zzserver.top/s/8Sew3gzyter7fDj/preview
---



# ASP.NET 之 IIS托管的后台任务

## 前言

&emsp;&emsp;可能有这样一个需求，我们希望网站后台每隔一段时间自动执行某些任务。然而在网站部署到IIS上的情况下，由于IIS应用程序池自动回收的原因，导致任务中断，无法一直保持后台运行。

​		现在，我们来找出一个解决方法。嗯。

## 环境简介

这里是我在做这件事情时的一个部署和开发所使用的环境。

### 系统环境

Windows 10 1903

### 开发环境

工具：Visual Studio 2019

SDK：.Netframework 4.6.2 

### 部署环境

IIS Express



## 实现

### 利用定时器实现后台任务

**JobManager.cs**

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

**这样实现的任务有个致命缺陷，IIS 应用程序池自动回收的时候会毫不留情的将其摧毁。**

下面有两种异曲同工的解决方式。

---

###  Application_End 中唤醒 IIS

​	应用程序池进行回收的时候会调用*Application_End* 方法，所以，在这里继续将IIS唤醒即可。

​	我把关键的代码放在这里

**Global.asax.cs**

```csharp
protected async void Application_End(object sender,EventArgs e)
{
    // 从这里唤醒你的网站，所以我们需要得到网站的地址
    string url = "http://www.yousite.com";
    try{
          using(HttpClient client = new HttpClient())
          {
			 var response = await client.GetAsync(url);
          }
    }catch(Exception)
    {
        // 处理可能出现的异常
    }
}
```

可能还需要写个日志来记录情况或者用*HttpWebRequest*来发起请求，都没问题。

我们来看下一种方式。

---

###  IRegisteredObject 唤醒IIS

####  IRegisteredObject 接口

```csharp
public interface IRegisteredObject
{
    void Stop(bool immediate);
}
```

​	它是在`System.Web.Hosting` 下的一个接口，它是做什么用的呢？

搬运一下MSDN的描述 ：

---

*可以通过调用创建已注册对象的实例[ApplicationManager.CreateObject](https://docs.microsoft.com/zh-cn/dotnet/api/system.web.hosting.applicationmanager.createobject?view=netframework-4.8)的应用程序管理器中的方法。 应用程序管理器将新创建的对象返回给调用方，然后在对象调用特定于类型的方法。 在启动期间，已注册的对象应调用[HostingEnvironment.RegisterObject](https://docs.microsoft.com/zh-cn/dotnet/api/system.web.hosting.hostingenvironment.registerobject?view=netframework-4.8)方法以完成注册的对象。*

*当应用程序管理器需要停止已注册的对象时，它将调用[Stop](https://docs.microsoft.com/zh-cn/dotnet/api/system.web.hosting.iregisteredobject.stop?view=netframework-4.8)方法*

是的，很抽象，反正这个描述我是看不懂。

简单说一下，就是**应用程序池将要回收的时候，会调用Stop方法**，那么我们可以在Stop中实现上面同样的逻辑。

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



在使用JobHost 之前需要调用

`HostingEnvironment.RegisterObject(this);` 进行注册

在Stop 被调用时  要调用

`HostingEnvironment.UnregisterObject(this); ` 进行反注册

---

### 完整代码

通过第二种解决方式实现后台任务的完整代码

##### JobHost.cs

```csharp
class JobHost : IRegisteredObject
{
    private readonly object m_lock = new object ();
    private bool m_shuttingDown;

    public JobHost ()
    {
        HostingEnvironment.RegisterObject (this);
    }

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

    public void DoWork (Action work)
    {
        lock (m_lock)
        {
            if (m_shuttingDown)
                return;
            work ();
        }
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
            // 处理可能出现的异常
        }
    }
    
}
```





我这里讲的不够详细，如果要参考更多细节，这里贴出参考的blog

[The Dangers of Implementing Recurring Background Tasks In ASP.NET](https://haacked.com/archive/2011/10/16/the-dangers-of-implementing-recurring-background-tasks-in-asp-net.aspx/) 



---

## 最后

​	这种方式的原理很简单，IIS就像个贪睡的孩子，当他快要打瞌睡的时候，你就拍拍他的脑袋。

   “ 嘿，好孩子，该工作了。”

   没错，就是这么残酷😂。



**当我们需要持续稳定的执行后台任务的时候，更好的方式应该是写一个服务程序，或者控制台**



## 参考

[IIS 自动回收导致后台定时器失效的问题解决](https://www.2cto.com/kf/201112/115331.html)

[The Dangers of Implementing Recurring Background Tasks In ASP.NET](https://haacked.com/archive/2011/10/16/the-dangers-of-implementing-recurring-background-tasks-in-asp-net.aspx/)

[Timer Class](https://docs.microsoft.com/zh-cn/dotnet/api/system.threading.timer?f1url=https%3A%2F%2Fmsdn.microsoft.com%2Fquery%2Fdev15.query%3FappId%3DDev15IDEF1%26l%3DZH-CN%26k%3Dk(System.Threading.Timer);k(TargetFrameworkMoniker-.NETFramework,Version%3Dv4.6.2);k(DevLang-csharp)%26rd%3Dtrue&view=netframework-4.8)