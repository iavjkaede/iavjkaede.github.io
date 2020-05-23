---
title: NetCore 使用 ImageSharp 为图片添加水印
date: 
categories: 
	- .NET
tags: 
	- .NET
	- Image
cover: https://image.zsver.com/2020/05/23/43208491ef0e8.jpg
---

## 环境简介

### 系统环境

Windows10 1903

### .NetCore SDK 版本

.NetCore 2.2

### ImageSharp 版本

SixLabors.ImageSharp  1.0.0-beta0006

## 1.安装ImageSharp Nuget 包

我写这篇博客时，ImageSharp还没有一个正式的发布版本，所以这里使用的是预览版本。

如果通过Visual Studio 安装Nuget 包，记得勾选**包含预发行版本**。如果不知道ImageSharp 是什么，可以先百度一下。

[ImageSharp 项目的 github 地址](https://github.com/SixLabors/ImageSharp)

通过CLR 安装ImageSharp

```bash
dotnet add '项目名称' package SixLabors.ImageSharp --version 1.0.0-beta0006
```

## 2. 编写添加水印的代码

```csharp
public Stream Watermark(Stream origin, Stream mark, string position, float scale, float opacity)
 {

     // TODO: 参数校验
     // origin mark 不可空，
     // position 可指定默认值
     // scale opacity 不可小于等于0

     using(var originImage = Image.Load(origin))
     {
         using(var markImage = Image.Load(mark))
         {
             // 设置水印的大小，如果图片宽大于高， scale缩放的是水印图的宽度，否则将应用到高度上
             float markWidth = originImage.Width * scale;
             if (originImage.Width < originImage.Height)
                 markWidth = originImage.Height * scale;

             // 对水印图片进行等比缩放
             float ratio = markWidth / markImage.Width;
             float markHeight = markImage.Height * ratio;

             markImage.Mutate(mk =>
                 mk.Resize((int)markWidth, (int)markHeight)

             );

             // 根据position 计算水印位置  
             // position可能的值如下 "left top botom center right"
             // 这些值可以进行组合 如果同一方向有重复值，后面的会覆盖掉前面的 
             // 按照 中左上右下的顺序处理
             position = position.ToLower();
             Point point = new Point();

             if (position.Contains("center"))
             {
                 point.X = (originImage.Width - markImage.Width) / 2;
                 point.Y = (originImage.Height - markImage.Height) / 2;
             }

             if (position.Contains("left"))
                 point.X = 0;

             if (position.Contains("top"))
                 point.Y = 0;

             if (position.Contains("right"))
                 point.X = originImage.Width - markImage.Width;

             if (position.Contains("bottom"))
                 point.Y = originImage.Height - markImage.Height;

             // 把水印加到图片上
             originImage.Mutate(oi =>
             {
                 oi.DrawImage(markImage, point, opacity);
             });

             origin.Position = 0;
             var format  = Image.DetectFormat(origin);
             var encoder = Configuration.Default.ImageFormatsManager.FindEncoder(format);
             var outStream = new MemoryStream();

             originImage.Save(outStream, encoder);
             origin.Position    = 0;
             mark.Position      = 0;
             outStream.Position = 0;
             return outStream;
         }
     }
 }
```

## 最后

ImageSharp 的使用起来很简单方便，有关更多使用ImageSharp的细节，可以参考[ImageSharp 的文档](https://docs.sixlabors.com/articles/ImageSharp/GettingStarted.html)