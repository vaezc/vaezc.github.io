---
title: iOS 网络请求耗时统计
published: 2019-04-04
category: 技术人生
tags: ["iOS"]
image: iOS.jpg
---

前言：最近的研发中，老大说需要监测一下每个接口的连接速度，针对一些接口优化一下。所以就需要 hook 到每个网络请求来进行监测了。通过网上一些查阅资料，初步将目标定位到`NSURLProtocol`这个类中了。

## NSURLProtocol

`NSRULProtocol` 是`URL loadin System`中的一部分，如下图
![](https://i.loli.net/2019/04/03/5ca4b8c97c82c.png)

`NSURLProtocol` 是个抽象类，不能直接使用它的实例。需要我们声明一个子类来使用它。当网络请求发生时，系统会创建合适的协议对象来处理这些请求。我们自定义的子类需要在 app 启动时调用 `registerClass`方法使得系统知道我们的协议，才能使得我们自定义的协议来处理这些网络请求。

`NSURLProtocol`作为上层接口，它的作用就是可以检测到每个网络请求，但是由于它是属于 `URL Loading System` 体系的，所以支持的协议有限，只支持 `FTP， HTTP， HTTPS` 等几个应用层协议。具体来说`NSURLProtocol`属于一个中间层，当我们发起网络请求时，我们注册的子类的对象会接受到请求，接受到请求后再做一些处理，然后再转发请求。这个时候我们可以针对请求来记录我们需要的数据或或者对请求做一些修改。总的来说就是你不必修改在网络调用上的其他部分，就可以改变 URL 加载行为的全部细节，使得代码结构更加清爽健壮。

[NSHipster](https://nshipster.cn/nsurlprotocol/)列举了一些可以使用`NSURLProtocol`做的事情：

1. [拦截图片加载请求，转为本地文件加载](https://stackoverflow.com/questions/5572258/ios-webview-remote-html-with-local-image-files)
2. [为了测试对`HTTP`返回内容进行`mock`和`stub](https://github.com/AliSoftware/OHHTTPStubs)
3. 对发出请求的`header`进行格式化
4. 对发出的媒体请求进行签名
5. 创建本地代理服务，用于数据变化时对`URL`请求的更改
6. 故意制造畸形或者非法返回的数据来测试程序的鲁棒性
7. 过滤请求和返回中的敏感信息
8. 在既有协议举出上完成对 `NSURLconnection`的实现且与原逻辑不产生矛盾

针对这次的需求，只需要拦截住工程里面发起的 api 的网络请求，然后统计出每个请求的耗时即可。

具体的看下实现。

我们首先定义`NSURLProtocol`的一个子类:

```objectivec
static NSString * const MYHTTPHandledIdentifier = @"MYHTTPHandledIdentifier";

@interface MYHTTPProtocol : NSURLProtocol

...

@end
```

## canInitWithRequest

```objectivec
+ (BOOL)canInitWithRequest:(NSURLRequest *)request {
  if (![request.URL.scheme isEqualToString:@"http"] &&
        ![request.URL.scheme isEqualToString:@"https"]) {
        return NO;
    }

    if ([NSURLProtocol propertyForKey:MYHTTPHandledIdentifier inRequest:request] ) {
        return NO;
    }
    return YES;
}
```

每一个网络请求，都会走到上述方法，我们可以拿到`request`的实例，这个函数的返回值代表是否处理这个请求。上述方法里面代表着我们只是监听`http`和`https`协议的方法。下面一个`if`语句后面再讲，可以先不看。虽然说我们在这个方法里面拿到了 request 对象，但是我们最好不要在这个方法里面操作对象，因为可能存在语义上的问题。这个方法只是筛选哪些网络请求需要被拦截。

## canonicalRequestForRequest

```objectivec
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request {

    NSMutableURLRequest *mutableReqeust = [request mutableCopy];
    [NSURLProtocol setProperty:@YES
                        forKey:MYHTTPHandledIdentifier
                     inRequest:mutableReqeust];
    return [mutableReqeust copy];
}
```

这个方法是针对我们筛选出来网络请求进行重新构造，`NSURLProtocol`提供了一些方法让我们来操作 request 的 metadata，如代码中所示，设置了一个 bool 值为 yes 的特殊属性，将这个属性添加到 request 中去了。因为在接下来我们还要使用这个 request 发起请求，所以需要添加一个标记，表示这个请求已经被处理过了不用在`canInitWithRequest`处理了，在`canInitWithRequest`方法中我们对 `request`中设置了特殊属性的请求不拦截，不然的话会使得这个请求在`canInitWithRequest`和`canonicalRequestForRequest`形成死循环。

## startLoading

```objectivec
- (void)startLoading {
 NSURLSession *session = [NSURLSession sessionWithConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration] delegate:self delegateQueue:nil];
 NSURLSessionDataTask *task = [session dataTaskWithRequest:self.request];
    self.dataTask  = task;
 [self.dataTask resume];
}
```

这个是开始加载网络请求的方法，我们可以在这个方法里对协议对象持有的`request`对象进行转发。我们可以使用`NSURLSession`，`NSURLConnection`或者第三方的网络库，这里我使用了`NSURLsession`来进行转发这个请求。我们可以在这个请求里面记录当前的时间。后面再网络请求结束的时候再一次记录时间，就可以计算出网络加载的耗时了。

## stopLoading

```objectivec
- (void)stopLoading {
    [self.dataTask cancel];
    self.dataTask           = nil;
    // 解析 response，流量统计，网络耗时等
}
```

## client

每个  `NSURLProtocol`  的子类实例都有一个  `client`  属性，该属性对 URL 加载系统进行相关操作。它不是  `NSURLConnection`，但看起来和一个实现了  `NSURLConnectionDelegate`  协议的对象非常相似 。你需要在合适的时期调用`client`方法，将数据传回给`client`就行。`client`相当于这个网络的发起者，我们需要将网络中加载的情况返回给调用者。

```objectivec
#pragma mark - NSURLSessionTaskDelegate

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error {
    if (!error) {
        [self.client URLProtocolDidFinishLoading:self];
    } else if ([error.domain isEqualToString:NSURLErrorDomain] && error.code == NSURLErrorCancelled) {
    } else {
        [self.client URLProtocol:self didFailWithError:error];
    }
    self.dataTask = nil;
}

#pragma mark - NSURLSessionDataDelegate

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data {
    [self.client URLProtocol:self didLoadData:data];
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveResponse:(NSURLResponse *)response completionHandler:(void (^)(NSURLSessionResponseDisposition))completionHandler {
    [[self client] URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageAllowed];
    completionHandler(NSURLSessionResponseAllow);
    self.response = response;
}

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task willPerformHTTPRedirection:(NSHTTPURLResponse *)response newRequest:(NSURLRequest *)request completionHandler:(void (^)(NSURLRequest * _Nullable))completionHandler {
    if (response != nil){
        self.response = response;
        [[self client] URLProtocol:self wasRedirectedToRequest:request redirectResponse:response];
    }
}
```

## registerClass

当然写完这些还是不够的，需要在 app 启动的地方注册我们写好的`NSURLProtocol`的子类:

```objectivec
[NSURLProtocol registerClass:[MyURLProtocol class]];
```

这相当于向 url 加载系统进行注册。URL protocol 会被以注册顺序的反序访问，所以当在  `-application:didFinishLoadingWithOptions:`  方法中调用  `[NSURLProtocol registerClass:[MyURLProtocol class]];`  时，你自己写的 protocol 比其他内建的 protocol 拥有更高的优先级。

## 问题

### URL 无法拦截

基本的使用也就上述讲的那么多了，但是在使用的时候发现即使我注册了`NSURLProtocol`的子类，我还是无法拦截自己的网络请求。但是一些第三方库的请求拦截住了。在网上查找资料发现：

如果说需要支持自定义的`NSURLProtocol`，需要将自定义的`NSURLProtocol`子类赋给`NSURLSessionConfiguration`的`protocolClassess`属性。所以，如果需要`NSURLProtocol`来截获`NSURLSession`发出的请求，需要每一个`NSURLSession`在创建时配置的`NSURLSessionConfiguration`类的`protocolClasses`属性附上自定义的`NSURLProtocol`。

查看`AFNetworking`的代码发现，它创建的`session`关联的`NSURLSessionConfiguration`为`defaultSessionConfiguration` 。所以我们在不能直接修改第三方的源码的时候只能使用`method swizzling`的方法去 hook 住`defaultSessionConfiguration` 的方法，然后将我们自定义的`NSURLProtocol`的子类注册到 url 加载系统里面去了。

```objc
@implementation NSURLSessionConfiguration (UrlProtocolSwizzling)

+(void)load{

 [self jr_swizzleClassMethod:@selector(defaultSessionConfiguration) withClassMethod:@selector(my_defaultSessionConfiguration) error:nil];



 [NSURLProtocol registerClass:[TrackNetWorkURLProtocol class]];
}

+(NSURLSessionConfiguration *)my_defaultSessionConfiguration{
 NSURLSessionConfiguration *configuration = [self my_defaultSessionConfiguration];
 NSArray *protocolClasses = @[[TrackNetWorkURLProtocol class]];
 configuration.protocolClasses = protocolClasses;
 return configuration;
}

```

### POST 请求 body 为空

通过拦截请求时，在`stoploading`方法里面想对请求做一些操作时发现有些`post`请求的`body`为空，这个时候发现网上也有人碰到这个问题，初步判断是苹果的 bug，这个时候我们只能通过将`HTTPBodyStream`中得到的数据写入到`HTTPBody`中了。写一个`NSURLRequest`的分类，将`HTTPBodyStream`的数据写入到`HTTPBody`中，在`canonicalRequestForRequest`方法中调用写入操作，然后返回这个`request`。

```objectivec

@implementation NSURLRequest (RequestIncludeBody)

-(NSURLRequest *)getPostRequestIncludeBody{
 return [[self getMutablePostRequestIncludeBody] copy];
}

-(NSMutableURLRequest *)getMutablePostRequestIncludeBody{
 NSMutableURLRequest *request = [self mutableCopy];
 if ([self.HTTPMethod isEqualToString:@"POST"]) {
  if (!self.HTTPBody) {
   NSInteger maxLength = 1024;
   uint8_t d[maxLength];
   NSInputStream *stream = self.HTTPBodyStream;
   NSMutableData *data = [[NSMutableData alloc] init];
   [stream open];
   BOOL endOfStreamReached = NO;
   while (!endOfStreamReached) {
    NSInteger bytesRead = [stream read:d maxLength:maxLength];
    if (bytesRead == 0) { //文件读取到最后
     endOfStreamReached = YES;
    } else if (bytesRead == -1) { //文件读取错误
     endOfStreamReached = YES;
    } else if (stream.streamError == nil) {
     [data appendBytes:(void *)d length:bytesRead];
    }
   }
   request.HTTPBody = [data copy];
   [stream close];
  }
 }
 return request;
}
@end
```

## 无法拦截 wkwebview 中的请求

无论是  `NSURLProtocol`、`NSURLConnection`  还是  `NSURLSession`  都会走底层的 `socket`，但是  `WKWebView`  可能由于基于 `WebKit`，并不会执行 `C socket` 相关的函数对 `HTTP` 请求进行处理。所以如果要拦截`WKWebview`中的请求只能通过`WKWebview`对应的代理方法进行处理了。

> [GitHub - aozhimin/iOS-Monitor-Wedjat（华狄特）开发过程的调研和整理](https://github.com/aozhimin/iOS-Monitor-Platform)Platform: iOS 性能监控 SDK ——
>
> [NSURLProtocol 拦截 NSURLSession 请求时 body 丢失问题解决方案探讨 - 个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000013991357)
>
> [NSURLProtocol 无法截获 NSURLSession 解决方案 钟武的技术博客](https://zhongwuzw.github.io/2016/08/31/NSURLProtocol%E6%97%A0%E6%B3%95%E6%88%AA%E8%8E%B7NSURLSession%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/)
>
> [https://zeeyang.com/2017/09/09/debug-tool-base-on-NSURLProtocol/](https://zeeyang.com/2017/09/09/debug-tool-base-on-NSURLProtocol/)
>
> [https://developer.apple.com/documentation/foundation/url_loading_system?language=objc](https://developer.apple.com/documentation/foundation/url_loading_system?language=objc)
>
> [NSURLProtocol - NSHipster](https://nshipster.cn/nsurlprotocol/)
>
> [GitHub - mattt/NSEtcHosts: /etc/hosts with NSURLProtocol](https://github.com/mattt/NSEtcHosts)
>
> [https://blog.newrelic.com/engineering/right-way-to-swizzle/](https://blog.newrelic.com/engineering/right-way-to-swizzle/)
