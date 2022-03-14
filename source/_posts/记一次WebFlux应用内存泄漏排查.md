---
layout: post
title: 记一次WebFlux应用内存泄漏排查
date: 2022-02-17 20:15:15
categories: 
  - 问题排查记录
tags:
  - WebFlux
comments: true
---

# 背景

公司项目存在一个服务，类似于爬虫，需要解析给定的URL，从返回的HTML中提取页面的标题、封面图、摘要、icon等信息。由于这是一个无DB访问的纯内存服务，且下游服务（需解析的URL地址）并非内部服务，无需考虑并发压力，在服务搭建时选用`WebFlux`作为web层框架，选用spring的`WebClient`作为请求下游服务的HTTP客户端。

服务部署于k8s容器内，JDK版本为OpenJDK11，Pod配置4C4G，Java服务配置最大堆内存2G。

<!-- more -->


# 问题描述

服务上线后请求压力不大，但长时间运行后，服务堆内存占用达到99%，日志监控出现大量OOM报错，继而容器Pod重启。重启后可正常工作一段时间，之后再次堆内存占用99%，出现OOM报错。


# 解决过程

## 初步分析

通过容器监控，查看Pod重启前一段时间的机器内存占用图，发现图呈现持续上升趋势，且到达堆内存分配上限后，Pod发生重启。初步推测是发生了内存泄漏。
{% asset_image pod_memory_used.png %}

使用`jmap -histo:live 1`查看存活对象分布，发现byte数组占用内存较多，且`PoolSubpage`对象数量也较多，怀疑是netty发生了内存泄漏。
{% asset_image jmap_histo_live.png %}

排查ELK中的ERROR日志，除OOM报错外，另发现少量netty的报错信息，异常堆栈如下：
```text
LEAK: ByteBuf.release() was not called before it's garbage-collected. See https://netty.io/wiki/reference-counted-objects.html for more information.
Recent access records: 
Created at:
    io.netty.buffer.PooledByteBufAllocator.newHeapBuffer(PooledByteBufAllocator.java:332)
    io.netty.buffer.AbstractByteBufAllocator.heapBuffer(AbstractByteBufAllocator.java:168)
    io.netty.buffer.AbstractByteBufAllocator.heapBuffer(AbstractByteBufAllocator.java:159)
    io.netty.handler.codec.compression.JdkZlibDecoder.decode(JdkZlibDecoder.java:180)
    io.netty.handler.codec.ByteToMessageDecoder.decodeRemovalReentryProtection(ByteToMessageDecoder.java:493)
    io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:432)
    ...
```
从异常提示信息可见，netty的堆内存`ByteBuf`在未被释放的情况下被GC回收，而netty使用内存池进行堆内存管理，如`ByteBuff`未经过`release()`方法调用即被GC回收，将导致内存池中大量内存块的引用计数无法归零，导致内存无法回收。且`ByteBuf`被GC回收后，应用程序已经无法再调用`release()`方法，即导致了内存泄漏。


## 定位问题出现位置

项目中使用netty的地方有：`Redisson`、`WebFlux`、`WebClient`。考虑到第三方库很成熟，经过很多商业项目应用，问题不太可能出现在库代码中，可能是自己的使用方式有误。应用程序中自己编码使用的主要是`WebClient`，用于请求第三方页面HTML。

业务使用场景中，需要读取 ResponseHeader 和 ResponseBody 两部分内容。Header 用于从 Content-Type 中解析编码；Body 用于直接读取二进制数据，确定页面真正的编码格式。

  > 之所以需要确定页面真正编码格式，是因为有些第三方页面，response header中通过 Content-Type 声明编码格式为 UTF-8，但真正的编码格式却是 GBK 或 GB2312，导致解析中文摘要时乱码。因此需要读取二进制流后，根据流内容判断真实编码格式。写过爬虫的兄弟应该理解。

`WebClient`提供了如下多个获取 Response 的方法：
  1. WebClient.RequestHeadersSpec#retrieve
     可以将 body 直接处理为指定类型的对象，但是无法直接操作 response；

  2. WebClient.RequestHeadersSpec#exchange
     可以直接操作 response，但 body 的读取操作需要自行处理；

