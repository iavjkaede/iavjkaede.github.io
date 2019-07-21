---
title: .NetCore Web 项目的插件实现
date: 
categories:
	- Web
tags: 
	- .NetCore
	- C#
	- Plugins
cover: https://cloud.zzserver.top/s/ojFWrYriHidHmwH/preview
---

## 前言

为什么使用插件的方式去完成项目？并不是因为这很酷，仅仅是，我们希望项目变得更容易维护，更容易拓展，彼此之间的依赖更清晰更简单。使用插件，我们可以在不影响现有功能情况下去开发新的功能，随意替换现有的功能模块，连重新编译都不需要。看起来确实挺酷，这么酷的功能，在C#中实现起来却并不复杂。嗯，那就分享一下我在.NetCore 项目中使用插件的方式。

## 环境简介

### 系统环境

Windows 10 LTSC 2019

### 开发环境

SDK： .NetCore 2.2

Database：SQLite 3

### 开发工具

Editor：Visual Studio Code

Shell：Powershell

## 准备

### 项目准备

使用以下命令去创建以下项目。

```powershell
mkdir PluginDemo
cd PluginDemo

# 保证使用的是2.2版本的.netcore sdk
# 使用 dotnet --list-sdks 查看已安装的.netcore 版本
dotnet new global.json --sdk-version 2.2.204

# 创建项目
dotnet new web -o Host
dotnet new classlib -o Share
dotnet new classlib -o Plugin1

# 添加引用
dotnet add .\Host\ reference .\Share\
dotnet add .\Plugin1\ reference .\Share\

# 在 VS Code 中打开
code ./
```

按下`F5`进行调试，保证项目成功启动。

这里创建了三个项目，分别是**Host**、**Plugin1**、**Share**，依赖关系如下

Host <- Share

Plugin1 <- Share

### 数据库准备

在真正开始之前，最好部署一个可用的数据库，因为后面的Model映射会用到。

## 开始

准备好了吗？准备好那可就开始了。🚀

### 加载Controller

首先要做到的一点就是，把编写的程序集（其中包含继承自Controller的子类）丢到定义的插件目录，重启应用后，程序集中的Controller能够被成功路由。

废话不多说，用**ApplicationPart**来实现这一点。

#### 修改Startup.cs*

项目 -- **Host**

编辑**Startup.cs** 中的`ConfigureServices`方法中使用以下代码配置MVC

```csharp
public void ConfigureServices(IServiceCollection services)
{
    .....
        
    services.AddMvc().ConfigureApplicationPartManager(manager =>
    {
        string path = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location);
        var assmeblyFiles = Directory.GetFiles (path, "*.dll");
        assmeblyFiles.Select(AssemblyLoadContext.Default.LoadFromAssemblyPath)
            .ToList().ForEach(assmebly =>
            {
                if (assmebly.GetTypes().Any(typeof(Controller).IsAssignableFrom))
                {
                        AssemblyPart part = new AssemblyPart(assmebly);
                        manager.ApplicationParts.Add(part);
                }
            });

    });
    
    .....
}

...
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    /*
    app.Run(async (context) =>
    {
        await context.Response.WriteAsync("Hello World!");
    });*/
    app.UseMvc();
}


```

'*.dll' 可以用如 *.plugin.dll 这样的字符串代替，取一个插件命名规则。最好把这个规则放在配置文件中。

接下来编写一个Controller进行测试。

---

项目 -- **Plugin1**

#### 添加**PluginController.cs**

**PluginController.cs**

```csharp
using Microsoft.AspNetCore.Mvc;

namespace Plugin1
{
    [ApiController]
    [Route ("api/[controller]/[action]")]
    public class PluginController : Controller
    {
        public PluginController()
        {

        }

        public string Ping() => "pong";

    }
}
```

编写完之后还需要添加MVC的NuGet包。

```powershell
dotnet add .\Plugin1\ package Microsoft.AspNetCore.Mvc
```

完成NuGet包添加后就可以构建Plugin1项目了。

