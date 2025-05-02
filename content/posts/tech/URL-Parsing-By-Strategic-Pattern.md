---
title: "[Tech]使用策略模式实现将多条URL消息解析成多张小卡片"
date: 2025-01-18T22:40:03+08:00
lastmod: 2025-01-24T22:40:03+08:00
author: ["Fubos"]

categories:
- tech

tags:
- 策略模式
- URL解析

keywords:
- 策略模式
- URL解析

description: "" # 文章描述，与搜索优化相关
summary: "" # 文章简单描述，会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true
showToc: true # 显示目录
TocOpen: true # 自动展开目录
autonumbering: true # 目录自动编号
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
searchHidden: false # 该页面可以被搜索到
showbreadcrumbs: true #顶部显示当前路径
ShowWordCounts: true #显示字数统计
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

使用策略模式实现多条url消息解析成多张小卡片进行展示，并实现跳转功能。普通的url消息发送只有一串地址，不点进去并不知道网页的具体内容，如果我们能够在发送消息的时候就解析出来对应的链接中的内容，这样就能够提前让其他用户知道链接中的主要主题是什么，感不感兴趣，从而有更好的用户体验。
![实际效果图](/posts_imgs/url-parsing-cards.png))

### 实现原理：

在目前的一些内容平台，一般采用正则匹配对发送内容中包含的url进行识别，首先提前访问url，并进行标题解析，然后获取对应网站的title标签内容。其中的难点在于什么时候进行解析或匹配url，是后端还是前端做，是入库的时候做还是列表查询时做，和前端的消息体是如何对齐的。

目前已有的类似功能是知识星球中的消息中如果包含url，会解析对应的主题，并进行展示，

我们的实现主要是让发送消息的发送者进行解析，然后将解析后得到的消息进行入库，下次别的用户读取该url消息时，能够快速读取，无需再次请求。

1）因为如果在用户访问消息列表的时候，将消息查出来交给后端解析，这样不同的人进行请求时，会重复解析，拉高整体的接口响应，占用后端资源。

2）如果将原文丢给用户，让前端进行解析，当有千人同时在群里发一条链接，这样就会有一千个前端对该网站进行请求解析url，这对于别人的网站不太友好。

### 实现细节：

首先进行url识别测试，针对比较复杂的网页url，我们参考了gpt生成的正则匹配进行优化，使得对于大部分的网页都能正确解析。

然后通过Jsoup来尝试获取标题，Jsoup是个爬虫工具，能够根据给定的url内容进行html内容解析，从爬取得到的document对象中获取对应的标题信息。

{{< collapse "分别对百度和微信文章进行识别测试" >}}
    
```JAVA
public static void main(String[] args) throws IOException {
    Connection connect = Jsoup.connect("http://www.baidu.com");
    Document document = connect.get();
    String title = document.title();
    System.out.println(title);
}
public static void main(String[] args) throws IOException {
    Connection connect = Jsoup.connect("https://mp.weixin.qq.com/s/GQGidprakfticYnbVYVYGQ");
    Document document = connect.get();
    String title = document.getElementsByAttributeValue("property", "og:title").attr("content");
    System.out.println(title);
}
```
{{< /collapse >}}

一般网页的标题都是title标签，但也有例外，比如微信文章的网页，因为它根本不存在title标签，它的标题存在于meta标签中。

也就是说对于不同的页面，标题存在不同的标签中，需要不同的解析方式，因此，只有将解析不出来的标题打日志，然后再去慢慢添加更多的解析方式。

我们也可以将标题的解析方式做成解析器，每个解析器串成一个链条，通用的解析器优先级更高，只有当解析器中的某个解析器解析出标题才返回。

这种解析器串起来的链条就是责任链模式，创建责任链的地方为工厂模式，不同的类实现不同的url解析方法，这就是策略模式，抽象类中的逻辑，类似于模板方法，将多个模式都用起来了。

### 搭建url解析框架

定义接口urlTitleDiscover，核心的接口是getContentTitleMap，其他方法是细分的获取步骤。

公共的逻辑都放在抽象类中，子类commonUrl和WxUrl就是不同的标题解析策略。

prioritiedUrlTitleDiscover是我们的策略类，同时也是组装责任链的工厂，调用它会顺序执行责任链，直到解析出url标题。

### 当消息中存在多个url时

