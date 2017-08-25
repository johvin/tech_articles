# Upgrade Insecure Requests

> 本文介绍了 `Upgrade-Insecure-Requests` 指令的诞生背景、作用原理及使用注意事项，如果在此之前你对其中某一点或部分内容没有详细的了解，相信本文会给你带来收获的。

文章写于 <time>2017-08-24</time>，最后更新 <time>2017-08-25</time>

<!-- TOC -->

- [Upgrade Insecure Requests](#upgrade-insecure-requests)
  - [从 `Upgrade-Insecure-Requests："1"` 讲起](#从-upgrade-insecure-requests1-讲起)
  - [为什么要设计 `Upgrade-Insecure-Requests`?](#为什么要设计-upgrade-insecure-requests)
  - [使用 `Upgrade-Insecure-Requests` 完成向 https 升级的最后一步](#使用-upgrade-insecure-requests-完成向-https-升级的最后一步)
    - [非导航升级](#非导航升级)
    - [导航升级](#导航升级)
    - [发现不安全请求](#发现不安全请求)
  - [不完美的地方](#不完美的地方)
  - [总结](#总结)
  - [参考资料](#参考资料)

<!-- /TOC -->


## 从 `Upgrade-Insecure-Requests："1"` 讲起

留心最新的浏览器（Chrome、Firefox、Safari）发出的网络请求，大家能看到有一个请求头字段是 `Upgrade-Insecure-Requests`，这个请求头以前并没有出现过，那么它有什么作用呢？查看 [MDN][MDN Upgrade-Insecure-Requests] 得到如下的解释：

```text
The HTTP Upgrade-Insecure-Requests request header sends a signal to the server expressing the client’s preference for an encrypted and authenticated response, and that it can successfully handle the upgrade-insecure-requests CSP directive.
```

于是我们知道了，这是浏览器告诉服务器，我优先选择经过加密认证的响应（可以理解为 https 响应），并且我能够处理 `upgrade-insecure-requests` 这个 CSP 指令（ CSP 的内容，不在本文范围内，可以参考 [MDN CSP][MDN CSP]）。于是服务器在接收到 `Upgrade-Insecure-Requests："1"` 后，可以重定向的返回请求资源的 https URL，当然也可以做其他的事情，这个在后面会详细介绍。上面的解释的后半句提到了 `upgrade-insecure-requests` 指令，但没有更多的介绍，于是直接看了 [w3c 相关规范][w3c Upgrade-Insecure-Requests]，看到了这句话：

```
This document defines a mechanism which allows authors to instruct a user agent to upgrade insecure resource requests to secure transport before fetching them.
```

这句话的意思通俗来讲就是说程序员通过 `Upgrade-Insecure-Requests` 指令让浏览器使用安全的网络传输去获取不安全的网络资源。举个例子，页面中一张图片的 URL 是 http 协议的，如果设置了 `Upgrade-Insecure-Requests` 指令，那么浏览器会使用 https 协议去获取这个图片。这里有一个[例子][upgrade-insecure-requests example]，可以一观其风采。

看起来很帅的样子，那么具体怎么使用这一指令呢？这个在下文详细介绍，但在此之前，我们先来了解下，w3c 定义这个新指令是为了什么呢？

## 为什么要设计 `Upgrade-Insecure-Requests`?

这就要从 https 说起了，现在很多站点都从 http 升级到 https 了，为的是增加安全性。但整个升级过程并不轻松，主要有两个工作要做：

- 服务器及 CDN 等增加安全证书
- 传统 http 页面中的所有资源替换为 https 协议

步骤一相对简单，步骤二的工作量就很大了，而且这还不是全部，在实际操作中，对于程序员维护的页面还好，麻烦的是提供给运营使用的类似 cms 这种根据输入动态生成的页面，很难保证运营人员不犯错误的将所有资源的链接都使用 https，一旦 https 页面中出现了 http 资源，那么在控制台你将看到类似下图的警告或者错误信息：

![mixed content warning][mixed warning]

这个 warning 是浏览器安全机制决定的，具体内容详见 [Mixed Content 安全机制][Mixed Content]。混合被动/显示内容并不影响用户使用，混合活动内容则直接被浏览器阻止加载，严重的情况站点直接不可用。混合活动内容的严重性暂且不提，即便是混合被动/显示内容，因为木桶原理，瞬间就会瓦解所有为了升级 https 所做的工作，浏览器会认定这样的站点是不安全的。这个问题也是整个互联网在升级到 https 过程中遇到的问题。为了解决这一问题，w3c 提出了基于 `Upgrade-Insecure-Requests` 的解决方案，支持这一指令的浏览器在遇到 http 资源的时候能够自动升级使用 https 协议请求该资源。下面我们一起来了解下，如何使用这一指令完成升级 https 的最后一步。

## 使用 `Upgrade-Insecure-Requests` 完成向 https 升级的最后一步

`Upgrade-Insecure-Requests` 是 CSP 的一种，所以它的使用方式也就是 CSP 的使用方式，一共两种。
- http header
- html meta

http header 方法只需要在响应的 http header 中添加 `Content-Security-Policy: upgrade-insecure-requests;` 设置即可。html meta 方法需要在 `<head></head>` 中间添加 `<meta http-equiv="Content-Security-Policy" content="upgrade-insecure-requests">`。实际中使用哪种方法都可以，方法一操作起来更简单一些。

根据规范描述，对不安全的资源进行升级也分两种情况，非导航升级和导航升级。

### 非导航升级

什么是非导航升级呢？就是对非导航的的不安全资源请求的升级。非导航请求可以理解为不发生跳转的资源请求，如图片、视频、css 链接、js 链接等。非导航的不安全资源请求会自动升级为 https。比如一个站点为 `https://example.com`，其页面包含如下非导航的不安全资源请求：

```html
<img src="http://example.com/image.png">
<img src="http://not-example.com/image.png">
```

在设置了 `upgrade-insecure-requests` 指令后，浏览器会将上面的请求解析为：

```html
<img src="https://example.com/image.png">
<img src="https://not-example.com/image.png">
```

这里的 URL 改变是在网络请求发出之前，所以所有这类非导航的不安全的资源请求都会被转化为安全请求。这里需要注意的是，如果升级后的安全资源不存在的话，会导致网络错误，因为浏览器不会再请求不安全的网络资源。

### 导航升级

假如站点 `https://example.com` 的页面中包含如下的导航类不安全链接：

```html
<a href="http://example.com/">Home</a>
<a href="http://not-example.com/">Link</a>
```

设置了 `upgrade-insecure-requests` 指令后，浏览器会自动升级当前站点的不安全链接，同时保留第三方站点的链接不变，因为直接升级导航类不安全的第三方链接会有很大可能破坏第三方站点，因此这类链接不升级。

*注意，经过测试，Firefox 54 中的导航升级表现与规范描述一致，但在 Chrome 60.0.3112.101 中链接到站内的导航链接并没有自动升级。safari 的表现也与规范不同，首次加载页面点击站内导航链接会自动升级，但有了缓存后，再后退回原页面点击同一站内导航链接则不会自动升级。其他浏览器的表现没有测试，点击查看[测试代码][upgrade-insecure-requests test demo]。*

### 发现不安全请求

如果我们想了解页面中有哪些不安全的网络请求，w3c 规范中也为我们提供了基于浏览器的解决方案。使用 `report-uri` 指令，我们可以借助浏览器自动收集页面中出现的不安全的链接信息，设置如下：

```http
Content-Security-Policy: upgrade-insecure-requests; default-src https:; report-uri /report-csp
```

这里要注意的事，report-uri 只能使用 http header 的方式生效，html meta 的方式不起作用。另外，如果我们不设置页面的安全策略，只收集不安全资源请求的信息也是可以的，此时可以使用 `Content-Security-Policy-Report-Only` 这一设置。

```http
Content-Security-Policy-Report-Only: default-src https:; report-uri /endpoint
```

当遇到不安全的资源时，浏览器会先向 /endpoint 发出信息，然后再发起资源的网络请求。


## 不完美的地方

这个解决方案的设计是好的，但不是有那么句话吗，“理想很丰满，现实很骨感”。我们来看下避不开的兼容性：

![upgrade-insecure-requests compatible][upgrade-insecure-requests caniuse]

caniuse 的数据显示，PC 端 IE 和 Edge “阵亡了”，移动端就更差一些了，android 浏览器、UC 浏览器目前都还不支持。不过该规范目前还在 CR 阶段，而 Edge 也正在考虑实现阶段，待到 REC 了，相信一切都会好起来的。不过话说回来，`upgrade-insecure-requests` 可以认为是渐进增强的一种，所以即便现在支持度还不够，那又怎样呢？

## 总结

说了这么多，最后总结下，`upgrade-insecure-requests` 指令的应用需要注意以下几点：

1. 针对支持的浏览器进行应用（当然，不支持的浏览器应用该指令也不会造成任何负面影响）
1. 应用方式有两种，推荐在服务端使用 http header 方式，简单高效
1. 应用前确保 CDN 或其他资源提供服务都已经支持 https
1. 升级分为非导航类升级和导航类升级
1. 可以对不安全资源进行上报
1. 兼容性


## 参考资料
- [W3C Upgrade-Insecure-Requests 规范][w3c Upgrade-Insecure-Requests]
- [MDN Upgrade-Insecure-Requests][MDN Upgrade-Insecure-Requests]
- [MDN CSP][MDN CSP]
- [Mixed Content][Mixed Content]

<!-- References -->
[MDN Upgrade-Insecure-Requests]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Upgrade-Insecure-Requests
[MDN CSP]: https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
[w3c Upgrade-Insecure-Requests]: https://w3c.github.io/webappsec-upgrade-insecure-requests/
[upgrade-insecure-requests example]: https://googlechrome.github.io/samples/csp-upgrade-insecure-requests/index.html
[upgrade-insecure-requests test demo]: ../demo/upgrade-insecure-requests.html
[mixed warning]: https://mdn.mozillademos.org/files/12543/mixed_content_webconsole.png
[Mixed Content]: https://developer.mozilla.org/en-US/docs/Web/Security/Mixed_content
[upgrade-insecure-requests caniuse]: ../images/upgrade-insecure-requests-caniuse.png
