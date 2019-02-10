---
WebView
---

#### 目录

1. 思维导图
2. WebView 的基本使用
   - WebView
   - WebSettings
   - WebViewClient
   - WebChromeClient
3. WebView 与 JS 交互
   - Android 去调用 JS 代码
   - JS 调用 Android 代码
4. WebView 常见问题汇总
5. WebView 优化
6. 参考

#### 思维导图

![](https://github.com/Omooo/Android-Notes/blob/master/images/WebView.png?raw=true)

#### 基本使用

WebView 是一个基于 webkit 引擎，展示 web 页面的空间。WebView 在低版本和高版本采用了不同的 webkit 内核版本，4.4 ( API 19 ) 之后直接使用了 Chrome。

##### WebView 类

```java
    /**
     * Back 键后退网页
     * 如果又重写了 onBackPressed 方法，只会回调 onKeyDown
     */
    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        if (keyCode == KeyEvent.KEYCODE_BACK && mWebView.canGoBack()) {
            mWebView.goBack();
            return true;
        }
        return super.onKeyDown(keyCode, event);
    }
	mWebView.onPasue();
    //清除缓存数据
    mWebView.clearCache(true);	//清除缓存
    mWebView.clearHistory();	//清除浏览记录
    mWebView.clearFormData();	//清除自动填充的表单数据
```

##### WebSetting 类

对 WebView 进行配置和管理。

```java
    private void setWebViewSettings(WebView webView){
        WebSettings webSettings=webView.getSettings();
        webSettings.setJavaScriptEnabled(true); //支持 JS
        webSettings.setJavaScriptCanOpenWindowsAutomatically(true); //支持通过 JS 打开新的窗口
        //设置自适应屏幕
        webSettings.setUseWideViewPort(true);
        webSettings.setLoadWithOverviewMode(true);

        webSettings.setLoadsImagesAutomatically(true);  //设置自动加载图片
        webSettings.setCacheMode(WebSettings.LOAD_NO_CACHE);    //不使用缓存
		//...
    }
```

##### WebViewClient 类

处理各种通知和请求事件等等。

```java
//在当前 WebView 打开页面，而不是系统浏览器
//如果不需要转发处理，只需要传递一个 WebViewClent 实例，根本不需要重写 shouldOverrideUrlLoading 方法     
public class MyWebViewClient extends WebViewClient {

    private Context mContext;

    public MyWebViewClient(Context context) {
        mContext = context;
    }

    @Override
    public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
        //当前 WebView 处理
        if (request.getUrl().getHost().equals("https://www.example.com")) {
            return false;
        }
        //如果需要转发处理
        mContext.startActivity(new Intent(Intent.ACTION_VIEW, request.getUrl()));
        return true;
    }

    @Override
    public void onReceivedError(WebView view, WebResourceRequest request, WebResourceError error) {
        switch (error.getErrorCode()){
            case WebViewClient.ERROR_CONNECT:   //连接失败
                view.loadUrl("file:///android_asset/error.html");
                break;
        }
    }

    @Override
    public void onReceivedSslError(WebView view, SslErrorHandler handler, SslError error) {
        handler.proceed();  //等待证书响应 
        //handler.cancel();   //挂起连接 默认行为
    }
}

mWebView.setWebViewClient(new MyWebViewClient(MainActivity.this));
```

##### WebChromeClient 类

辅助 WebView 处理 JS 的对话框、网站标题等等。

```java
public class MyWebChromeClient extends WebChromeClient {

    /**
     * 网页加载进度
     */
    @Override
    public void onProgressChanged(WebView view, int newProgress) {
        super.onProgressChanged(view, newProgress);
    }

    /**
     * 网页标题加载完毕回调
     */
    @Override
    public void onReceivedTitle(WebView view, String title) {
        super.onReceivedTitle(view, title);
    }

    /**
     * 拦截输入框
     */
    @Override
    public boolean onJsPrompt(WebView view, String url, String message, String defaultValue, JsPromptResult result) {
        return super.onJsPrompt(view, url, message, defaultValue, result);
    }

    /**
     * 拦截确认框
     */
    @Override
    public boolean onJsConfirm(WebView view, String url, String message, JsResult result) {
        return super.onJsConfirm(view, url, message, result);
    }

    /**
     * 拦截弹框
     */
    @Override
    public boolean onJsAlert(WebView view, String url, String message, JsResult result) {
        return super.onJsAlert(view, url, message, result);
    }
}


mWebView.setWebChromeClient(new MyWebChromeClient());
```

#### WebView 与 JS 交互

##### Android 调用 JS 代码

- webView.loadUrl(url)
- webView.evaluateJavascript()

首先选准备一个静态文件：

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Title</title>
    <script>
   function callJS(){
      alert("Android调用了 JS 的 callJS() 方法");
   }
    </script>
    <p3>
        WebView 与 JS 交互！
    </p3>
</head>
</html>
```

第一种方式：loadUrl()

```java
        mWebView.loadUrl("javascript:callJS()");
        mWebView.setWebChromeClient(new WebChromeClient(){
            @Override
            public boolean onJsAlert(WebView view, String url, String message, final JsResult result) {
                AlertDialog dialog=new AlertDialog.Builder(WebViewContactActivity.this)
                        .setTitle("Title")
                        .setPositiveButton("确认", new DialogInterface.OnClickListener() {
                            @Override
                            public void onClick(DialogInterface dialog, int which) {
                                result.confirm();
                            }
                        })
                        .setCancelable(false)
                        .setMessage(message)
                        .create();
                dialog.show();
                return true;
            }
        });
```

可以看到，WebView 只是载体，内容的渲染还的通过 WebChromeClient 承载。

第二种方式：evaluateJavascript()

```java
        mWebView.evaluateJavascript("javascript:callJS()", new ValueCallback<String>() {
            @Override
            public void onReceiveValue(String value) {
                //JS 返回的结果
                Toast.makeText(WebViewContactActivity.this, "value " + value, Toast.LENGTH_SHORT).show();
            }
        });
```

只是把上面的 loadUrl 换成 evaluateJavascript 方法而已。但是这种方法比第一种方式效率高，因为该方法的执行不会使页面刷新。

两种方法的对比：

| 调用方式            | 优点     | 缺点                       | 使用场景                           |
| ------------------- | -------- | -------------------------- | ---------------------------------- |
| loadUrl             | 方便简洁 | 效率低                     | 不需要获取返回值，对性能要求较低时 |
| evaluatedJavascript | 效率高   | 向下兼容性差（ API > 19 ） | API > 19                           |

当然也可以通过 Build.VERSION 来进行判断执行。

##### JS 调用 Android 代码

- 通过 WebView.addJavascriptInterface 进行对象映射
- 通过 WebViewClient.shouldOverrideUrlLoading 方法回调拦截 url
- 通过 WebChromeClient 的 onJsAlert、onJsConfirm、onJsPrompt 方法回调拦截 JS 对话框 alert、confirm、prompt 消息

第一种方式：WebView.addJavascriptInterface 进行对象映射

首先先准备好资源文件，用于模拟 WebView 加载的网页：

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Demo</title>
    <script>
         function callAndroid(){
            test.hello("js调用了android中的hello方法");
         }
      </script>
</head>
<body>
<p2>JS 调用 Android 方法</p2>
<button type="button" id="button1" onclick="callAndroid()">点击按钮调用 Android 的 hello 方法</button>
</body>
</html>
```

然后定义一个 JS 对象映射关系的 Android 类：

```java
public class JSObject extends Object {

    private Context mContext;

    public JSObject(Context context) {
        mContext = context;
    }

    @JavascriptInterface
    public void hello(String msg){
        Toast.makeText(mContext, "JS 调用了 Android 的 hello 方法", Toast.LENGTH_SHORT).show();
    }
}
```

最后就是通过 WebView 设置 Android 类与 JS 代码的映射：

```java
mWebView.loadUrl("file:///android_asset/js_to_android.html");
mWebView.addJavascriptInterface(new JSObject(this),"test");
```

第二种方式：WebViewClient.shouldOverrideUrlLoading 方法回调拦截 url

Android 通过 WebViewClient 的回调方法 shouldOverrideUrlLoading 拦截 url，解析该 url 协议，如果检测到是预先约定好的协议，就调用 Android 相应的方法。

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Demo</title>
    <script>
         function callAndroid(){
            document.location = "js://webview?arg1=2333&arg2=222";
         }
      </script>
</head>
<body>
<p2>JS 调用 Android 方法</p2>
<button type="button" id="button1" onclick="callAndroid()">点击按钮调用 Android 的方法</button>
</body>
</html>
```

```java
mWebView.loadUrl("file:///android_asset/js_call_android.html");
        mWebView.setWebViewClient(new WebViewClient(){
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
                if ("js".equals(request.getUrl().getScheme())){
                    if ("webview".equals(request.getUrl().getAuthority())){
                        Toast.makeText(WebViewContactActivity.this, "JS 调用 Android 方法，参数一为："+request.getUrl().getQueryParameter("arg1"), Toast.LENGTH_SHORT).show();
                    }
                    return true;
                }
                return super.shouldOverrideUrlLoading(view, request);
            }
        });
