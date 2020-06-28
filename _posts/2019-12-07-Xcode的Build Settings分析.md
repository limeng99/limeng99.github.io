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

Build Libraries for Distribution : Xcode11新增，构建库时选择Yes。

Build Variants : 此项可以设定生成产品的变种。您可以创建额外的产品变种作为特殊用途。例如，您可以使用编译配置
文件的名称来创建一个高度定制的二进制文件。
Build Variants的值有三个 : 
	normal-用于生成普通的二进制文件；
	profile-用于可以生成配置信息的二进制文件；
	debug-用于生成带有debug标志、额外断言和诊断代码的二进制文件。
	
Compiler For C/C++/Objective-C : 选择使用的编译器。

Debug Information Format : 记录debug信息的文件格式。共有DWARF with dSYM File和DWARF两种可以选择。建议
选择DWARF with dSYM File。DWARF是较老的文件格式，会在编译时将debug信息写在执行文件中。

Enable BitCode : bitcode是被编译程序的一种中间形式的代码。包含bitcode配置的程序将会在App store上被编译
和链接。bitcode允许苹果在后期重新优化我们程序的二进制文件，而不需要我们重新提交一个新的版本到App store上。
对应iOS，bitcode是可选的；对于watchOS，bitcode是必须的；Mac OS不支持bitcode。

Enable Index-While-Building Functionality : 控制编译器在编译时是否应该发出索引数据。

Enable Testability : 是否允许测试性。当该设置被激活时，将使用适合运行自动化测试的选项构建产品，例如让测试可以访问私有接口。这个选项可能导致编译速度变慢。

Excluded Source File Names : 在编译阶段不包括的源文件。这个设置一般用于定义复杂的筛选器，比如，
*.$(CURRENT_ARCH).c排除基于正在构建的体系结构的特定文件。

Generate Profiling Code : 是否生成配置代码。是否生成分析代码，此选项为Yes的时候，编译器和链接器会生成分析
代码。

Precompiled Header Uses Files From Build Directory : 预编译build路径中的头文件。由于编译过程比较耗时，且两次编译之间未必会改动所有文件。因此将不会改动的常用文件保留成预编译文件将大大减少编译时的时间。建议这一项选择YES。

Require Only App-Extension-Safe API : 如果我们要想应用扩展使用内嵌框架，那么首先要配置一下。将target的Require Only App-Extension-Safe API选项设置为Yes。如果你不这样设置，那么Xcode会向你提示警告：linking against dylib not safe for use in application extensions。

Scan All Source Files for Includes : 扫描include文件所包含的所有源文件。

Validate Built Product : 这个选项决定了是否在编译的时候进行验证。验证的内容和app store的审查内容一致。默认选项是debug时不验证，release时验证。
```

### 5. Search Paths

![xcode-searchpaths](https://raw.githubusercontent.com/limeng99/limeng99.github.io/master/assets/img/screenshots/xcode-searchpaths.png)

```
Always Search User Paths : 是否搜索用户指定的路径，默认No。
Framework Search Paths : 工程引用的framework搜索路径。
Header Search Paths : 工程中引用的头文件搜索路径。
Library Search Paths : library搜索路径，比如静态.a库。
Sub-Directories to Exclude in Recursive Searches : 指定哪些类型的子目录在递归查找时忽略。
Sub-Directories to include in Recursive Searches : 指定哪些类型的子目录在递归查找时包含。
User Header Search Paths : 设置头文件搜索路径，这个只有当Always Search User Path开启后才有效。
```

### 6. Packaging
![xcode-packasing](https://raw.githubusercontent.com/limeng99/limeng99.github.io/master/assets/img/screenshots/xcode-packaging.png)

```
Defines Module : 是否定义模块。默认App类的工程为No，Framwork工程默认为Yes。

Expand Build Setting in Info.plist File : 告诉编译器是否处理info.plist。默认是Yes。这是一个很大的特点，因为它避免了有根据您的构建设置和配置不同的Info.plist中，避免您在多个地方修改设置。 但是如果你真的不想要它，只需在项目或目标的构建设置中关闭此设置。

Info.plist File : 创建工程后默认会创建一个info.plist文件。也可以根据需要进行主动创建。
Private Headers Folder Path : 私有头文件的存放位置。
Product Bundle Identifier : 产品bundle id。
Product Module Name : 应用模块名称。
Product Name : 应用名称。
Public Headers Folder Path : 公共头文件路径。
Wrapper Extension : 打包的扩展名，默认app。
```


### 7. Signing

![xcode-signing](https://raw.githubusercontent.com/limeng99/limeng99.github.io/master/assets/img/screenshots/xcode-signing.png)

```
Code Signing Entitlements : 授权机制。在Xcode的capabilities选项卡下选择一些选项后，Xcode就会生成这样一段XML，Xcode会自动生成一个entitlements文件，然后再需要的时候往里面添加条目。当构建整个应用时，这个文件也会提及给codesign作为应用所需要拥有哪些授权的参考。这些授权信息必须都在开发者中心的AppID中启用，并且包含在配置文件中。

Code Signing Identity : 配置证书。
Code Signing Style : Automatic自动配置, Manual手动配置。
Development Team : 开发者所在的群组。
Provisioning Profile : 配置描述文件。
```

