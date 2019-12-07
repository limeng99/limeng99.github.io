---
layout: post
title: "Xcode的Build Settings分析"
author: "李萌"
categories: learning
tags: [learning]
feature-img: "assets/img/article/xcode.jpg"
thumbnail: "assets/img/article/xcode.jpg"
typora-root-url: ../assets
---

开发之余，对于Xcode工程配置中Build Settings了解不够清晰准确，通过查阅资料，希望能够通过这篇文章加深Build Setting的理解，并记录分享。

### 1. Architectures

![xcode-architectures](https://raw.githubusercontent.com/limeng99/limeng99.github.io/master/assets/img/screenshots/xcode-architectures.png)

```
Additional SDKs : 在编译时要附加的SDK。
Architectures : 支持的处理器架构。支持的指令集越多，就会编译出包含多个指令集代码的数据包，对应生成二进制包就
会越大，最终目标文件也会变大。
Base SDK : 当前编译用的SDK版本。
Build Active Architecture Only : 如果此项为Yes，则Xcode会根据当前所连接的设备版本只将相应的Architecture编
译入App。否则会同时编译'Valid Architectures'中的指令集。建议在Debug模式下设置为Yes，Release模式下设置为No，
加快编译速度。
Supported Platform : App支持的平台。
Valid Architectures : 限制可能被支持的指令集的范围，也就是Xcode编译出来的二进制包类型最终从这些类型产生，而
编译出哪种指令集的包，将由Architectures与Valid Architectures（因此这个不能为空）的交集来确定。
```

Valid Architectures指令集代表什么？

```
armv7｜armv7s｜arm64都是ARM处理器的指令集
i386｜x86_64 是Mac处理器的指令集
i386是针对intel通用微处理器32位处理器
x86_64是针对x86架构的64位处理器

arm64：iPhone6s | iphone6s plus｜iPhone6｜ iPhone6 plus｜iPhone5S | iPad Air｜ iPad mini2(iPad 
mini with Retina Display)
armv7s：iPhone5｜iPhone5C｜iPad4(iPad with Retina Display)
armv7：iPhone4｜iPhone4S｜iPad｜iPad2｜iPad3(The New iPad)｜iPad mini｜iPod Touch 3G｜iPod Touch4

模拟器32位处理器测试需要i386架构，模拟器64位处理器测试需要x86_64架构，
真机32位处理器需要armv7,或者armv7s架构，真机64位处理器需要arm64架构。
```

### 2. Assets

![xcode-assets](https://raw.githubusercontent.com/limeng99/limeng99.github.io/master/assets/img/screenshots/xcode-assets.png)

```
Asset Pack Manifest URL Prefix : 资源包清单的下载路径URL前缀
Embed Asset Packs In Product Bundle : 是否将资源包嵌入产品的bundle中
Enable On Demand Resources : 是否开启按需获取资源功能
On Demand Resources Initial Install Tags : 按需加载资源时的初始安装资源文件标签
On Demand Resources Prefetch Order : 按需加载资源时预加载的标签顺序
```

### 3. Build Locations

![xcode-locations](https://raw.githubusercontent.com/limeng99/limeng99.github.io/master/assets/img/screenshots/xcode-locations.png)

```
Build Products Paths : 产品文件和编译中间文件的根目录。产品文件和编译时临时文件都将放在这个目录的子目录中。
Intermediate Build Files Path : 编译时临时文件的存放位置。编译中间文件格式为product name+.build，
如MyProduct.build。
Per-configuration Build Products Path : 当前编译设置下的产品存放位置。
```

### 4. Build Options

![xcode-options](https://raw.githubusercontent.com/limeng99/limeng99.github.io/master/assets/img/screenshots/xcode-options.png)

```
Always Embed Swift Standard Libraries : 始终嵌入swift标准库。对于未使用swift代码的情况可以设置为NO。

```