我们可以并行进行解析，如果串行解析，将导致消息发送者在发送包含多条url的消息时，一直阻塞，影响体验。

也有一些特殊的情况，比如github这种网站，请求时间可能会很长，它决定了用户的等待上限，因此我们需要对它们进行熔断，也就是当超过2s还没拿到网站，就返回连接超时，表示这个网站可能访问不到了。

对于这种解析超时直接丢弃，其他请求继续并行的框架，我们可以使用juc中的CompletableFuture实现，

{{< collapse "通过正则表达式获取消息中的链接" >}}
    
```JAVA
 //通过正则表达式获取消息中的链接
    private static final Pattern PATTERN = Pattern.compile("((http|https)://)?(www.)?([\\w_-]+(?:(?:\\.[\\w_-]+)+))([\\w.,@?^=%&:/~+#-]*[\\w@?^=%&/~+#-])?");

    @Nullable
    @Override
    public Map<String, UrlInfo> getUrlContentMap(String content) {
        if (StrUtil.isBlank(content)) {
            return new HashMap<>();
        }
        List<String> matchList = ReUtil.findAll(PATTERN, content, 0);

        //通过CompletableFuture.supplyAsync实现并行请求
        List<CompletableFuture<Pair<String, UrlInfo>>> futures = matchList.stream().map(match -> CompletableFuture.supplyAsync(() -> {
            UrlInfo urlInfo = getContent(match);
            return Objects.isNull(urlInfo) ? null : Pair.of(match, urlInfo);
        })).collect(Collectors.toList());
        CompletableFuture<List<Pair<String, UrlInfo>>> future = FutureUtils.sequenceNonNull(futures);
        //结果组装
        return future.join().stream().collect(Collectors.toMap(Pair::getFirst, Pair::getSecond, (a, b) -> a));
    }

```
{{< /collapse >}}




### 为什么会选择CompletableFuture？

对业界广泛流行的解决方案做了横向调研，主要包括Future、CompletableFuture、RxJava、Reactor。它们的特性对比如下：

![Future、CompletableFuture、RxJava、Reactor对比](/posts_imgs/CompletableFuture-compare-with-others.webp)


- **可组合**：可以将多个依赖操作通过不同的方式进行编排，例如CompletableFuture提供thenCompose、thenCombine等各种then开头的方法，这些方法就是对“可组合”特性的支持。
- **操作融合**：将数据流中使用的多个操作符以某种方式结合起来，进而降低开销（时间、内存）。
- **延迟执行**：操作不会立即执行，当收到明确指示时操作才会触发。例如Reactor只有当有订阅者订阅时，才会触发操作。
- **回压**：某些异步阶段的处理速度跟不上，直接失败会导致大量数据的丢失，对业务来说是不能接受的，这时需要反馈上游生产者降低调用量。

Future用于表示异步计算的结果，只能通过阻塞或者轮询的方式获取结果，而且不支持设置回调方法，Java 8之前若要设置回调一般会使用guava的ListenableFuture，回调的引入又会导致臭名昭著的回调地狱。

CompletableFuture对Future进行了扩展，可以**通过设置回调的方式处理计算结果，同时也支持组合操作，支持进一步的编排，同时一定程度解决了回调地狱的问题**。

CompletableFuture实现了两个接口：Future、CompletionStage。Future表示异步计算的结果，CompletionStage用于表示异步执行过程中的一个步骤（Stage），这个步骤可能是由另外一个CompletionStage触发的，随着当前步骤的完成，也可能会触发其他一系列CompletionStage的执行。从而我们可以根据实际业务对这些步骤进行多样化的编排组合，CompletionStage接口正是定义了这样的能力，我们可以通过其提供的thenAppy、thenCompose等函数式编程方法来组合编排这些步骤。

RxJava与Reactor显然更加强大，它们提供了更多的函数调用方式，支持更多特性，但同时也带来了更大的学习成本。而我们最需要的特性就是“异步”、“可组合”，综合考虑后，我们选择了学习成本相对较低的CompletableFuture。


> 参考文献
> [1] https://www.yuque.com/snab/mallchat/nouoyvf80k9rar8n
> [2] https://mp.weixin.qq.com/s/GQGidprakfticYnbVYVYGQhttps://mp.weixin.qq.com/s/GQGidprakfticYnbVYVYGQ


---END---