---
title: iOS 自动化打包发布
published: 2019-04-15
category: 技术人生
tags: ["iOS"]
image: iOS.jpg
---

从远古`iOS`开发过来的开发人员都经历过一段痛苦的开发过程，被苹果的证书以及缓慢的打包上传速度所折磨，这几年兴起的`devops`使得外国一些大佬鼓捣出一些`iOS`的`ci cd`工具。各种脚本各显神通，不过今天我想说的是其中一个比较著名的工具 **[fastlane]**(<https://github.com/fastlane/fastlane>) 。截止到目前为止`github`上面的`start`数目已经达到 **24978** 个。

![](https://ws2.sinaimg.cn/large/006tKfTcly1g1qnxk4gm4j30o708gdfy.jpg)

## fastlane 是什么

> _fastlane_ is a tool for iOS and Android developers to automate tedious tasks like generating screenshots， dealing with provisioning profiles， and releasing your application.

官网上是这样介绍的，帮助 iOS 和安卓开发者自动化操作一些任务比如说截图，管理证书或者发布应用。

## fastlane 如何用

按照官网的`doc`进行[操作](https://docs.fastlane.tools/getting-started/ios/setup/)。一共分为 3 个步骤：

### 安装

首先安装 xcode 命令行工具：

```shell
xcode-select --install

```

接着使用`gem` 安装 `fastlane` 或者 使用`homebrew`也都行.(题外话说两句，gem 是`ruby`的包管理工具，由于`fastlane`是使用`ruby`来编写的一个工具，所以可以使用 gem 来安装。对于 iOS 开发中大名鼎鼎的`Cocoapods`也是使用`ruby`来编写的，我们安装是也是通过`gem`来安装的)。`homebrew` 家庭酒酿这个是用来喝酒的。。 大雾。`homebrew`是`Mac`里面的一个包管理工具，它使得我们脱离各种繁琐的环境配置，只需要输入命令安装，卸载，`brew`管理包都是存放在`/usr/local`下，通过软链接的形式来使用。

![homebrew](https://ws2.sinaimg.cn/large/006tKfTcly1g1qoca8ypuj30l40ac0uc.jpg)

扯远了，使用下面的命令即可安装：

如果没有安装`homebrew` (<https://brew.sh/>) 的去官网按照步骤安装即可：

```shell
# Using RubyGems
sudo gem install fastlane -NV

# Alternatively using Homebrew
brew cask install fastlane
```

### 使用

用命令行`cd`到工程目录下，使用命令初始化`fastlane init：`

![步骤一](https://ws2.sinaimg.cn/large/006tNc79ly1g23a6ecb4tj30l40glgpc.jpg)

命令行会显示几种不同的自动化方式，我们选择上传`App store`的方式。结束之后的文件目录形式如下图：

![步骤二](https://ws2.sinaimg.cn/large/006tNc79ly1g23a5nid3pj30is0qeacs.jpg)

可以看到文件夹中多了一个文件和一个文件夹，`gemfile` 这个文件很简单，类似于`iOS`工程的`Podfile`一样，里面存放的是`ruby`所需要的第三方库，我们看到里面的内容，可以看到填写的是`fastlane`。再看到文件夹，文件夹里面有两个文件，我们重点关注的是`fastfile`这个文件。

```ruby
source "https://rubygems.org"

gem "fastlane"
```

### Appfile

`Appfile` 主要存储的是一些每一个`fastlane`脚本需要使用的通用信息，比如说 `Apple ID` 或者是 `APP` 的`Bundle Id`。更详细的配置可以参考官网的[doc](https://docs.fastlane.tools/advanced/Appfile/#appfile)。

### FastFile

这里就是我们自定义脚本的地方，可以打开这个文件看到里面有了一段默认的代码：

```ruby

default_platform(:ios)

platform :ios do
  desc "Description of what the lane does"
  lane :custom_lane do
    # add actions here: https://docs.fastlane.tools/actions
  end
end

```

这里的语言使用的是`ruby` 语言，不太了解的可以看看`ruby`的语法，也挺简单的。我这里简单的说一下。`default_platform(:ios)`调用一个函数设置默认的平台`iOS`。后面针对`iOS`的平台来写脚本，`lane` 是定义一个模块的意思，我们自定义的每个不同的操作都可以用`lane`来表示。

`fastlane`里面有很多已经被人写好的[`action`](https://docs.fastlane.tools/actions/)，`aciton`就是类似于一段别人写好的函数，我们可以在我们自定义的`lane`中调用。在官方的 doc 中已经有很多 action 了，测试，构建，截图，打包，证书管理，上传到 App Store 等等。

这里我自己写了个测试的 lane，没有做测试，仅供参考，让读者大概知道这么个意思就行，具体的还是需要根据自己的实际需求进行测试和验证。

```ruby
  lane :releaseDemo do
    app_identifier = "demo.com"
    cocoapods

    get_certificates(
      app_identifier:app_identifier
    )

    get_provisioning_profile(
      app_identifier:app_identifier
    ）

    build_app(workspace: "Project.xcworkspace"， scheme: "test")
    upload_to_app_store(app_identifier:app_identifier，skip_metadata: true，skip_screenshots:true)
  end
```

从第一行代码来看，顾名思义，这一行对下面这个 lane 的描述。这些描述在执行命令之后 fastlane 都会为用户生成一个 md 的文档，文档中显示着每个 lane 的执行方式和描述。接下来的代码使用 lane 来定义一个操作，`releaseDemo` 是这个 lane 的名称，`fastlane`定义了很多的`action`，这些我们都可以直接拿来组合使用，这和调用 api 差不多，具体的参数可以查询官方文档也可以使用命令行来查询。

1. `cocoapods` 这个`action` 是将当前的工程执行`pod install`命令，`podfile`默认在当前工程目录下寻找，如果说不是则需要自己指定`podfle`文件路径。
2. `get_certificates` 是获取证书的，`app_identifier`参数 是`app`的`bundle id`。
3. `get_provisioning_profile` 是获取 `provisioning_profile`的。
4. `build_app` 是用来构建`app`的，`workspace` 参数指定`xcworkspace`文件，`scheme`指定工程的`target`。
5. `upload_to_app_store` 上传构建出`ipa`文件到`APP store`，`skip_metadata`设置为`true`这个参数是用来跳过元数据的，`skip_screenshots` 设置为`true` 用来跳过上传截图。

关于上述的`skip_metadata`和 `skip_screenshots`这两个参数有的读者会比较疑惑，其实 `upload_to_app_store` 这个`action`有个别名叫[`deliver`](https://docs.fastlane.tools/actions/deliver/)，这个`action`可以上`ipa` 还有 `APP store connect` 中需要的元数据和截图。在这里我并没有使用这个功能。在下面会讲到。

上述的 lane 写好之后我们就可以保存，在工程目录下执行

```sh
fastlane releaseDemo
```

就可以看到项目按照我们上面所定义的步骤开始自动化操作了。

### deliver

我们在工程目录下面使用命令行执行

```sh
fastlane deliver init
```

根据提示输入`Apple ID` 和 `Bundle Identifier` ，我们就得到了两个新的文件夹和一个文件.

![文件夹](https://ws3.sinaimg.cn/large/006tNc79ly1g23bvyuqikj30qr0rndk7.jpg)

`Deliverfile` 可以在文件中对上传到 `App Store` 的过程中的操作进行一些配置，比如说设置语言，针对不同的语言来设置不同的元数据。

`metadata` 里面存放的就是`App Store Connect` 中存放关于`app`的元数据，当执行 `fastlane deliver init` 时 这些信息都会从线上下载到本地，包括截图。当我们需要使用 deliver 的功能来上传截图或者修改元数据时，只需要修改这里面对应的文件即可。

### 多 target 打包

想必现在的工程肯定存在多 target。使用的时候就需要建立几个不同的环境配置文件，这种文件是以.env.targetName 这种形式来命名的。下面是公司现在的项目的配置文件的截图。

![配置文件](https://ws1.sinaimg.cn/large/006tNc79ly1g23cdb411aj30cx0c6jsr.jpg)

在里面设置针对不同 target 所需要的不同配置进行填写。

再到 fastfile 中进行使用。

```ruby
  lane :release do |options|
  app_identifier = ENV['app_identifier']
  scheme = ENV['scheme']
  type = options[:type]
 if type.eql?('release')
        #上传到app store
   upload_to_app_store(app_identifier:app_identifier，ipa:ipa_path， skip_metadata: true，skip_screenshots:true)
        #上传dsym到 fabric
      upload_symbols_to_crashlytics(dsym_path: dsym_path)
      elsif type.eql?('beta')
        upload_to_testflight(app_identifier:"com.GoToBus.app"，ipa:"outPut/GotoBus/GotoBus.ipa"， skip_waiting_for_build_processing: true)
      end
```

上面是截取公司的一部分的代码，使用时取得 app id 和 app 的 target name。就可以针对不同的 target 进行打包。 上面代码中有个 option，这是一个可以供用户输入的东西，后面调用的时候可以使用

```sh
    sh "fastlane release type:'release' --env GotoBus"
```

来调用这个这个 lane，option 得到的 type 类型时 release，或者还可以将 release 改成 beta 来打包测试版本的。

最后还可以加上执行成功之后发送邮件或者 slack 消息进行通知。

## More

fastlane 还可以执行测试，自动化截图，生成测试报告等等功能。还有官网上很多的 action，我们可以自定义组合这些东西来生成最适合自己的工作流。当然还可以结合 jekins 或者 Git lab 的 ci/cd 工具都是可行的，本质上通过调用 lane 来进行操作，这些我都进行尝试过。行文比较粗糙。如果对于文章中存在什么问题或者疑问，可以发邮件来和我进行讨论。
