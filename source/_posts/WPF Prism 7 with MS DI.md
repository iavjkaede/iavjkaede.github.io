---
title: WPF Prism 7 中使用基于MS DI 的Services
date: 2020年7月13日
categories: 
  - .NET
tags: 
  - C#
  - Prism
  - WPF
  - IOC
cover: https://image.zsver.com/2020/07/13/770f2166bbe89.jpg
---

## 前言

Prism 是基于WPF 界面技术的MVVM应用框架，框架里集成了IOC容器，默认可以选择的是Unity和DryIoc，出于某种原因，我这里想使用 .netcore 原生IOC容器，下面就讲一下怎么做的，还挺波折的。

工作环境如下  

- VisualStudio 2019  
- .netcore 3.1.301  
- Prism 7.2.0.1422（DryIoC）

## 1. Prism.Container.Extensions

我先在nuget中查了下和**Prism，IOC**相关的包，发现`Prism.Container.Extensions` 可能就是解决相关问题的，不错，
先装上它。  
版本*7.2.0.1054*  

安装之后，`IContainerRegistry`类下面多了一个扩展方法`RegisterServices`。  
小试一手

```csharp
 protected override void RegisterTypes(IContainerRegistry containerRegistry)
{
    containerRegistry.RegisterServices(service =>
    {
        service.AddLogging();
    });

}
```

看上面的代码，可以用`ServiceCollection`来进行注入。那么其他的诸如 AddOptions,AddDbContext 操作是不是易如反掌呢，看样子倒是异常容易。

不料运行就抛出异常了。  

`System.NotImplementedException:“The method or operation is not implemented.”`

`System.NotImplementedException:“The method or operation is not implemented.”`

`System.NotImplementedException:“The method or operation is not implemented.”`

我下载了项目的源码然后跟踪了一下错误，大致问题就是在扩展方法`RegisterServices`中注入Scope生命周期的服务时便会出错，尚无对应实现，也就是当前Prism里不支持注入Scope生命周期的依赖。

## 2. Prism.Microsoft.DependencyInjection

上面的方式行不通，为什么不读一下文档呢？  
于是乎打开 Prism.Container.Extensions 的 [github](https://github.com/dansiegel/Prism.Container.Extensions)主页，
翻阅Prism 的[文档](https://prismplugins.com/)。示例给出了这样的用法。😥

```csharp  

PrismContainerExtension.Current.RegisterServices(s => {
    s.AddHttpClient();
    s.AddDbContext<AwesomeAppDbContext>(o => o.UseSqlite("my connection string"));
});

```

我马上依照上面的用法做了尝试，关键代码如下。

```csharp

PrismContainerExtension.Current.RegisterServices(services =>
{
    services.AddLogging();

});

```

**！需要安装 Prism.Microsoft.DependencyInjection**  
*此处使用的版本 7.0.2.1054*  

这里注入是没有问题了，但是在ViewModel 的构造里面却无法正确解析`ILogger`，导致IOC抛出异常，无法构造ViewModel。具体原因不详，要获取`ILogger`的实例，只能通过`PrismContainerExtension.Current。Resolve<ILogger<MainWindowViewModel>>();` 这个是可以获取到的，但这不是我想要的结果，我期望能在构造函数中成功解析依赖，这显然也行不通。

## 3. DryIoc.Microsoft.DependencyInjection

于是开始查阅项目的issue  

[PrismApplicationBase.Container having difficulty resolving ILogger\<T>](https://github.com/dansiegel/Prism.Container.Extensions/issues/100)

[Prism.Forms dependency injection IServiceCollection problem](https://github.com/PrismLibrary/Prism/issues/1352)

[Cannot register grpc client using ms ioc and dryioc](https://github.com/dadhi/DryIoc/issues/287)

这几个issue 帮助极大，我这里就直接总结一下方法。  
此处不再依赖 `Prism.Container.Extensions`。
而是从 DryIoc 下手，只要能在DryIoc  中注入ServiceCollection 就大功告成了。  

1. 安装 `DryIoc.Microsoft.DependencyInjection`  
2. 使用重载方法 `CreateContainerExtension`完成注入

代码如下

```csharp

protected override IContainerExtension CreateContainerExtension()
{
    var containerExtension = base.CreateContainerExtension() as DryIocContainerExtension;

    ServiceCollection services = new ServiceCollection();
    services.AddLogging();
    containerExtension.Instance.Populate(services);

    return containerExtension;
}

```

这样便达成了目标，贴下`App.xaml.cs` 完整代码

```csharp

using Prism.Ioc;
using BlankCoreApp1.Views;
using System.Windows;
using Microsoft.Extensions.DependencyInjection;
using Prism.DryIoc.Ioc;
using DryIoc.Microsoft.DependencyInjection;

namespace BlankCoreApp1
{
    /// <summary>
    /// Interaction logic for App.xaml
    /// </summary>
    public partial class App
    {
        protected override Window CreateShell()
        {
            return Container.Resolve<MainWindow>();
        }

        protected override IContainerExtension CreateContainerExtension()
        {
            var containerExtension = base.CreateContainerExtension() as DryIocContainerExtension;

            ServiceCollection services = new ServiceCollection();
            services.AddLogging();
            containerExtension.Instance.Populate(services);

            return containerExtension;
        }

        protected override void RegisterTypes(IContainerRegistry containerRegistry)
        {
        }
    }
}


```

## 终

Prism 8 可能会在这个问题上有所改变。  
这个问题上浪费了一些时间，此处小记。

## Reference

[Cannot register grpc client using ms ioc and dryioc · Issue #287 · dadhi/DryIoc](https://github.com/dadhi/DryIoc/issues/287)

[Prism.Forms dependency injection IServiceCollection problem · Issue #1352 · PrismLibrary/Prism](https://github.com/PrismLibrary/Prism/issues/1352)

[dansiegel/Prism.Container.Extensions: The packages here provide additional extensions around the Prism Ioc abstractions. This allows for more advanced scenarios.](https://github.com/dansiegel/Prism.Container.Extensions)

[PrismApplicationBase.Container having difficulty resolving ILogger\<T> · Issue #100 · dansiegel/Prism.Container.Extensions](https://github.com/dansiegel/Prism.Container.Extensions/issues/100)
