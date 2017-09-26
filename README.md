# 页面的性能优化

## 工作中遇到的提升页面性能，减少页面加载时间主要有以下几个方法：

### 1. 资源压缩合并，减少 HTTP 的请求
### 2. 非核心的代码异步加载
### 3. 利用浏览器的缓存
### 4. 使用 CDN
### 5. 预加载 DNS

-----
## 下面将详细的说下每个方法的使用
### 一. 资源压缩合并，减少 HTTP 的请求
#### 1. 如果是图片的话
我们可以使用 data:url 模式。可以在页面中渲染图片但无需额外的 HTTP 请求。
```
<img scr="data:image/jpg;base64, xxxxxxxxxxxxxxxx">
```
 但是该方法有个小缺点，**无法利用浏览器的缓存, 所以为了解决问题，我们得把图片数据放在 CSS 文件里。**

```
.imageA {
   background-image: url(data:image/jpg;base64, xxxxxxxxxxxxxxxx);
}
```
**注意:** 移动端不建议用src="data:image..."，性能非常不好。

**使用 base64 优点：** 
> - 减少了 http 请求
> - 数据就是图片
> - 某些文件可以避免跨域的问题

**使用 base64 缺点：** 
> - IE6/IE7 浏览器是不支持的
> - 增加了 CSS 文件的尺寸

#### 2. 合并 JS 和 CSS 文件
> - 利用项目构建工具，如 gulp, grunt, webpack 等等，都可以做到 JS 或者 CSS 文件的压缩
> - HTTP 1.1 默认在 request header 里面开启 gzip, 使用 gzip 编码来压缩 HTTP 响应包，由此可以减少网络响应时间. 例子：Accept-Encoding:gzip.

----

### 二. 非核心的代码异步加载
#### 1. 动态加载
所谓动态加载脚本就是利用 javascript 代码来加载脚本，通常是手工创建 script 元素，然后等到 HTML 文档解析完毕后插入到文档中去。这样就可以很好地控制脚本加载的时机，从而避免阻塞问题。
```
function loadJS(src) {
  const script = document.createElement('script');
  script.src = src;
  document.getElementsByTagName('head')[0].appendChild(script);
}
loadJS('http://example.com/scq000.js');
```
#### 2. 异步加载方式
1） defer
```
<script src="file.js" defer></script> 
```
**注:** defer 是在 HTML 解析完之后才会执行，如果是多个，按照加载的顺序依次执行。

2） async
```
<script src="file.js" async></script> 
```
**注:** async 属性是 HTML5 新增的。作用和 defer 类似，但是它将在下载后立即执行，**不能保证脚本会按顺序执行**。它们将在 onload 事件之前完成。

**上面介绍的异步加载脚本并不是十分完美的**, 如果你熟悉 promise 的话，就知道这是在 JS 中处理异步的一种强有力的工具。下面以 promise 技术来实现处理异步脚本加载过程中的依赖问题：
```
// 执行脚本
function exec(src) {
    const script = document.createElement('script');
    script.src = src;

      // 返回一个独立的promise
    return new Promise((resolve, reject) => {
        var done = false;

        script.onload = script.onreadystatechange = () => {
            if (!done && (!script.readyState || script.readyState === "loaded" || script.readyState === "complete")) {
              done = true;

              // 避免内存泄漏
              script.onload = script.onreadystatechange = null;
              resolve(script);
            }
        }

        script.onerror = reject;
        document.getElementsByTagName('head')[0].appendChild(script);
    });
}

function asyncLoadJS(dependencies) {
    return Promise.all(dependencies.map(exec));
}

asyncLoadJS(['https://code.jquery.com/jquery-2.2.1.js', 'https://cdn.bootcss.com/bootstrap/3.3.7/js/bootstrap.min.js']).then(() => console.log('all done'));
```
可以看到，我们针对每个脚本依赖都会创建一个 promise 对象来管理其状态。采用动态插入脚本的方式来管理脚本，然后利用脚本 onload 和 onreadystatechange (兼容性处理)事件来监听脚本是否加载完成。一旦加载完毕，就会触发 promise 的 resovle 方法。最后，针对依赖的处理，是 promise 的 all 方法，这个方法只有在所有 promise 对象都 resolved 的时候才会触发 resolve 方法，这样一来，我们就可以确保在执行回调之前，所有依赖的脚本都已经加载并执行完毕。

### 三. 利用浏览器的缓存
#### 缓存的分类
浏览器缓存分为 **强缓存** 和 **协商缓存**，浏览器在加载资源时，会先根据 http header 判断它是否命中强缓存，强缓存如果命中，**则直接在自己的缓存中读取资源**，且**不会发送请求到服务器**。当强缓存没有命中时，浏览器会发送请求到服务器，通过服务器端的 http header 验证这个资源是否命中协商缓存，如果命中，服务器会将这个请求返回，**（不会返回这个请求资源）**，而是告诉浏览器，可以直接从缓存中加载这个资源。