---

#### 编写tasks.json

等等，如果不想每次都`dotnet build .\Plugin1\`，还需要配置下**tasks.json**

第一次`F5`调试的时候会配置构建任务生成该文件。

**tasks.json**中默认添加了三个任务`build，publish，watch`，现在添加**Plugin1**的构建任务。

编辑**tasks.json** 加入以下内容

```json
{
    "label": "build plugin1",
    "command": "dotnet",
    "type": "process",
    "args": [
        "build",
        "${workspaceFolder}/Plugin1/Plugin1.csproj"
    ],
    "problemMatcher": "$tsc"
},
{
    "label": "build & install plugin1",
    "command": "cp",
    "type": "shell",
    "args": [
        "${workspaceFolder}/Plugin1/bin/Debug/netstandard2.0/Plugin1.dll",
        "${workspaceFolder}/Host/bin/Debug/netcoreapp2.2/Plugin1.dll"
    ],
    "windows": {
        "command": "xcopy",
        "args": [
            "/y",
            "${workspaceFolder}\\Plugin1\\bin\\Debug\\netstandard2.0\\Plugin1.dll",
            "${workspaceFolder}\\Host\\bin\\Debug\\netcoreapp2.2\\Plugin1.dll"
        ]
    },
    "dependsOn": [
        "build plugin1"
    ],
    "problemMatcher": "$tsc"
}
```

在VS Code中执行任务**build & install plugin1** 

`ctrl+shift+p`  输入 `run task` 选择 `build & install plugin1`

*注意，第一次运行命令可能会询问是复制的文件还是文件夹，在任务终端中，输入`F`确定是文件即可。*

执行成功后在**Host**的`bin/Debug/netcoreapp2.2`目录下瞄一眼有没有Plugin1.dll，有的话就👌。

---

#### 启动

`F5`启动项目，然后在浏览器输入https://5001/api/plugin/ping。

当看到浏览器回复了一个'pong' 之后，意味着，成功了😀。

### 注入Services

Plugin1.dll 除了需要响应请求之外，它还需要使用IOC容器进行DI。.NetCore 已经内置了IOC容器，需要解决的问题就是：怎么把这个容器传递给插件🤔。这里使用反射来完成它。

思路是这样，定义一个用来注入服务的接口。在**Host**项目中，需要遍历所有实现了该接口的程序集并调用注入方法，然后在插件项目中实现该接口。当然，这个接口需要放在**Share**项目中。

#### 用来注入的接口*

项目 -- **Share**

在VS Code 中新建**IServicesRegister.cs**

**IServicesRegister.cs**

```csharp
using Microsoft.Extensions.DependencyInjection;

namespace Share
{
    public interface IServicesRegister
    {
        void RegisterServices(IServiceCollection services);
    }
}
```

这里还需要添加**Microsoft.Extensions.DependencyInjection** NuGet 包。

```powershell
dotnet add .\Share\ package Microsoft.Extensions.DependencyInjection
```

*命令是在**PluginDemo** 路径下执行的，可不要混了。*

这个接口极其简单，只有一个`RegisterServices`的方法，我这里保证这个接口有且只有这一个方法，方便待会能找到它。

---

#### 完成注入*

现在在**Startup.cs**中添加一个`RegisterServices`方法（名字是和接口的重复的，这没关系，不要混淆了即可）,或者可以直接`ConfigureServices` 中操作。

**Startup.cs**

```csharp
...
public void ConfigureServices(IServiceCollection services)
{
    ...
    // 最好是在最后一步去注入这些服务
    RegisterServices(services);
    ...
}
    
