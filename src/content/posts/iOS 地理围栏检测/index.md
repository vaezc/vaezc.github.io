---
title: iOS 地理围栏检测
published: 2019-04-19
category: 技术人生
tags: ["iOS"]
image: location.jpg
---

公司中有个业务是司机端的业务，司机进入到某个站点之后会上传进站的数据到服务器上，出站之后同样上传数据。现有的代码中是用坐标距离来判断进出站的操作，从实际反馈回来的结果来看，这样存在着不准确的情况。还有可能司机将 app 放在后台，app 被系统回收的可能，那样数据都不会上传回来。

经过笔者的一番调研，将目标确定在地理围栏上面。首先我们来看看什么是地理围栏。

## 地理围栏

在百科上面是这样解释地理围栏的

> **地理围栏**（Geo-fencing）是 LBS 的一种新应用，就是用一个虚拟的栅栏围出一个虚拟**地理**边界。 当手机进入、离开某个特定**地理**区域，或在该区域内活动时，手机可以接收自动通知和警告。 有了**地理围栏**技术，位置社交网站就可以帮助用户在进入某一地区时自动登记。

![](https://ws2.sinaimg.cn/large/006tNc79ly1g27xiorjvyj30vt0gptb6.jpg)

> By SpyToMobile (SpyToMobile.com). - Own work., CC BY-SA 4.0, <https://commons.wikimedia.org/w/index.php?curid=34557848>

其实说白了就是一块我们在地图上自定义的块区域，用户进入或者离开时我们都可以检测到。从原理上就初步符合我们的要求。

## 地理围栏在 iOS 中的应用

可以从苹果官方文档中找到对区域检测的[文档](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/LocationAwarenessPG/RegionMonitoring/RegionMonitoring.html#//apple_ref/doc/uid/TP40009497-CH9-SW11%20---------------------%20)。地理围栏也是区域检测中的一种，另一种是苹果在 iOS 7 中推出的 iBeacon 功能。我们将重点放在地理围栏上面。

在 doc 中我们看到这样一句话：

> Apps can use region monitoring to be notified when a user crosses geographic boundaries or when a user enters or exits the vicinity of a beacon. While a beacon is in range of an iOS device, apps can also monitor for the relative distance to the beacon. You can use these capabilities to develop many types of innovative location-based apps. Because geographical regions and beacon regions differ, the type of region monitoring you decide to use will likely depend on the use case of your app.
>
> In iOS, regions associated with your app are tracked at all times, including when the app isn’t running. If a region boundary is crossed while an app isn’t running, that app is relaunched into the background to handle the event. Similarly, if the app is suspended when the event occurs, it’s woken up and given a short amount of time (around 10 seconds) to handle the event. When necessary, an app can request more background execution time using the `beginBackgroundTaskWithExpirationHandler:` method of the `UIApplication` class.

大致的意思就是说：在 iOS 里,地理围栏会一直在监听，包括 app 不在运行的时候。当设备经过地理围栏是，app 会在后台重新启动来执行一段任务。同样的，当 app 在后台挂起的时候，会唤起 app 给一段时间（大概 10 秒左右）让 app 去处理一些任务。必要时，app 还可以通过执行后台任务的方法来获取更多的执行时间。

这句话读完我们心里就有底了，想不到苹果还有这种骚操作，即使 app 被 kill 掉，还能通过触发地理围栏的事件来重新唤起 app 来执行一段操作(厉害了，我的果:laughing:)。

### 限制

当使用地理围栏检测时，app 首先要检测设备是否支持，下面存在这几种情况不能使用地理围栏检测：

- 硬件设备不支持地理围栏检测
- 用户没有授权给地理围栏检测
- 用户禁止位置服务在设置里面
- 用户禁止 app 的后台刷新
- 用户开启了飞行模式

> [isMonitoringAvailableForClass:](https://developer.apple.com/documentation/corelocation/cllocationmanager/1423654-ismonitoringavailable) 判断设备是否支持地理围栏当用户没有授权给 app 使用地理围栏的时候，可以先注册地理围栏后授权来使用。当你不希望 app 在没有授权的时候使用地理围栏，可以在 **locationManager:didChangeAuthorizationStatus:** 这个方法里面去监听权限的改变和当你需要的时候删除地理围栏的监听。

### 使用

当需要使用地理围栏是，必需先使得用户授权。使用`CLLocationManager`来提醒用户授权，设置`CLLocationManager`的代理。

1. 调用`- (void)startMonitoringForRegion:(CLRegion *)region;`方法来监听需要检测的范围。

2. 当用户进入到监测的地理围栏时，会触发代理方法

   `-(void)locationManager:(CLLocationManager *)manager didEnterRegion:(CLRegion *)region);`

3. 当用户离开检测的地理围栏时，会触发代理方法

   `-(void)locationManager:(CLLocationManager *)manager`

   `didExitRegion:(CLRegion *)region;`

4. 当不需要检测这个地点时，需要调用 `- (void)stopMonitoringForRegion:(CLRegion *)region`方法来取消监听，因为监听是在 app 中是全局的，并不会随着 manager 的销毁而消失，并且 app 检测的地理围栏个数是有限制的，最大的数目是 20 个，所以当我们不需要监听时要及时的释放掉。

### 测试

原理研究清楚了，代码也写好了，要如何测试呢？

很简单，我们使用 xcode 新建 GPX 文件：

![](https://ws4.sinaimg.cn/large/006tNc79ly1g27yxi3x0vj30ka0emn20.jpg)

建好之后，将里面的 lat，lon 的值替换成你想 mock 的地理位置的经纬度：

```xml
<?xml version="1.0"?>
<gpx version="1.1" creator="Xcode">

    <!--
     Provide one or more waypoints containing a latitude/longitude pair. If you provide one
     waypoint, Xcode will simulate that specific location. If you provide multiple waypoints,
     Xcode will simulate a route visiting each waypoint.
     -->
    <wpt lat="31.230390" lon="121.473702">
        <name>shanghai</name>

        <!--
         Optionally provide a time element for each waypoint. Xcode will interpolate movement
         at a rate of speed based on the time elapsed between each waypoint. If you do not provide
         a time element, then Xcode will use a fixed rate of speed.

         Waypoints must be sorted by time in ascending order.
         -->
    </wpt>

</gpx>

```

建立好多个 mock 文件之后，我们就可以在 debug 时来切换自己想要 mock 的位置。

![](https://ws3.sinaimg.cn/large/006tNc79ly1g27z2m6j3sj30e60cbq69.jpg)

红色标记出的位置就是刚刚我们新建的 gpx 文件的名称，点击之后手机的位置就会变成 mock 的地理位置。

当然，用模拟器也可以 mock 坐标，只不过用文件的方式切换比较方便。

![](https://ws1.sinaimg.cn/large/006tNc79ly1g27z54bwngj30oo0cd46p.jpg)

模拟器的 mock 如上图所示。

当 app 被 kill 时，可以通过本地通知来检测进出地理围栏。

## 后续

关于坐标位置的坑点，后面在另一篇文章继续深究下。