为满足需求，项目中使用了`WebClient.RequestHeadersSpec#exchange`方法，这也是项目中唯一一处可以直接操作 ByteBuf 数据的地方。在使用此方法时，仅进行了数据读取操作，并没有释放 body。而在方法的注释上，刚好有这么一段：
{% asset_image webclient_requestheadersspec_exchange.png %}

NOTE 部分翻译过来的大致意思是：
> 与 retrieve() 不同，在使用 exchange() 时，不论在任何情况下（成功、异常、无法处理的数据等），应用程序都应当消费掉响应内容。不这样做可能会导致内存泄漏。请参阅 ClientResponse 以获取可用于消费 body 的方式。通常应该使用 retrieve()，除非您有充分的理由使用exchange()，它允许您检查响应状态和标题，并在之后用于决定是否消费body、如何消费body。

而刚好在一些业务校验失败的情况下，如 Content-Type 中标识返回的数据不是 HTML 内容时，应用代码直接进行了 return，而没有消费 body，导致了内存泄漏。
```java
// 请求代码示例
WebClient.builder().build()
    .get()
    .uri(ctx.getUri())
    .headers(headers -> {
        headers.set(HttpHeaders.USER_AGENT, CHROME_AGENT);
        headers.set(HttpHeaders.HOST, ctx.getUri().getHost());
    })
    .cookies(cookies -> ctx.getCookies().forEach(cookies::add))
    .exchange()
    .flatMap(response -> {
        // 再次检测是否超时
        // 注意，这里直接返回了Mono.error，而没有释放response
        if (ctx.isParseTimeout(PARSE_TIMEOUT)) {
            return Mono.error(ReadTimeoutException.INSTANCE);
        }

        // 先解析重定向，不存在重定向则解析body
        return judgeRedirect(response, ctx)
                .flatMap(redirectTo -> followRedirect(ctx, redirectTo))
                .switchIfEmpty(Mono.defer(() -> Mono.just(parser.parse(ctx))))
                .map(LinkParseResult::detectParseFail);
    })
```


## 解决问题

已经定位到问题发生的原因，且官方文档已给出了解决办法`参阅 ClientResponse 以获取可用于消费 body 的方式`。在`ClientResponse`接口的注释上，列出来所有用于消费 Response 的方法：
{% asset_image clientresponse.png %}

具体每个方法的作用就不赘述，根据业务场景，应当在不需要消费 body 时调用 `releaseBody()` 方法进行释放。修改后的代码如下：
```java
// 请求代码示例
WebClient.builder().build()
    .get()
    .uri(ctx.getUri())
    .headers(headers -> {
        headers.set(HttpHeaders.USER_AGENT, CHROME_AGENT);
        headers.set(HttpHeaders.HOST, ctx.getUri().getHost());
    })
    .cookies(cookies -> ctx.getCookies().forEach(cookies::add))
    .exchange()
    .flatMap(response -> {
        // 再次检测是否超时，并释放response
        if (ctx.isParseTimeout(PARSE_TIMEOUT)) {
            return response.releaseBody()
                    .then(Mono.error(ReadTimeoutException.INSTANCE));
        }

        // 先解析重定向，不存在重定向则解析body
        return judgeRedirect(response, ctx)
                .flatMap(redirectTo -> followRedirect(ctx, redirectTo))
                .switchIfEmpty(Mono.defer(() -> Mono.just(parser.parse(ctx))))
                .map(LinkParseResult::detectParseFail);
    })
```


# 总结
在使用响应式HTTP客户端`WebClient`时，接受响应数据使用了 `exchange()` 方法，但又在一些流程分支中没有调用 `ClientResponse#releaseBody()` 方法，导致大量数据得不到释放，netty内存池占满，后续的请求在申请内存时报OOM异常。

得到经验教训：使用不熟悉的三方库时，一定要阅读方法注释、类注释。


--- 
参考文档：
1. [Netty内存泄漏排查](http://www.bewindoweb.com/291.html)
2. [Web on Reactive Stack](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html)