public void RegisterServices(IServiceCollection services)
{
    // 查找dll文件
    var dllFiles = Directory.GetFiles(Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location), "*.dll");
    // 查找程序集
    var assemblies = dllFiles.Select(AssemblyLoadContext.Default.LoadFromAssemblyPath).ToList();
    assemblies.ForEach(ass =>
    {
        // 获取实现IServiceRegister 接口的类型
        var registers = ass.GetTypes().Where(m => m.IsClass && typeof(IServicesRegister).IsAssignableFrom(m)).ToList();
        registers.ForEach(reg =>
        {
            var instance = Activator.CreateInstance(reg);
            // 方法 保证该接口仅有一个方法的情况下，使用SingleOrDefault，应该用Single。
            var method = typeof(IServicesRegister).GetMethods().SingleOrDefault();
            
            method.Invoke(instance, new object[]
            {
                services
            });
        });
    });
}

...
```

---

#### 编写Service进行测试

项目 -- **Plugin1**

需要添加三个文件

- ServicesRegister.cs
- IAddService.cs
- AddService.cs

**ServicesRegister.cs** 用来实现注入，**AddService.cs** 将实现 IAddService 接口用来测试。

**ServicesRegister.cs** 

```csharp
using System.Diagnostics;
using Microsoft.Extensions.DependencyInjection;
using Share;

namespace Plugin1
{
    public class ServicesRegister : IServicesRegister
    {
        public void RegisterServices(IServiceCollection services)
        {
            Debug.WriteLine("Register plugin1's services", "info: ");

            services.AddTransient<IAddService,AddService>();

        }
    }
}
```

**IAddService.cs**

```csharp
namespace Plugin1
{
    public interface IAddService
    {
        string Add(string a, string b);
    }
}
```



**AddService.cs**

```csharp
namespace Plugin1
{
    public class AddService : IAddService
    {
        public string Add(string a, string b) => a + b;
    }
}
```

最后修改**PluginController.cs** 添加AddService的注入依赖和测试方法。

**PluginController.cs**

```csharp
using Microsoft.AspNetCore.Mvc;

namespace Plugin1
{
    [ApiController]
    [Route("api/[controller]/[action]")]
    public class PluginController : Controller
    {

        readonly IAddService m_addService;
        public PluginController(IAddService addService)
        {
            m_addService = addService;
        }

        public string Ping() => "pong";
        public string PingService(string a, string b) => m_addService.Add(a, b);
    }
}
```

---

#### 启动

先执行 `build & install plugin1`命令，然后`F5`启动。🚀

一切完成，在浏览器地址栏输入https://localhost:5001/api/plugin/pingservice?a=service&b=done。

看看浏览器响应，是否把'service' 和 'done' 拼到一起了呢。

再去调试控制台看打印的`Debug.WriteLine("Register plugin1's services", "info: ");` 有没有正确输出。👏👏👏

### 独立配置文件

当决定让插件成为一个完整独立的功能模块时，插件应该使用独立的配置文件，配置过程和普通的项目配置没有太大区别，这里就简单讲一下。

项目 -- **Plugin1**

添加以下两个文件

- plugin1.json
- Plugin1Config.cs

#### **ServicesRegister.cs**中配Config*

修改**ServicesRegister.cs** 中 方法  *RegisterServices*  如下

```csharp
 public class ServicesRegister : IServicesRegister
    {
        public void RegisterServices(IServiceCollection services)
        {
            Debug.WriteLine("Register plugin1's services", "info: ");
			// 路径可以自行配置
            var config = new ConfigurationBuilder()
                .SetBasePath(Directory.GetCurrentDirectory())
                .AddJsonFile("plugin1.json", false)
                .Build();

            services.Configure<Plugin1Config>(config.GetSection(nameof(Plugin1Config)));

            services.AddTransient<IAddService, AddService>();

        }
    }
```

#### Plugin1Config.cs

**Plugin1Config.cs**

```csharp
namespace Plugin1
{
    public class Plugin1Config
    {
        public string Name
        {
            get;
            set;
        }
        public string Version
        {
            get;
            set;
        }

