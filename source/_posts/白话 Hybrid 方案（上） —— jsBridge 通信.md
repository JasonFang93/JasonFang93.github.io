title: 白话 Hybrid 方案（上） —— jsBridge 通信
category: 技术
date: 2018-05-06 00:00:00
tags: [Mobile, Hybrid, jsBridge]
photos: [https://ws1.sinaimg.cn/large/006cGJIjly1fiza4vwhtkj30b40640sz.jpg]
---

## 写在前面
从 PhoneGap 开始，发展到现在，Hybrid 方案已经变得相当成熟了，在很多应用中都能见到它的身影，而最具代表性的杀手级应用微信的出现，将 Hybrid 方案推向了一个高潮。Hybrid 应用简单的可以理解为一个可以调用 Native 功能的web app，其中的核心就是借助 jsBridge 实现 js 与 Native 间的相互通信。为什么说相互通信是核心呢，js 与 Native 之间又是如何通信的呢？
<!-- more -->
## 背景
我们先来描述一个具体的 Hybrid 项目场景：在客户端中打开了一个页面 A，在 A 页面通过新开一个 view 的方式打开页面 B（表明当前项目不是单页应用），在页面 B 中查看当前手机的网络状态并对客户端导航条的点击事件进行监听，点击导航条上的标题，最后点击返回按钮，回到页面 A；在“新开 view 的方式打开页面”、“查看网络状态”、“监听导航条点击事件”这一系列的操作中，都是借助 Native 来完成的，换句话说就是调用了 Native 的接口来实现的。这些操作（接口）可以分成两类：一是普通调用，例如“查看网络状态”；另一类是事件监听，例如“监听导航条点击事件”。

<img src="https://ws1.sinaimg.cn/mw690/c4b5f11bly1fr23db8doog20go0wo4qr.gif" style="width: 320px; height: 568px;" />

那么，接下来，笔者将结合背景和上文中提出的两个问题，一一来填坑。

## 相互通信
普通调用、事件监听本质上和 js 与 Native 间的相互通信密不可分。无论是查看网络状态还是监听导航条点击事件，都是在 js 层处理的，但操作实际是 Native 控制的，换句话说，js 告诉了 Native 具体的操作，而当 Native 完成这些操作后，需要将操作的结果反馈给 js，这就是 js 与 Native 之间的相互通信。
上文背景中提到的两类操作，虽然本质上都是 js 和 Native 的相互通信，但在处理上还是有一定区别的，虽然由于篇幅限制，具体的区别笔者将会在下部 jsBridge 具体实现部分来讲述，但可以来聊聊 jsBridge 具体是什么，在通信中扮演怎么的角色。
![jsBridge](https://wx1.sinaimg.cn/mw690/71c50075ly1feyxr21wdkj20n905n74c.jpg)
jsBridge，顾名思义，js 实现的桥，js 与 Native 的通信必须借助它来完成。笔者理解的 jsBridge 是 Native 注入执行一段 js 代码，并在 js 上下文挂载的一个全局对象（和方法），即 `window.bridge`。

``` javasrcipt
bridge:
{
    invoke: function() {
        // js 发送消息给 Native，不同环境，通信方式实现不同
    },
    // 根据当前 Native 环境分别挂载对应的方法
    __oc2js__: function() {
        // 接收并处理 OC 发给 js 的消息
    },
    __java2js__: function() {
        // 接收并处理 Java 发给 js 的消息
    }
}
```

Hybrid 方案是基于 WebView 的，js 执行在 WebView 的 Webkit 引擎中。
## js 与 OC 间相互通信
### js -> OC
js 调用 Native，给 OC 发消息的方式主要有`使用 Ajax 发起请求`、`注入 API`和`拦截 URL SCHEME`三种方式。笔者主要来聊一聊第一种方式，后两种可以参考附录。
使用 Ajax 发起同域请求，将调用参数放在 head 或 body 中，Native 拦截请求，如果命中规则，则调用 Native 对应的方法，否则将其作为视为正常的 Ajax 请求。例如当查看网络状态时，js 发送给 Native 的 handleName 为 `network.getType`。

``` javasrcipt
var sendMessageQueue = [{
    handlerName: handlerName,
    data: data,
    // callbackId 作为回调的唯一标识，在返回调用结果给 js 时，根据 callbackId 调用对应的回调方法
    callbackId: id++ + (Math.random() + '').replace(/\D/g, "")
}];
function() {
    // Re-using the XHR improves exec() performance by about 10%.
    var xhr = xhr || new XMLHttpRequest();
    // Changing this to a GET will make the XHR reach the URIProtocol on 4.2.
    // For some reason it still doesn't work though...
    // Add a timestamp to the query param to prevent caching.
    xhr.open('HEAD', requestPath + (+new Date()));
    xhr.setRequestHeader('data', encodeURI(JSON.stringify(sendMessageQueue)));
    //INFO 'Send message over xhr:', JSON.stringify(sendMessageQueue)
    xhr.send(null);
};
```

### OC -> js
OC 调用 js 方法，回传数据给 js 的方式相对比较简单，通过 Native 通过的接口调用 js 代码。调用的 js 方法即是挂载在 bridge 上的 `__oc2js__` 方法。

``` Objective-C
[uiWebview stringByEvaluatingJavaScriptFromString:jsFunction];
```

## js 与 Java 间相互通信
### js -> Java
js 调用 Native，给 Java 发消息的方式主要有`拦截通信通道`、`注入 API`两种方式。笔者主要来聊一聊第一种方式，后一种同样可以参考附录。
借助 WebView 的 `setWebChromeClient` 方法设置 `WebChromeClient` 对象，对象包含 `onJsAlert`，`onJsConfirm`，`onJsPrompt` 三个方法，与 js 的 `alert`、`confirm` 和 `prompt` 对应，但由于前两个 js 方法使用场景相比 `prompt` 更多，所以一般 js 与 Java 通信使用 `prompt`。

``` javasrcipt
function(sendMessageQueue) {
    //promptName 是使用prompt传值的标识
    //INFO 'Send message through prompt:', sendMessageQueue
    prompt(JSON.stringify(sendMessageQueue), promptName);
}
```

### Java -> js
Java 调用 js 方法，回传数据给 js 的方式与 OC 一样，也相对比较简单，通过 Native 通过的接口调用 js 代码。调用的 js 方法即是挂载在 bridge 上的 `__java2js__` 方法。

``` Java
webView.evaluateJavascript(jsFunction, new ValueCallback<String>() {
    @Override
    public void onReceiveValue(String value) {

    }
});
```
或

``` Java
WebView.loadUrl("javascript:" + jsFunction);
```

## 总结
在这篇文章中，笔者主要以一个具体的 Hybrid 调用实例作为切入点，聊了聊 js 与 Native 间是如何通过 jsBridge 实现通信的；关于 jsBridge 具体的加载流程和实现，且听我下回分解。
附录：[移动混合开发中的 JSBridge](https://blog.ymfe.org/%E6%B7%B7%E5%90%88%E5%BC%80%E5%8F%91%E4%B8%AD%E7%9A%84JSBridge/)
