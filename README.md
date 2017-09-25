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

