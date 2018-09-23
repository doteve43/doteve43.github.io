---
layout: post
title: 'wkwebview基础总结'
subtitle: '顺便练习一下markdown'
date: 2018-09-23
categories: 技术
cover: ''
tags: iOS开发
---
最近有个webapp的项目交到我的手上，说是有些问题，比如某些iPhone无法打开视频，启动有时比较慢等问题。其实这个项目我客户端处理的只是展示一个webview，显示的内容都是前端控制的，遇到这些问题我只能从webview下手。所幸苹果公司在iOS8之后推出wkwebview可以替换现在项目使用的uiwebview。

>## 使用效果

替换了wkwebview之后，之前所说的怪问题都解决了，更好的是，内存占用比之前减少了70%左右，顺便添加了左右划动手势和进度条显示，更加方便操作。

>## 如何从uiwebview过渡到wkwebview

其实基本使用方法是一样的，主要实现wkwebview几个delegate

 * WKNavigationDelegate

WKNavigationDelegate主要负责跳转、加载处理等操作。

需要实现的方法为：

```
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler;

- (void)webView:(WKWebView *)webView decidePolicyForNavigationResponse:(WKNavigationResponse *)navigationResponse decisionHandler:(void (^)(WKNavigationResponsePolicy))decisionHandler;

- (void)webView:(WKWebView *)webView didStartProvisionalNavigation:(null_unspecified WKNavigation *)navigation;


- (void)webView:(WKWebView *)webView didReceiveServerRedirectForProvisionalNavigation:(null_unspecified WKNavigation *)navigation;


- (void)webView:(WKWebView *)webView didFailProvisionalNavigation:(null_unspecified WKNavigation *)navigation withError:(NSError *)error;


- (void)webView:(WKWebView *)webView didCommitNavigation:(null_unspecified WKNavigation *)navigation;


- (void)webView:(WKWebView *)webView didFinishNavigation:(null_unspecified WKNavigation *)navigation;


- (void)webView:(WKWebView *)webView didFailNavigation:(null_unspecified WKNavigation *)navigation withError:(NSError *)error;


- (void)webView:(WKWebView *)webView didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * _Nullable credential))completionHandler;


- (void)webViewWebContentProcessDidTerminate:(WKWebView *)webView API_AVAILABLE(macosx(10.11), ios(9.0));

```

其中比较重要的是decidePolicyForNavigationResponse这个方法，这个方法会在wkwebview收到网页响应之后调用，在这个方法可以处理是否对此网页进行跳转。
公司项目中遇到的大部分问题都可以在这个回调中进行处理，大致方法为：

`navigationAction.request.URL.absoluteString`通过这个方法可以获取到需要拦截的url，如果需要进行拦截就调用`decisionHandler(WKNavigationActionPolicyCancel);`,如果无需处理就调用`decisionHandler(WKNavigationActionPolicyAllow);`。



* WKUIDelegate

WKUIDelegate主要是处理js回调的确认框，警告框等。

```
#pragma mark -WKUIDelegate
- (WKWebView *)webView:(WKWebView *)webView createWebViewWithConfiguration:(WKWebViewConfiguration *)configuration forNavigationAction:(WKNavigationAction *)navigationAction windowFeatures:(WKWindowFeatures *)windowFeatures
{
    //创建一个新的webView-createWebViewWithConfiguration
    return [[WKWebView alloc] init];
}

// 输入框
- (void)webView:(WKWebView *)webView runJavaScriptTextInputPanelWithPrompt:(NSString *)prompt defaultText:(nullable NSString *)defaultText initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(NSString * __nullable result))completionHandler
{
    completionHandler(YES);

}
// 确认框
- (void)webView:(WKWebView *)webView runJavaScriptConfirmPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(BOOL result))completionHandler
{
    completionHandler(YES);

}
// 警告框
- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(void))completionHandler
{
    completionHandler();
}

```
公司项目没有使用到这个回调的方法， 所以具体使用方法不是十分了解，具体使用方法可以去[苹果官方文档](https://developer.apple.com/documentation/webkit/wkuidelegate?language=objc)进行查询。


 * WKScriptMessageHandler

WKScriptMessageHandler是处理OC和js交互逻辑。

>OC-->js

OC调用js的方法：

```
[webView evaluateJavaScript:@"XXXX" completionHandler:^(id _Nullable result, NSError * _Nullable error) {
        NSLog(@"%@",result);
    }];
```

其中XXXX就是js方法名，completionHandler会回调的block。

>js-->OC

js调用OC的方法：

1)首先js和OC要先定义一个handler名，在wkwebview初始化的时候设置，js通过这个名字来发送消息，OC也通过这个名字来判断回调过来是哪一个方法。

