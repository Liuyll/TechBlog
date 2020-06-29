原理

native -> js

原生调用stringByEvaluatingJavaScriptFromString (类似js里的eval)执行js代码



js -> native

+ url加载Custom URL Scheme，原生拦截url做处理

  如在`uiwebview`里，任何请求都能在native层被拦截，此时可以定于这样的格式

  ```
  jsbridge://
  ```

  以这样开头的schema，不进行数据发送，而进行jsbridge逻辑处理。

   要注意的是，修改的url应该是新建的iframe的url

​		alert

​		confirm

​		也是可以拦截的

+ 通过native注入在webview上的全局对象来调用

  ```
  eg:window.JsBridge.open(...)
  ```

  





### webview

+ WebViewJavaScriptBridge

  javscriptcore

  使用：

  在web和native端都需要注册一个对象来保持[沟通](https://juejin.im/post/5cecd746e51d45778f076cac)

  

+ WVJBCallbacks

+ UIWebView





### eventmanager

#### DeviceEventEmitter ios

这个相当于一个事件车：

可以在A页面发送信息，在B页面接收信息

`RCTDeviceEventEmitter`

这是个底层通信事件触发器



#### NativeEventEmitter android



原生通过DeviceEventEmitter向js发送事件