强缓存与协商缓存的 **共同点是：** 如果命中，都是从客户端缓存中加载资源，而不是从服务器加载资源。

**不同点是：** 强缓存不发请求到服务器，协商缓存会发请求到服务器。

### 1. **强缓存**
a. 当浏览器对某个资源的请求命中了强缓存时，返回的 http 状态为 200

b. 强缓存是利用 Expires 或者 Cache-Control 这两个 http response header 实现的,它们都用来表示资源在客户端缓存的有效期。
> **Expires 是 http1.0 提出的一个表示资源过期时间的 header，它描述的是一个绝对时间,用 GMT 格式的字符串表示.**
> - 浏览器第一次跟服务器请求一个资源，服务器在返回这个资源的同时，在 respone 的 header 加上 Expires 的 header
> - 浏览器在接收到这个资源后，会把这个资源连同所有 response header 一起缓存下来
> - 浏览器再请求这个资源时，**先从缓存中寻找**，找到这个资源后，拿出它的 Expires 跟当前的请求时间比较，如果请求时间在 Expires 指定的时间之前，就能命中缓存。如果缓存没有命中，浏览器直接从服务器加载资源时，**Expires Header 在重新加载的时候会被更新**。

> **Expires 是较老的强缓存管理 header,它是服务器返回的一个绝对时间.所以在 http1.1 的时候，提出了一个新的 header，就是 Cache-Control，这是一个相对时间**
> - 浏览器第一次跟服务器请求一个资源，服务器在返回这个资源的同时，在 respone 的 header 加上 Cache-Control 的 header.
> - 浏览器在接收到这个资源后，会把这个资源连同所有 response header 一起缓存下来.
> - 浏览器再请求这个资源时，**先从缓存中寻找**，找到这个资源后，根据它第一次的请求时间和 Cache-Control 设定的有效期，计算出一个资源过期时间，再拿这个过期时间跟当前的请求时间比较，如果请求时间在过期时间之前，就能命中缓存。如果缓存没有命中，浏览器直接从服务器加载资源时，**Cache-Control Header 在重新加载的时候会被更新**。

Cache-Control 描述的是一个相对时间，在进行缓存命中的时候，都是利用客户端时间进行判断，**所以相比较 Expires，Cache-Control 的缓存管理更有效，安全一些。**

这两个 header 可以只启用一个，也可以同时启用，当 response header 中，Expires 和 Cache-Control 同时存在时，**Cache-Control 优先级高于 Expires**

强缓存还有一点需要注意的是，**通常都是针对静态资源使用，动态资源需要慎用**，除了服务端页面可以看作动态资源外，**那些引用静态资源的 html 也可以看作是动态资源**，如果这种 html 也被缓存，当这些 html 更新之后，可能就没有机制能够通知浏览器这些 html 有更新，尤其是前后端分离的应用里，页面都是纯 html 页面，每个访问地址可能都是直接访问 html 页面，**这些页面通常不加强缓存，以保证浏览器访问这些页面时始终请求服务器最新的资源**。

### 2. **协商缓存**
a. 当浏览器对某个资源的请求没有命中强缓存，就会发一个请求到服务器，验证协商缓存是否命中.

b. 如果协商缓存命中，**请求响应返回的 http 状态为 304** 并且会显示一个 **Not Modified** 的字符串.

c. 查看单个请求的 Response Header，也能看到 304 的状态码和 Not Modified 的字符串，只要看到这个就可说明这个资源是命中了协商缓存，然后从客户端缓存中加载的，而不是服务器最新的资源.

d. 协商缓存是利用的是 **【Last-Modified，If-Modified-Since】和【ETag、If-None-Match】** 这两对 Header 来管理的.

> **Last-Modified**
> - 浏览器第一次跟服务器请求一个资源,服务器将资源传递给浏览器时，会将资源最后更改的时间以 “Last-Modified: GMT” 的形式加在实体首部上一起返回给客户端。
> - 浏览器会为资源标记上该信息，下次再次请求时，会把该信息附带在请求报文中一并带给服务器去做检查，若传递的时间值与服务器上该资源最终修改时间是一致的，则说明该资源没有被修改过，直接返回 304 状态码即可,但是不会返回资源内容。
> - 如果有变化，就正常返回资源内容。当服务器返回304 Not Modified的响应时，response header 中不会再添加 Last-Modified 的 header，因为既然资源没有变化，那么 Last-Modified 也就不会改变.

Last-Modified 说好却也不是特别好，因为如果在服务器上，一个资源被修改了，但其实际内容根本没发生改变，会因为Last-Modified时间匹配不上而返回了整个实体给客户端.

为了解决上述 Last-Modified 可能存在的不准确的问题，Http1.1 还推出了  ETag 实体首部字段。