        public string Author
        {
            get;
            set;
        }
    }
}
```



#### 添加配置文件

**plugin1.json**

```json
{
    "Plugin1Config":{
        "Name":"Plugin1",
        "Version":"1.1",
        "Author":"Your Name"
    }
}
```





想要成功构建**Plugin1**项目还需要添加以下NuGet包

- Microsoft.Extensions.Configuration
- Microsoft.Extensions.Configuration.FileExtensions
- Microsoft.Extensions.Configuration.Json
- Microsoft.Extensions.Options.ConfigurationExtensions

```powershell
dotnet add .\Plugin1\ package  Microsoft.Extensions.Configuration
dotnet add .\Plugin1\ package  Microsoft.Extensions.Configuration.FileExtensions
dotnet add .\Plugin1\ package  Microsoft.Extensions.Configuration.Json
dotnet add .\Plugin1\ package  Microsoft.Extensions.Options.ConfigurationExtensions
```

---

项目 -- **Host**

另外 ，**Host** 中还需要在**Startup.cs**文件 *ConfigureServices*  方法中加上一行

```csharp
 services.AddOptions();
```

---

#### 测试配置

emmm，Controller的代码看起来是这样。

**PluginController.cs**

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Options;

namespace Plugin1
{
    [ApiController]
    [Route("api/[controller]/[action]")]
    public class PluginController : Controller
    {

        readonly IAddService m_addService;
        readonly Plugin1Config m_config;
        public PluginController(IAddService addService, IOptions<Plugin1Config> options)
        {
            m_addService = addService;
            m_config = options.Value;
        }

        public string Ping() => "pong";
        public string PingService(string a, string b) => m_addService.Add(a, b);

        public ActionResult<Plugin1Config> PingConfig() => m_config;
    }
}
```



添加NuGet引用

```powershell
dotnet add .\Plugin1\ package  Microsoft.Extensions.Options
```

---

#### 启动

现在像之前那样构建项目，在启动项目之前，把**Plugin1**的配置文件**plugin1.json**拷贝到**Host**目录下。

```csharp
// 这里我配置的是当前目录
var config = new ConfigurationBuilder()
                .SetBasePath(Directory.GetCurrentDirectory())
                .AddJsonFile("plugin1.json", false)
                .Build();
```



启动之后，在浏览器地址栏输入https://localhost:5001/api/plugin/pingconfig。

看看有没有正常读取到配置。🙂

### 映射Model

在插件中完成某些拓展功能时，可能需要数据库的表结构支持，但是，我不想把插件中的Model放到**Share**项目中，插件应该自己管理自己的Model。（究竟是放在Share中还是放在插件项目中，具体需要看插件的定位）

需要准备一个数据库环境，我这里使用SQLite。

#### 数据库表结构

表名 *t_plugdata*

- Id
- Name
- Data

任意加一些数据进去，比如

| Id   | Name | Data         |
| ---- | ---- | ------------ |
| 1    | Ok   | true         |
| 2    | Pity | I am ok      |
| 3    | Lisa | Hi,I am here |

保证有点数据就好了，有关数据库链接字符串或者其他涉及的东西不在这里细讲。

#### 配置数据库上下文*

项目 -- **Share**

添加**HostDbContext.cs**

**HostDbContext.cs**

```csharp
using System;
using System.IO;
using System.Linq;
using System.Reflection;
using System.Runtime.Loader;
using Microsoft.EntityFrameworkCore;

namespace Share
{
    public class HostDbContext : DbContext
    {
        public HostDbContext(DbContextOptions options) : base(options)
        {
 			
        }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);
            
            // 依旧是使用反射的方式去完成
            var dllfiles = Directory.GetFiles(Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location), "*.dll");
            Array.ForEach(dllfiles, files =>
            {
                var assembly = AssemblyLoadContext.Default.LoadFromAssemblyPath(files);
                assembly.GetTypes().Where(type => type.GetInterface(typeof(IEntityTypeConfiguration<>).FullName) != null)
                    .ToList().ForEach(type =>
                    {
                        dynamic instance = Activator.CreateInstance(type);
                        modelBuilder.ApplyConfiguration(instance);
                    });
            });
        }
        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            if (!optionsBuilder.IsConfigured)
            {
                optionsBuilder.UseSqlite("Data Source=./HostDb.db");
            }
        }
    }
}
```

