---
title: Macbook 外接显示器开启HiDPI
published: 2018-11-08
category: Apple
tags: ["Apple"]
image: MacBook.jpg
draft: false
---

## 历史

2010 年乔老爷子发布 iPhone4 的时候带来一个新的名词."Retina 屏".
中文名称为视网膜屏.自此以后苹果产品线的多数产品都使用了这项显示技术. 很多人从浅意识上认为分辨率越高,显示的越清晰,很多手机厂商将手机屏幕分辨率做到了 2k,4k,观感上还没有苹果 1k 的显示清楚,准确来说 Retina 是一种显示技术,它让更多的像素点压缩到一块显示屏幕,从而达到更细腻的效果.

苹果今年的新品 iPhone Xr 的分辨率只有 720p 但是通过看真机其实没有网上云测评那么糟糕,得益于苹果的 Retina 技术,iPhone Xr 上面的显示效果令人满意.

## HiDPI

那么 HiDPI 是什么呢?

HiDPI 本质上是用软件的方式实现单位面积内的高密度像素。很多用过高分屏的朋友可能清楚越高分辨率的显示器显示的文字和图标越小,其实不然,可以通过 HiDPI 的方式在保证分辨率不变的情况下,使得字体和图标变大.达到平滑细腻的视网膜屏的显示效果.
![](https://ws4.sinaimg.cn/large/006tNbRwly1fx1159w6gij31kw1a8b2a.jpg)

## 外接显示器开启 HiDPI

打开系统自带终端,按照以下步骤操作.

### 开启 HiDPI

```
sudo defaults write /Library/Preferences/com.apple.windowserver.plist DisplayResolutionEnabled -bool true
```

回车后,输入当前管理员密码.

### 获取 ID

分别输入以下两个命令

```
ioreg -l | grep "DisplayVendorID"
ioreg -l | grep "DisplayProductID"

```

这两个命令会输出两个 10 进制的数字,一个是 DisplayVendorID,一个是 D isplayProductID.拿一个记事本记录下来.
![](https://ws1.sinaimg.cn/large/006tNbRwly1fx11cg8ywqj311w0vcaj7.jpg)
如果说你和我一样现在是两个显示器的话,那么记住第二个,第一个为 MacBook Pro 的内置显示器.对应的截图中的内容为 **4724** **9984** .

### 将得到的 ID 从 10 进制转为 16 进制

![](https://ws1.sinaimg.cn/large/006tNbRwly1fx11k4yftwj317o0hk0ut.jpg)
![](https://ws2.sinaimg.cn/large/006tNbRwly1fx11gxy9k5j315k0hsgno.jpg)

如图所示笔者的两个 id 转为 16 进制得到的是 aa4,2700.
转换工具,大家可以自行搜索.

### 建立文件夹

在任意地方建立一个文件夹,文件夹命名格式:DisplayVendorID-XXXX，其中 XXXX 即为你的 DisplayVendorID 的 16 进制值小写。对应的笔者的文件夹名为: **DisplayVendorID-1274**.

文件夹建好以后,在文件夹里面新建一个名称为:DisplayProductID-YYYY 的空文件**（没有扩展名）**。YYYY 即为你的 DisplayProductID 的 16 进制值小写。对应的笔者的文件名为: **DisplayProductID-2700**.

### 生成文件内容

[在线生成文件配置](https://comsysto.github.io/Display-Override-PropertyList-File-Parser-and-Generator-with-HiDPI-Support-For-Scaled-Resolutions/)
![](https://ws2.sinaimg.cn/large/006tNbRwly1fx11ylyzokj315y132q86.jpg)

如图,将上述步骤得到的 ProductID 还有 VendorID 的十六进制填入到网页中,将需要开启的 HiDPI 的分辨率填入其中(最好和你外接显示器的分辨一致或者缩放大小)

将配置好的内容从网页上粘贴到上一步骤建立的空文件中即可.
![](https://ws4.sinaimg.cn/large/006tNbRwly1fx11xbcg75j30um12aq9z.jpg)

### 移动文件夹

将 DisplayVendorID 的文件夹拷贝到/System/Library/Displays/Contents/Resources/Overrides/
(注：Mac OS 10.10 及以下是 /System/Library/Displays/Overrides/ )中
![](https://ws1.sinaimg.cn/large/006tNbRwly1fx11v7t3hoj311w0vcn5y.jpg)
可使用图片中的命令快速打开文件夹.

### 下载 RDM 切换分辨率

[RDM](http://avi.alkalay.net/software/RDM/)是一个用来切换分辨率的软件.

### 重启,切换分辨率

用 RDM 来切换分辨率
![](https://ws2.sinaimg.cn/large/006tNbRwly1fx11teqrayj30iu0makcd.jpg)
注意:带 ⚡️ 标志的才是开启了 HiDPI

### Enjoy yourself
