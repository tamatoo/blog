####系列文章包括：
web性能优化(一) 使用压缩传输  
[web性能优化(二) 合理利用cache-control](https://github.com/zhangyaowu/blog/blob/master/web/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%BA%8C)%20%E5%90%88%E7%90%86%E5%88%A9%E7%94%A8%E6%B5%8F%E8%A7%88%E5%99%A8%E7%BC%93%E5%AD%98.md  
"web性能优化(二)合理利用cache-control")  
[web性能优化(三) Minify Javascript and CSS](https://github.com/zhangyaowu/blog/blob/master/web/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%B8%89)%20Minify%20Javascript%20and%20CSS.md)   
web性能优化(四) 使用chrome developer tools做页面性能分析和需找页面性能优化点(待写)   
***
#####背景
企业级J2EE应用，大部分情况运行在企业内部网络，千兆网甚至万兆网。所以web性能是一个比较容易被忽略的问题。过去的 web应用还没有现在百花齐放的框架和类库，往往打开一个页面加载的资源可能只有几十K，大部分情况不超过200K，所以每打开页面加载的资源量一直不被关注。但是，一旦把应用部署到公网、或者网络条件不是太好的时候，页面加载缓慢的问题就一下子暴露出来了。并且现在前端框架和类库百花齐放，下面是当今流行的框架和类库的大小:  

框架/库 | 大小 
:--: | :--:
jquery-1.11.0.js | 276KB
jquery-1.11.0.min.js | 94KB
angular-1.2.15.js | 729KB
angular-1.2.15.min.js | 101KB
bootstrap-3.1.1.css | 118KB
bootstrap-3.1.1.min.css | 98KB
foundation-5.css | 186KB
foundation-5.min.css | 146KB
uee/app.js | 1.6MB
uee/app.datetime.js | 256KB

某应用的首页加载资源大小：  
![](https://github.com/kaelhuawei/blog/blob/master/web/images/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E4%BD%BF%E7%94%A8%E5%8E%8B%E7%BC%A9%E4%BC%A0%E8%BE%93/tagstore%20all%20resources%20.jpg)  
某应用标签详情页面加载资源大小：  
![](https://github.com/kaelhuawei/blog/blob/master/web/images/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E4%BD%BF%E7%94%A8%E5%8E%8B%E7%BC%A9%E4%BC%A0%E8%BE%93/tagdetail%20all%20resource.jpg)  
这是令人发指的和不能接受的，分别加载了2.8M和5.8M的资源！优化性能减少资源大小有两个途径，减小资源原始大小和减小网络传输大小。减小资源原始大小即ugilify JS/CSS、合并脚本等，这个涉及前端工程化的咚咚，单独写一篇文章分析（web性能优化(三) Minify Javascript and CSS），这篇文章谈谈怎么压缩组件以减小网络传输大小来优化web性能。  

从HTTP1.1开始，web客户端可以在HTTP请求中添加Accept-Encoding头来表示对压缩的支持，如：Accept-Encoding:gzip, deflate, sdch  
客户端在response中以Content-Encoding头来通知客户端响应的编码格式，如Content-Encoding:gzip  

需要注意的是：  
* 压缩仅仅是减少了网络传输量
* 消除不必要的数据是最好的（见 web性能优化(四) 合并、删除js和样式表&利用chrome developer tools做页面性能分析）
* 压缩技术和算法有很多种，这边文章里没有特别指定的话特指gzip算法
* 启用压缩传输需要服务器支持，这篇文章只介绍tomcat如何启用压缩传输  

实测了一下google首页，除了图片、dom、个别大小很小的html，其他资源(html、javascript、CSS、xhr)全部启用了gzip传输：  
![](https://github.com/kaelhuawei/blog/blob/master/web/images/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E4%BD%BF%E7%94%A8%E5%8E%8B%E7%BC%A9%E4%BC%A0%E8%BE%93/google%20s%20request%20and%20response.jpg)  

压缩传输是一种用cpu转移带宽压力的手段。这里看下tomcat怎么启用content-encoding.  
[tomcat的文档](https://tomcat.apache.org/tomcat-7.0-doc/config/http.html#Connector_Comparison "")对connector的compression属性是这么描述的：  
The Connector may use HTTP/1.1 GZIP compression in an attempt to save server bandwidth. The acceptable values for the parameter is "off" (disable compression), "on" (allow compression, which causes text data to be compressed), "force" (forces compression in all cases), or a numerical integer value (which is equivalent to "on", but specifies the minimum amount of data before the output is compressed). If the content-length is not known and compression is set to "on" or more aggressive, the output will also be compressed. If not specified, this attribute is set to "off".
Note: There is a tradeoff between using compression (saving your bandwidth) and using the sendfile feature (saving your CPU cycles). If the connector supports the sendfile feature, e.g. the NIO connector, using sendfile will take precedence over compression. The symptoms will be that static files greater that 48 Kb will be sent uncompressed. You can turn off sendfile by setting useSendfile attribute of the connector, as documented below, or change the sendfile usage threshold in the configuration of the DefaultServlet in the default conf/web.xml or in the web.xml of your web application.
支持compression的connector有BIO, NIO 和 APR/native.  
#####tomcat的典型配置：  
```xml
compression="on" compressionMinSize="2048" noCompressionUserAgents="gozilla,traviata"
compressableMimeType="text/html,text/xml,text/plain,text/css,text/javascript,text/json,application/x-javascript,application/javascript,application/json"
```
其中，  
* compression="on"启用response压缩传输能力
* compressionMinSize="2048"超过2kb的文件才压缩传输
* CompressionUserAgents="gozilla,traviata"不启用压缩传输的浏览器
* compressableMimeType需要压缩传输的文件Mine，典型的html、js、css都应该被压缩，设置是超过一定大小的json。图片和pdf不应该被压缩，因为它们本来已经被压缩过了，再压缩只会浪费CPU资源，还有可能会增加文件大小

关于compressionMinSize要补充说一些，这个配置项存在的意义是，如果一个文件压缩前就足够小，当gzip字典大于压缩产生的节省字节数时，会出现压缩后的大小比原始大小还要打的情况。所以合理的设置一个阈值有重要的意义。 

#####关于缓存代理和reponse中的Vary头
浏览器直接与服务器交互时，上面的配置能很好的工作。当浏览器通过代理发送请求时，情况就变得复杂了。假设访问某个URL的第一个请求来自某个不支持gzip的浏览器，经过代理转发到服务端的请求返回的响应是未压缩的资源，当请求同样URL的第二个请求来自支持gzip的浏览器时，代理商缓存的是未压缩的版本，代理不会去服务端请求压缩版本的资源。而如果第一次请求来自支持gzip的浏览器，后续的请求来自不支持gzip的浏览器，这种情形错误就更严重了。  
针对这个问题，就需要Vary头出场了，web服务器告诉代理根据一个或者多个请求头来改变缓存的响应。如在reponse中加这样的头：Vary: Accept-Ecoding.

#####效果
有人专门统计过，gzip传输大约能将响应整体减小66%。
看一下启用compression以后，某应用的前后效果：  
首页：  
![](https://github.com/kaelhuawei/blog/blob/master/web/images/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E4%BD%BF%E7%94%A8%E5%8E%8B%E7%BC%A9%E4%BC%A0%E8%BE%93/tagstore%20all%20resources%20.jpg)  
![](https://github.com/kaelhuawei/blog/blob/master/web/images/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E4%BD%BF%E7%94%A8%E5%8E%8B%E7%BC%A9%E4%BC%A0%E8%BE%93/tagstore%20after%20compression.jpg)  
标签详情的前后效果：  
![](https://github.com/kaelhuawei/blog/blob/master/web/images/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E4%BD%BF%E7%94%A8%E5%8E%8B%E7%BC%A9%E4%BC%A0%E8%BE%93/tagdetail%20all%20resource.jpg)![](https://github.com/kaelhuawei/blog/blob/master/web/images/web%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96(%E4%B8%80)%20%E4%BD%BF%E7%94%A8%E5%8E%8B%E7%BC%A9%E4%BC%A0%E8%BE%93/tagdetail%20after%20compression.jpg)  
虽然少了小一半，结果仍然是令人发指的。优化未完。
