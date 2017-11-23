从写C#到写JS也有三年左右了，其间遇到过很多挫折，也有很多成长。趁现在公司不太忙，总结下自己积累的一些知识。

web全链路优化的一些总结

浏览器请求

DNS

从浏览器窗口输入一个URL，首先浏览器会查找本地DNS缓存缓存，如果没有，会请求DNS服务器，进行域名解析。如果想页面中有链接或者其他资源链接想要预先解析DNS可以使用link标签来强制预解析DNS，如：<link rel="dns-prefetch" href="https://www.github.com/">，这将加快网站资源的加载性能。更多有关DNS相关知识：https://developer.mozilla.org/zh-CN/docs/Controlling_DNS_prefetching

网络请求相关

减小带宽

我们知道Http底层是基于TCP/IP的，一次IP报文最大长度为65535字节，而减少传输量对网站性能至关重要。

服务器开启gzip:服务器会把Response的body部分使用zip算法压缩，减少内容传输。目前大部分浏览器器都支持gzip。NodeJS Http Server中可以使用 compression包来压缩，具体使用方法请参阅：https://www.npmjs.com/package/compression
图片压缩：网页中，一张jpg、png的图片往往几百KB，甚至于几兆。大大占用了带宽。其实大部分的图片都是有压缩空间的，我们从PS切图得到的图片 往往有50%甚至更高的压缩空间。图片压缩可以大大减少往来传输时间。推荐两个在线压缩图片的地址：http://optimizilla.com/zh/ 、 https://tinypng.com/， 都是无损压缩，一张图片可以两个网站都minify下哦。
减少网络请求

一次TCP/IP的请求是要经过三次握手和四次挥手的动作的。并且浏览器同事发送的请求也是有限制的，见下图：

浏览器最大连接数
因此减少http请求对提高性能来说有着至关重要的影响。

合并压缩JS、CSS、HTML：对网页加载完毕就要马上执行的JS，可以合并压缩成一个JS文件，减少频繁网络请求。CSS和html也可以压缩成一行，减少带宽传输。可以使用gulp,webpack等现代化工具。
使用雪碧图（Sprite）：对网站经常使用图片类型的图标，可以合并成一张图片，使用background-position属性来显示不同的图标。
JS加载：浏览器同时的http请求是有限制的，有一些模块的JS并不是页面加载完毕就要马上运行的，因此我们可以在windows的onload事件中来加载这一部分JS代码。 当然我们也可以在script标签中使用defer，添加defer属性的标签将在页面的DOMContentLoaded事件后去加载。现代的前端工程化已经做到了按需加载，当需要某一部分JS或者HTML片段的时候 再去服务器加载。 PS:DOMContentLoaded事件是指浏览器html结构物已经加载完毕，但不保证其他资源（如：image,video,页面内嵌ifarme）等加载完毕。JQuery的ready利用该事件实现。 而onload事件则是页面所有资源都已加载完毕.
keep-alive:早期的浏览器都是每一次请求都会进行一次TPC/IP连接，每一次连接都会有三次握手和四次挥手的动作消耗。现在浏览器和服务器都支持Keep-Alive机制，开启Keep-Alive后 服务器在第一次建立连接进行响应后会暂时不释放该连接，等设置的keep-alive timeout时间后再释放此次连接。Keep-Alive会减少客户端和服务器连接的操作，减少TCP/IP的socket的accept()和 close()调用，节省CPU消耗。但是如果keep-alive timeout的时间过长，会造成服务器内存飙高，无用socket过多，造成更大的性能消耗。所以根据业务场景设置正确的keep-alive timeout时间 十分重要。推荐阅读：http://www.nowamagic.net/academy/detail/23350305
webSocket:一些业务场景中有频繁更新数据的需求，以往我们一般通过Ajax轮询的办法或者Http长连接的办法来实现服务器同客户端的数据的快速更新，但是，Ajax轮询的方式会有很多的请求并不会得到 数据，而长连接的方式会造成服务器内存占用，同一时间连接太多，会造成网站运行缓慢，甚至崩溃。 Html5的webSocket解决了这种困局，WebSocket协议是基于TCP的一种新的网络协议。它实现了浏览器与服务器全双工(full-duplex)通信——允许服务器主动发送信息给客户端。彻底改变了 以往的交互方式。Websocket一次连接后可以在双方发送多次消息。大大提高了网站性能。更多参考MDN：https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket
缓存

response响应头控制：

Expires:内容过期时间，如果浏览器再次请求该路径是，请求时间小于过期时间，则直接使用该缓存。缺点：用户浏览器时间可能和服务器时间不一致，导致缓存策略失效。
Cache-Control：如max-age=**，已秒为单位，在这段时间内请求该资源，全部使用缓存;no-cache,不使用缓存。其他诸如：private，public,no-store等，目前阶段最常用一般是max-age=**秒数。
Last-Modified/If-Modified-Since：服务在返回资源的时候根据资源最后更新的日期来设置Last-Modified值，浏览再次请求该资源的时候再请求头中添加If-Modified-Since标识已上次Last-Modified为值， 服务器发现请求头有If-Modified-Since请求头后跟请求资源的最新做比较，如果时间一致，则返回状态码304告诉浏览器使用本地缓存。否则返回最新资源。 缺点：Last-Modified只能控制到秒这一级别，如果服务器一秒钟内对一个资源有过多次更新，则改缓存规则失效。
ETag/If-None-Match: 该机制和Last-Modified机制相似，不同的是ETag是返回是资源的Hash值，一般使用最后更新的毫秒数+文件名称的MD5.再次请求时使用请求头添加If-None-Match使用ETage的值，服务器做对比后判断是否响应304. 缺点：如果使用服务器集群，不同服务器之间可能返回的Hash码，不同而导致缓存策略失效。而且ETag和Last-Modified缓存方式读取缓存的时候都会询问一次服务器，两者适合于资源更新频繁的资源缓存。
其他缓存控制

服务器缓存：服务器可以把经常请求的文件或者模板存入内存中，避免频繁从磁盘读取的IO操作所消耗的性能和时间。webpack-dev-middleware就是把打包完的文件直接写入内存，加快访问。
缓存常用接口数据：Html5为我们提供了LocalStorage等存储功能，我们可以一些更新频率较低的基础数据（如经常用到的省市地区信息、下拉框绑定的各种选项）缓存在本地存储中，来减少网络请求，同时也加快了dom渲染进度。 但是一定要有一个好的更新数据的策略，比如登录时返回最新的基础数据更新版本等。
DOM优化

style标签：todo
