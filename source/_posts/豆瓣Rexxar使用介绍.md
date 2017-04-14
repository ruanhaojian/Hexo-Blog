---
title: 豆瓣Rexxar使用记录
date: 2016-12-22 16:58:53
tags: 
	- rexxar
	- hybrid
categories:
	- hybrid
    - 原创
---

> Rexxar 是一个针对移动端的混合开发框架。现在支持 Android 和 iOS 平台。rexxar-web 是 Rexxar 的 Web 端实现，提供配合 Rexxar Container 运行的基础工具。

<!--more-->

## 相关hybrid方案

* 1.在JS 中做一次URL跳转，然后在OC中拦截跳转。（这里分为UIWebView 和 WKWebView两种，去年因为还要兼容iOS 6，所以没办法只能采用UIWebView来做。）
* 2.利用WKWebView 的MessageHandler。
* 3.利用系统库JavaScriptCore，来做相互调用。（iOS 7推出的）
* 4.利用第三方库WebViewJavascriptBridge。JSBridge(Android: JsBridge，iOS：WebViewJavascriptBridge)。
* 5.利用第三方cordova库，以前叫PhoneGap。（这是一个库平台的库）
* 6.当下盛行的React Native。
* 7.阿里推出的weex(使用了vue框架)，现已捐赠Apache。

以下介绍的Rexxar使用的便是拦截的方案。

## Rexxar 简介

关于 Rexxar 的整体介绍，可以查看文档：[Rexxar 简介](http://lincode.github.io/Rexxar-OpenSource)。

关于 Rexxar iOS，可以访问：[https://github.com/douban/rexxar-ios](https://github.com/douban/rexxar-ios)。

关于 Rexxar Android，可以访问：[https://github.com/douban/rexxar-android](https://github.com/douban/rexxar-android)。

## 使用感受

关于rexxar的使用在博客还有github上都有详细介绍，用户可以先大致了解，这里再补充我在实际项目中使用所遇到的问题。

### iOS平台

> 采用拦截请求转发的方式，来达到js与native的交互。

1、 rexxar-container api root 设置问题
按照demo所设置的 douban://rexxar-container/api 这种写法，会报domain无法访问的错误。所以实际应该设置一个可访问的域名来作为container-api的接口地址。

2、rexxar的缓存机制
rexxar自己设定了一套路由缓存机制，会通过请求回来的路由映射到相应的地址，并且在默认开启缓存模式的时候，是先下载网页再打开页面。
这个时候我就遇到了不太满足我自己业务的功能：
* 如果网络地址不是一个网页资源，如：www.baidu.com这样的网站，它并不能正确下载，所以这时加载是错误的。
* 假如我加载的是.html结尾的地址，能够成功load出网页，但是如果你网站其余资源地址是相对路径的话，网页加载的也是有问题的，因为相对路径出来就是错误的。
* 地址路径的一些额外信息，要知道query和hash对于一个网页来说也是很重要的，开启缓存机制的话，这些信息都将会被丢弃。

我的解决方案：继承RexxarViewController，重写`_rxr_htmlURLWithUri`方法，使得每次都加载完整的网络地址。并且关闭缓存`[RXRConfig setCacheEnable:NO]`。

```ObjectiveC
- (NSURL *)_rxr_htmlURLWithUri:(NSURL *)uri htmlFileURL:(NSURL *)htmlFileURL
{
    // 当前网络url
    NSURL *currentUrl = nil;
    if (!htmlFileURL) {
        // 没有设置 htmlFileURL，则使用本地 html 文件或者服务器读取 html 文件。
        
        htmlFileURL = [[RXRRouteManager sharedInstance] remoteHtmlURLForURI:self.uri];
        currentUrl = [htmlFileURL copy];
        
        if ([RXRConfig isCacheEnable] && [htmlFileURL.path containsString: @".html"]) {
            // 如果缓存启用，尝试读取本地文件。如果没有本地文件（本地文件包括缓存，和资源文件夹），则从服务器读取。
            NSURL *localHtmlURL = [[RXRRouteManager sharedInstance] localHtmlURLForURI:self.uri];
            if (localHtmlURL) {
                htmlFileURL = localHtmlURL;
            }
        }
        
    }
    
    
    if (htmlFileURL.query.length != 0 && htmlFileURL.fragment.length != 0) {
        // 为了方便 escape 正确的 uri，做了下面的假设。之后放弃 iOS 7 后可以改用 `queryItem` 来实现。
        // 做个合理假设：html URL 中不应该有 query string 和 fragment。
        RXRWarnLog(@"local html 's format is not right! Url has query and fragment.");
    }
    
    // `absoluteString` 返回的是已经 escape 过的文本，这里先转换为原始文本。
    NSString *uriText = uri.absoluteString.stringByRemovingPercentEncoding;
    // 把 uri 的原始文本所有内容全部 escape。
    NSCharacterSet *set = [NSCharacterSet characterSetWithCharactersInString:@""];
    uriText = [uriText stringByAddingPercentEncodingWithAllowedCharacters:set];
    
    NSURL *realLoadUrl = nil;
    // 拼接完整url
    if([htmlFileURL isFileURL]){
        NSString *query = currentUrl.query ? [NSString stringWithFormat:@"?%@&uri=%@",currentUrl.query, uriText] : [NSString stringWithFormat:@"?uri=%@", uriText];
        NSString *fragment = currentUrl.fragment ? [NSString stringWithFormat:@"#%@",currentUrl.fragment] : @"";
        
        realLoadUrl = [NSURL URLWithString:[NSString stringWithFormat:@"%@%@%@",htmlFileURL.absoluteString, query, fragment]];
    }else{
        NSString *scheme = htmlFileURL.scheme;
        NSString *host = htmlFileURL.host;
        NSString *port = htmlFileURL.port ? [NSString stringWithFormat:@":%@",htmlFileURL.port] : @"";
        NSString *path = htmlFileURL.path ? htmlFileURL.path : @"/";
        NSString *query = htmlFileURL.query ? [NSString stringWithFormat:@"?%@&uri=%@",htmlFileURL.query, uriText] : [NSString stringWithFormat:@"?uri=%@", uriText];
        NSString *fragment = htmlFileURL.fragment ? [NSString stringWithFormat:@"#%@",htmlFileURL.fragment] : @"";
        
        realLoadUrl = [NSURL URLWithString:[NSString stringWithFormat:@"%@://%@%@%@%@%@", scheme, host, port, path, query, fragment]];

    }
    
    return  realLoadUrl;
}
```

3、UIWebView自带缓存问题
可通过路由链接加时间戳解决，获取修改rexxar的源码，动态拼接时间戳。

### Android平台

> 同样是采用请求拦截方式

1、如果不能拦截到请求，可以查看是否将api 请求的域名添加到拦截列表中
`ResourceProxy.getInstance().addProxyHosts(List<>() hosts);`

2、添加headers信息
为请求添加在headers所需要的状态信息，android出现web自己添加的部分会莫名其妙的不见了，所以headers的信息在android只能由android全部设置，而iOS这里可由web控制一部分header信息。

### Web端(rexxar-web)

1、前端项目不限制使用何种技术框架
所以你可以使用react、vue、angualr等等。按照常规web项目开发模式即可。


## 总结

rexxar封装了路由以及缓存的策略，方便我们开发web与native的交互方式。但还是有以上几个方面考虑的不够，需要继续完善。