构建**Share**项目可能用到的NuGet 包

- Microsoft.EntityFrameworkCore
- Microsoft.EntityFrameworkCore.SQLite
- System.Runtime.Loader
- System.Linq

```powershell
dotnet add .\Share\ package  Microsoft.EntityFrameworkCore
dotnet add .\Share\ package  Microsoft.EntityFrameworkCore.SQLite
dotnet add .\Share\ package  System.Runtime.Loader
dotnet add .\Share\ package  System.Linq
```

#### 映射Model 并实现 IEntiyTypeConfigration*

项目 -- **Plugin1**

添加文件 **PluginData.cs**

**PluginData.cs**

```csharp
using System.ComponentModel.DataAnnotations.Schema;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace Plugin1
{
    [Table("t_plugdata")]
    public class PluginData
    {
        public string Id
        {
            get;
            set;
        }

        public string Name
        {
            get;
            set;
        }

        public string Data
        {
            get;
            set;
        }

    }

    public class PluginDataConfig : IEntityTypeConfiguration<PluginData>
    {
        public void Configure(EntityTypeBuilder<PluginData> builder)
        {
            builder.ToTable("t_plugdata");
        }
    }
}
```

构建Plugin1 需要添加的NuGet 包

```powershell
dotnet add .\Plugin1\ package  Microsoft.EntityFrameworkCore
```



#### 注入数据库上下文

项目 -- **Host**

**Host** 中还需要在**Startup.cs**文件 *ConfigureServices*  方法中加上一行

```csharp
services.AddDbContext<HostDbContext>();
```



#### 测试数据库上下文

在**PluginController.cs**中进行注入和测试。

**PluginController.cs**

```csharp
using System.Collections.Generic;
using System.Linq;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Options;
using Share;

namespace Plugin1
{
    [ApiController]
    [Route("api/[controller]/[action]")]
    public class PluginController : Controller
    {

        readonly IAddService m_addService;
        readonly Plugin1Config m_config;
        readonly HostDbContext m_dbContext;

        public PluginController(IAddService addService,
            IOptions<Plugin1Config> options,
            HostDbContext dbContext)
        {
            m_addService = addService;
            m_config = options.Value;
            m_dbContext = dbContext;
        }

        public string Ping() => "pong";
        public string PingService(string a, string b) => m_addService.Add(a, b);

        public ActionResult<Plugin1Config> PingConfig() => Json(m_config);
        public ActionResult<List<PluginData>> PingDb() => m_dbContext.Set<PluginData>().ToList();
    }
}
```

可能需要添加的NuGet包

```powershell
dotnet add .\Plugin1\  package  System.Linq
```

可以说是终于完成最后一步了。

最后一步？我还想补充一点。

**PluginController** *PingDb* 方法中用`m_dbContext.Set<PluginData>()`这种方法使用HostDbContext，为了方便使用，可以写一个HostDbContext的拓展类。大概像这个样子

**HostDbContextExtensions.cs**

```csharp
using Microsoft.EntityFrameworkCore;
using Share;

namespace Plugin1
{
    public static class HostDbContextExtensitions
    {
        public static DbSet<PluginData> PluginDatas(this HostDbContext context) => context.Set<PluginData>();

    }
}
```

emmm，这次是完结了。

#### 启动

构建插件，然后启动项目。

在浏览器地址栏输入https://localhost:5001/api/plugin/pingdb。
怎么样，有结果了吗？👀

## 最后

我的个人观点。

我觉得可以从两个方向去使用插件

1. Host项目功能完备，插件只是拓展功能。
2. Host项目完全依赖插件实现功能

混合使用这两种方式也好，单单只用其中一个方式也好。都没问题，具体取决于手中的键盘，不是么。

还有一点，.NetCore 中已经没有AppDomain的概念了，插件无法实现热插拔。

完结👌。