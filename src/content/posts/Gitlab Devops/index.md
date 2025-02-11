---
title: Gitlab Devops
published: 2019-11-19
category: 技术人生
tags: ["Devops"]
image: gitlab.jpg
---

## 概念

在这一篇文章中我主要把工作中碰到的 devops 的问题记录下来，公司现有采用的是用 git 来管理代码，所以顺理成章的使用了 gitlab 来做公司的仓库管理。所以对于放代码的流程，我这里使用了 gitlab 自带的 ci cd 工具来做，但是大体上使用任何其他工具都是差不多的，思维上都是相通的。

### 什么是 ci cd？

对于这个话题，网上有很多的讨论，这里我仅仅是根据我工作中碰到的问题做一个简单的总结。
Ci 全称是持续集成，因为每个项目可能是由很多的小的模块组成的，而我们做的开发任务是把大的任务给分解掉来做小的任务开发。对于这一个理念在 git_flow 上面具体的体现就是使用分支来管理，每一次有一个新的需求过来的时候，我们都会从 dev 分支上面来开一个新的特性分支。当新的特性分支我们做好之后就会对这个分支进行测试，就需要把分支上的改动和现有系统集成起来，这个操作就可以交给自动化集成来做了。

至于 cd 的话全称是 持续交付，简单理解就是我们做的东西需要放在服务器上或者是交付给用户使用，这个过程往往涉及到编译打包部署这些过程，这些重复性的工作就可以交给 cd 来做。

### gitlab 的 ci cd

Gitlab 提供了 ci 和 cd 的服务。具体的步骤可以大概分为以下几个步骤。

1. 服务器中注册 runner
2. Runner 中添加指定项目，每个项目都有个 token
3. 在工程目录下建立.gitlab.yml 文件，按照语法编写自动化流程
4. 推送这个文件，runner 会自动读取这个文件，执行自动化。
5. 每一次推送代码都会触发自动化流程。

#### 基本的 ci cd 流程

![gitlab 定义的自动化流程](https://tva1.sinaimg.cn/large/006y8mN6ly1g9dux2owfjj31i60u0dhg.jpg)

1. 本地开发建立新的功能分支
2. 推送代码变更
3. 自动构建和测试
4. 修复问题推送代码
5. 自动构建和自动测试
6. 部署测试环境
7. review 和 提交 mr
8. 同意合并
9. 自动构建，测试， 部署到生产环境

#### 深层次的 ci cd 流程

![深层次的流程](https://tva1.sinaimg.cn/large/006y8mN6ly1g9duxsr9j5j31b00tojt8.jpg)

### Caching

缓存，在做 gitlab 中有几个概念提前了解一下，对于后面编写自动化流程上还是很有好处的。

#### cache

在自动化集成过程中，每次都要重新安装第三方库依赖，可以通过使用 cache 将安装的依赖库文件缓存起来，避免了每次下载。

#### artifacts

Artifacts 也是一种 cache，不同的是它是构建之后的产物的缓存，不是依赖的。比如我们每次要将前端工程打包之后产生 dist 文件。可以将 dist 文件缓存起来，方便之后去还原。设置了 artifacts 之后 web gui 里就会显示下载 artifacts 的选项。

### stage vs job vs pipelines

总的来说 pipelines 就是流水线的意思，每个流水线中都是按照 stage 的顺序来执行的，stage 中可以定义多个 job，每个 job 之间都是并行的，而 stage 之间是串行的。只有当前的 stage 执行完毕才会执行下一个 stage。

下面是一个最基础的 yaml 文件模版。

```yaml
image: node:<版本号>

services:
    - mongo:<版本>
    - redis:<版本号>

cache:
    key: <KEY>
    paths:
        - <需要缓存的路径>
        - node_modules

before_script:
    - <执行job之前的命令>

after_script:
    - <build完之后运行的命令>

stages:
    - test
    - build
    - deploy

<job_name>:
    script:
        - <job执行的指令或脚本>
    tags:
        - ssr
    only：
        - <只应用在某个分支>
    except:
        - <不应用到某个分支>
    variables:

```

上面例子中还有几个东西值得说下：

- tag 指的是你在 gitlabrunner 上面所注册的 tag 名，也就是这个自动化是使用那台机器去执行。
- before_script 指的是在每个 job 执行前所执行的命令
- after_script 指的是每个 job 执行后所执行的命令

## 完整的流程

![完整的流程](https://tva1.sinaimg.cn/large/006y8mN6ly1g9dy6oiatsj30pv0493ye.jpg)
上图算是一个完整的流程了，包含 build，测试，预生产，生产。
按照现在项目的复杂程度只是用到了上述将的，还有些东西会在后面继续补充。

说到最后，最好自己能动手操作一下，真正部署一遍，这样更好能理解到每个配置的作用。

# 引用

[gitlab 文档](https://docs.gitlab.com/runner/)
