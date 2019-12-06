---
layout: post
title: "CocoaPods创建属于自己的Pod库"
author: "李萌"
categories: learning
tags: [learning]
feature-img: "assets/img/article/pod.jpg"
thumbnail: "assets/img/article/pod.jpg"
---

### 前言

项目想要模块化、组件化，就必须了解如何创建CocoaPods库，如何创建CocoaPods库呢，今天我们就来动手开始从头建立属于自己的CocoaPods库吧！

### 创建公有pod库

#### 1. 注册CocoaPods账户信息(**已注册过请略过)**

```
创建一个开源pod库, 首先我们需要注册CocoaPods账户, 使用trunk方式在终端执行:
$ pod trunk register '邮箱地址' '用户名'  --description=‘描述内容’ --verbose

可以使用GitHub邮箱和用户名, 然后在你的邮箱中会收到确认邮件在浏览器中点击链接确认即注册成功, 
成功之后可以终端执行:
$ pod trunk me
```

#### 2. 创建共享库文件, 上传到公有仓库

在[GitHub](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2F)上创建一个公开项目，创建指南 [Using Pod Lib Create](https://guides.cocoapods.org/making/using-pod-lib-create.html)

```
如果自己新建的库，可以用下面代码创建
$ pod lib create 库名

项目中必须包含这几个文件：
1. 共享文件夹：文件夹存放着你要共享的内容, 也就是其他人pod得到的文件, .podspec文件中的
   source_files需要指定此文件路径及文件类型
2. LICENSE：开源许可证，默认一般选择MIT;
3. README.md：仓库说明
4. 库描述文件.podspec：本库的各项信息描述, 需要提交给CocoaPods, pod通过这个文件查找到你共享的库.
```

然后可以使用[SourceTree](https://www.sourcetreeapp.com/)等工具上传你的代码到公共仓库, 或使用命令行上传代码到远端仓库, 如何操作可以移步:[Git常用命令](https://limeng99.club/learning/2019/11/19/Git%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4.html).

#### 3. 编辑*.podspec文件

`.podspec`是用`Ruby`的配置文件，描述你项目的信息，现在创建库后已自动生成*.podspec文件

```
自己想要创建podspec文件，执行以下命令
$ pod spec create 库名
```

`.podspec`文件内容

```
Pod::Spec.new do |s|
  s.name         = "LMBannerView" # 项目名称
  s.version      = "0.0.1"        # 版本号 与 你仓库的 标签号 对应
  s.summary      = "轮播图"        # 项目简介
  s.description  = <<-DESC
  这中间写描述内容
               DESC
  s.homepage     = "https://github.com/1805441570@qq.com/test" # 你的主页
  s.license      = { :type => 'MIT', :file => 'LICENSE' } # 开源证书
  s.author       = { "limeng" => "1805441570@qq.com" } # 作者信息
  # 你的仓库地址，不能用SSH地址
  s.source       = { :git => "https://github.com/limeng99/LMBannerView", :tag => "0.0.1"}
  # 代码位置， LMBannerView/**/*.{h,m} 表示 ** 文件夹下所有的.h和.m文件
  s.source_files     = "LMBannerView/Classes/*.{h,m}"
  s.requires_arc = true # 是否启用ARC
  s.platform     = :ios, "8.0" # 平台及支持的最低版本
  s.frameworks = "UIKit", "Foundation" # 支持的框架
  # s.dependency "AFNetworking", "~> 2.3" # 依赖库
end
```

验证 `.podspec` 文件的格式是否正确，`cd` 到 `*.podspec` 文件所在的目录下

```
$ pod lib lint 库名.podspec --allow-warnings
例如: $ pod lib lint LMBannerView.podspec --allow-warnings
```

验证成功会出现：

```
-> LMBannerView  (0.0.1)

LMBannerView passed validation.
```

#### 4. 给仓库打Tag

标签相当于将你的仓库的一个压缩包，用于稳定存储当前版本。标签号与你在 `s.version = "0.0.1"` 的版本号一致 `0.0.1`

```
创建标签
$ git tag -a 0.0.1 -m '标签说明'

推送到远程
$ git push origin --tags
```

#### 5. 发布 *.podspec

```
仓库目录下执行
$ pod trunk push LMBannerView.podspec --allow-warnings

将你的 *.podspec 发布到公有的 speecs 上，这一步操作包括：
1. 更新本地 pods库 ~/.cocoaPods.repo/master
2. 验证*.podspec格式是否正确
3. 将 *.podspec 文件转成 JSON 格式
4. 对 master 仓库进行合并、提交
```

成功后会出现`Congrats`信息如下：

```
limMac-2:LMBannerView limengmeng$ pod trunk push LMBannerView.podspec --allow-warnings
Updating spec repo `trunk`
Validating podspec
 -> LMBannerView (0.1.3)
    - WARN  | github_sources: Github repositories should end in `.git`.
    - NOTE  | xcodebuild:  note: Using new build system
    - NOTE  | [iOS] xcodebuild:  note: Planning build
    - NOTE  | [iOS] xcodebuild:  note: Constructing build description
    - NOTE  | [iOS] xcodebuild:  warning: Skipping code signing because the target does not have an Info.plist file and one is not being generated automatically. (in target 'App' from project 'App')

Updating spec repo `trunk`

--------------------------------------------------------------------------------
 🎉  Congrats

 🚀  LMBannerView (0.1.3) successfully published
 📅  December 6th, 02:40
 🌎  https://cocoapods.org/pods/LMBannerView
 👍  Tell your friends!
--------------------------------------------------------------------------------
```

#### 6. 使用仓库

发布到Cocoapods后，在终端更新本地pods仓库信息

```
$ pod setup
$ pod search LMBannerView

如果`search`出现以下错误：
[!] Unable to find a pod with name, author, summary, or description matching `LMBannerView`

删除 cocoapods 的索引，然后重新 search
$ rm ~/Library/Caches/CocoaPods/search_index.json
$ pod search LMBannerView
终端输出：Creating search index for spec repo 'artsy'.. Done!
会触发cocoapods重新拉这个索引文件
```

#### 7. 更新维护

- 更新 `*.podspec` 中的版本号
- 打上标签推送流程
- `pod trunk push *.podspec` 推送到`pods`仓库

#### 8. 遇到的问题

-  `xcodebuild: Returned an unsuccessful exit code.` error，`*.podspec` 文件里面设置的 `s.ios.deployment_target = '8.0'`，最低支持版本为 `8.0`，而在代码里面用到 了 `xib`，里面勾了 `Use Safe Area Layout Guides`，项目支持最低版本为 `9.0`，`Build` 不会报错，执行 `pod lib lint *.podspec` 报 `xcodebuild: Returned an unsuccessful exit code.`，原来是 `Use Safe Area Layout Guides` 的问题，修改 `*.podspec` 文件 `s.ios.deployment_target = '9.0'` 修复了这个问题

- `xib` 报 `Use Safe Area Layout Guides` error, 修改 `xib` 的 `Builds for` 选项 改为  `Deployment Target (9.0)`

### 创建私有库

私有Pod库和公有Pod库的创建方式没有什么区别, 不一样的是管理他们的spec repo不一样

所以我们需要自己创建一个跟[CocoaPods/Specs](https://github.com/CocoaPods/Specs)类似的仓库来管理内部创建的Pod库的podspec文件, 供内部人员更新和依赖使用内部Pod组件库。
私有repo的构建形式有两种, 一种是私有git服务器上面创建，一种是本机创建。本机创建请参考官方文档:[Private Pods](https://guides.cocoapods.org/making/private-cocoapods.html).

这里介绍的是在公司内部搭建的git服务器上面创建整个服务的方式。
#### 1. 创建一个git仓库用来做内部私有库的Spec Repo

在私有服务器创建一个仓库,一个用来存放所有共享库的podspec, 这里创建好之后的内部SSH协议地址是:[git@git.xxxx:podspecs/xxx.git](git@git.xxxx:podspecs/xxx.git), 创建的私有仓库给到的http/https地址也一样.终端输入命令:

```
$ pod repo add PodSpec git@git.xxxx:podspecs/xxx.git
```

将PodSpec添加到本地repo, 添加成功后可以在/.cocoapods/repos/目录下可以看到官方的specs:master和刚刚加入的specs:PodSpec

#### 2. 创建私有Pod组件库

这一步跟上面创建共有pod库方式是一致的。即创建git工程，podspace文件，pod lib lint , 打tag 。但不是用trunk方式提交到master上

#### 3. 将podspec加入私有Sepc repo中

公有库使用trunk方式将.podspec文件发布到[CocoaPods/Specs](https://github.com/CocoaPods/Specs), 内部的pod组件库则是添加到我们第一步创建的私有Spec repo中去, 在终端执行:

```
$ pod repo push podspec名称  Demo.podspec
```

添加成功之后PodSpec中会包含新建Demo库的podspec信息, 可以前往~/.cocoapods/repos下的PodSpec文件夹中查看, 同时git服务器中的远端也更新了.

#### 4.查找和使用内部组件库

执行pod search Demo库 就能查到刚刚创建好的库了，然后在想要使用此组件的工程的Podfile中加入pod 'Demo', '~>0.0.1'即可使用内部组件啦！
 值得注意的是:必须在Podfile前面需要添加你的私有Spec repo的git地址source, pod install时, 才能在私有repo中查找到私有库, 像这样:

```
source 'git@git.xxx.net:podspecs/PodSpec.git'
    
    platform :ios, '9.0'
    target "test" do
        pod 'Demo', '~>0.0.1'
    end
```