> **ETag**
> - 服务器会通过某种算法，给资源计算得出一个唯一标志符（比如md5标志），在把资源响应给客户端的时候，会在实体首部加上“ETag: 唯一标识符”一起返回给客户端。
> - 客户端会保留该 ETag 字段，并在下一次请求时将其一并带过去给服务器。服务器只需要比较客户端传来的 ETag 跟自己服务器上该资源的 ETag 是否一致，就能很好地判断资源相对客户端而言是否被修改过了。
> - 如果服务器发现 ETag 匹配不上，那么直接以常规 GET 200 形式将新的资源（当然也包括了新的ETag）发给客户端；如果 ETag 是一致的，则直接返回 304 知会客户端直接使用本地缓存即可。

协商缓存跟强缓存不一样，强缓存不发请求到服务器，所以有时候资源更新了浏览器还不知道，但是协商缓存会发请求到服务器，所以资源是否更新，服务器肯定知道。

如果没有协商缓存，每个到服务器的请求，就都得返回资源内容，这样服务器的性能会极差。

【Last-Modified，If-Modified-Since】和【ETag、If-None-Match】一般都是同时启用，这是为了处理 Last-Modified 不可靠的情况.

> **有一种场景需要注意：**
> - 分布式系统里多台机器间文件的 Last-Modified 必须保持一致，以免负载均衡到不同机器导致比对失败；
> - 分布式系统尽量关闭掉 ETag (每台机器生成的 ETag 都会不一样）

### 四. 使用 CDN

网站 80-90% 响应时间消耗在资源下载上，减少资源下载时间是性能优化的黄金发则。

相比分布式架构的复杂和巨大投入，静态内容分发网络（CDN）可以以较低的投入，获得加载速度有效提升。

但是在实际开发中没有用到过，没法详细解释。

### 五. 预加载 DNS

通过 DNS 预解析来告诉浏览器未来我们可能从某个特定的 URL 获取资源，当浏览器真正使用到该域中的某个资源时就可以尽快地完成 DNS 解析。

#### DNS 预解析 DNS-Prefetch
```
<link rel="dns-prefetch" href="//example.com">
```
当我们从该 URL 请求一个资源时，就不再需要等待 DNS 的解析过程。该技术对使用第三方资源特别有用。

在 Harry Roberts 的[文章](https://csswizardry.com/2013/01/front-end-performance-for-web-designers-and-front-end-developers/#section:dns-prefetching)中提到：
>  通过简单的一行代码就可以告知那些兼容的浏览器进行 DNS 预解析，这意味着当浏览器真正请求该域中的某个资源时，DNS 的解析就已经完成了。

#### 预连接 Preconnect

与 DNS 预解析类似，preconnect 不仅完成 DNS 预解析，同时还将进行 TCP 握手和建立传输层协议。可以这样使用：
```
<link rel="preconnect" href="http://example.com">
```
在 Ilya Grigorik 的[文章](https://www.igvita.com/2015/08/17/eliminating-roundtrips-with-preconnect/)中有更详细的介绍：
>现代浏览器都试着预测网站将来需要哪些连接，然后预先建立 socket 连接，从而消除昂贵的 DNS 查找、TCP 握手和 TLS 往返开销。然而，浏览器还不够聪明，并不能准确预测每个网站的所有预链接目标。好在，在 Firefox 39 和 Chrome 46 中我们可以使用 preconnect 告诉浏览器我们需要进行哪些预连接。

#### 预获取 Prefetching
如果我们确定某个资源将来一定会被使用到，我们可以让浏览器预先请求该资源并放入浏览器缓存中。例如，一个图片和脚本或任何可以被浏览器缓存的资源：
```
<link rel="prefetch" href="image.png">
```
与 DNS 预解析不同，预获取真正请求并下载了资源，并储存在缓存中

#### Subresources
这是另一个预获取方式，这种方式指定的预获取资源具有最高的优先级，在所有 prefetch 项之前进行：
```
<link rel="subresource" href="styles.css">
```
根据 Chrome 文档：
> rel=prefetch 为将来的页面提供了一种低优先级的资源预加载方式，而 rel=subresource 为当前页面提供了一种高优先级的资源预加载。
所以，如果资源是当前页面必须的，或者资源需要尽快可用，那么最好使用 subresource 而不是 prefetch。

#### 预渲染 Prerender

这是一个核武器，因为 prerender 可以预先加载文档的所有资源：
```
<link rel="prerender" href="http://example.com">
```
> 这类似于在一个隐藏的 tab 页中打开了某个链接 – 将下载所有资源、创建 DOM 结构、完成页面布局、应用 CSS 样式和执行 JavaScript 脚本等。当用户真正访问该链接时，隐藏的页面就切换为可见，使页面看起来就是瞬间加载完成一样。Google 搜索在其即时搜索页面中已经应用该技术多年了，微软也宣称将在 IE11 中支持该特性

需要注意的是不要滥用该特性，当你知道用户一定会点击某个链接时才可以进行预渲染，否则浏览器将无条件地下载所有预渲染需要的资源。