```
_scriptHandlerName = @"XXXX";
       WKWebViewConfiguration *config = [[WKWebViewConfiguration alloc]init];
       WKUserContentController *userContentController = [[WKUserContentController alloc] init];

       [userContentController addScriptMessageHandler:self name:_scriptHandlerName];
       config.userContentController = userContentController;
       CGFloat mainWidth = [UIScreen mainScreen].bounds.size.width;
       CGFloat mainHeight = [UIScreen mainScreen].bounds.size.height;
       CGRect statusRect = [[UIApplication sharedApplication] statusBarFrame];
       _webView = [[WKWebView alloc] initWithFrame:CGRectMake(0, statusRect.size.height-1, mainWidth, mainHeight-statusRect.size.height) configuration:config];
```

这里是简单初始化wkwebview的方法，XXXX就是handler名，需要将这个handler名告诉给前端开发。

2）前端js调用方法为：

`window.webkit.messageHandlers.XXXX.postMessage(xx);`

XXXX也就是上述说的handler名，postMessage携带的变量就是js需要回调给OC的内容。

3）OC处理回调

OC需要先实现delegate的方法

```

- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message {
    //收到js回调
}
```
message.body 就是js回调回来的内容。拿到内调内容就可以进行处理啦。

>## 遇到的问题

1.在调用支付宝和微信的时候发现wkwebview无法拉起微信或者支付宝app。之前的uiwebview通过alipay://或者weixin://可以直接打开客户端，但是kwwebview似乎只是单纯的加载这个url（原因不明），所以需要手动去加载。
### 解决方法
在decidePolicyForNavigationResponse 回调中 拦截url，比如拦截alipay://开头的url就使用`[UIApplication sharedApplication] openurl`方法打开这个url，这样就可以解决问题了。

2.测试发现，使用h5支付宝支付完成之后返回项目页面，再点返回（或者手势右划）又会返回支付宝完成界面，再右划会返回输密码的界面，体验上有问题。
### 解决方法
其实思路和上面的一样，也是拦截url，当界面是支付宝完成的url时（非支付时自动跳转），就直接返回

`decisionHandler(WKNavigationActionPolicyCancel);`

就可以解决问题。

3.js在发消息给oc的时候报错，undefined type error
### 解决方法
问前端说什么都没改，但是第二天自动就好了，原因不明。

4.在调试的时候遇到过白屏，比较严重，特别是从微信支付回调回项目中，三次基本上就会出现白屏，而且无论做什么操作（比如手势返回到上个界面），也会出现白屏。
### 解决方法
这个问题在第二天就没有再出现了，原因也不明。我怀疑是本地缓存的问题，所以我找了一些清除缓存的方法，但是这个现象没有再出现，所以解决方法需要等待验证。

```
//基于ios9，开发过程中遇到过微信返回游戏有概率出现白屏，无日志，没有走到任何回调，原因不明，此方法能否解决问题不明，需待测试
-(void) cleanCache {
    // 清除所有
    //NSSet *websiteDataTypes = [WKWebsiteDataStore allWebsiteDataTypes];
    NSSet *websiteDataTypes= [NSSet setWithArray:@[

            WKWebsiteDataTypeDiskCache,

            WKWebsiteDataTypeOfflineWebApplicationCache,

            WKWebsiteDataTypeMemoryCache,

            WKWebsiteDataTypeLocalStorage,

            //WKWebsiteDataTypeCookies,

            //WKWebsiteDataTypeSessionStorage,

            //WKWebsiteDataTypeIndexedDBDatabases,

            //WKWebsiteDataTypeWebSQLDatabases

    ]];

    //// Date from

    NSDate *dateFrom = [NSDate dateWithTimeIntervalSince1970:0];

    //// Execute

    [[WKWebsiteDataStore defaultDataStore] removeDataOfTypes:websiteDataTypes modifiedSince:dateFrom completionHandler:^{

        // Done
        NSLog(@"清楚缓存完毕");

    }];
}
```