```

第三种方式：通过 WebChromeClient 的 onJsAlert、onJsConfirm、onJsPrompt 方法回调拦截 JS 对话框的消息

这里只示例 onJsPrompt 的回调，因为这个方法可以返回任意类型的值。

```java
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Demo</title>
    <script>
         function callAndroid(){
            var result=prompt("js://demo?arg1=111&arg2=222");
            alert("demo " + result);
         }
      </script>
</head>
<body>
<p2>JS 调用 Android 方法</p2>
<button type="button" id="button1" onclick="callAndroid()">点击按钮调用 Android 的方法</button>
</body>
</html>
```

```java
       mWebView.loadUrl("file:///android_asset/js_call_android_demo.html");
        mWebView.setWebChromeClient(new WebChromeClient(){
            @Override
            public boolean onJsPrompt(WebView view, String url, String message, String defaultValue, JsPromptResult result) {
                Uri uri=Uri.parse(message);
                if ("js".equals(uri.getScheme())){
                    if ("demo".equals(uri.getAuthority())){
                        result.confirm("JS 调用了 Android 的方法");
                    }
                    return true;
                }
                return super.onJsPrompt(view, url, message, defaultValue, result);
            }
        });
```

三种方式的比较：

| 调用方式                                                     | 优点       | 缺点                     | 使用场景                           |
| ------------------------------------------------------------ | ---------- | ------------------------ | ---------------------------------- |
| WebView.addJavascriptInterface 对象映射                      | 方便简洁   | Android 4.2 一下存在漏洞 | Android 4.2 以上相对简单的应用场景 |
| WebViewClient.shouldOverrideUrlLoading 回调拦截              | 不存在漏洞 | 使用复杂，需要协议约束   | 不需要返回值情况下                 |
| WebChormeClient.onJsAlert / onJsConfirm / onJsPrompt 方法回调拦截 | 不存在漏洞 | 使用复杂，需要协议约束   | 能满足大多数场景                   |

#### WebView 常见问题

1. WebView 销毁

   ```java
       @Override
       protected void onDestroy() {
           super.onDestroy();
           if (mWebView != null) {
               mWebView.loadDataWithBaseURL("", null, "text/html", "utf-8", null);
               mWebView.clearHistory();
               ((ViewGroup) mWebView.getParent()).removeView(mWebView);
               mWebView.destroy();
               mWebView = null;
           }
       }
   ```

2. Android P 阻止加载任何 http 的请求

   Mainfest 中加入：

   ```
   android:usesCleartextTraffic="true"
   ```

3. Android 5.0 之后 WebView 禁止加载 http 与 https 混合内容

   ```java
   if (Build.VERSION.SDK_INT>Build.VERSION_CODES.LOLLIPOP){
             mWebView.getSettings().
                 setMixedContentMode(WebSettings.MIXED_CONTENT_ALWAYS_ALLOW);
           }
   ```

4. WebView 开启硬件加速导致的问题

   比如不能打开 PDF，播放视频花屏等等。

   关闭硬件加速，或者直接用第三方库吧。

#### WebView 优化

1. 给 WebView 加一个加载进度条

   重写 WebChromeClient 的 onProgressChanged 方法。

2. 提高 HTML 网页加载速度，等页面 finsh 在加载图片

   ```java
   public void int () {
       if(Build.VERSION.SDK_INT >= 19) {
           webView.getSettings().setLoadsImagesAutomatically(true);
       } else {
           webView.getSettings().setLoadsImagesAutomatically(false);
       }
   }
   ```

3. 自定义 WebView 错误页面

   重写 WebViewClient 的 onReceivedError 方法。

#### 参考

[https://www.jianshu.com/p/b9164500d3fb](https://www.jianshu.com/p/b9164500d3fb)