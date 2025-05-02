---
title: "[Tech]IM项目中的消息交互设计"
date: 2025-02-17T14:55:07+08:00
lastmod: 2025-02-17T14:55:07+08:00
author: ["Fubos"]

categories:
- tech

tags:
- IM
- 消息交互

keywords:
- IM
- 消息交互

description: "在IM系统的消息交互设计中，我们支持了文件，视频，图片，语音等消息类型，并支持已发送消息的回复和跳转以及消息的点赞和点踩，用户可以艾特群成员，并弹出消息提醒对应的群成员有艾特消息。用户还可以在时间线展示的情况下查看历史消息列表，以及对消息进行撤回操作。" # 文章描述，与搜索优化相关
summary: "在IM系统的消息交互设计中，我们支持了文件，视频，图片，语音等消息类型，并支持已发送消息的回复和跳转以及消息的点赞和点踩，用户可以艾特群成员，并弹出消息提醒对应的群成员有艾特消息。用户还可以在时间线展示的情况下查看历史消息列表，以及对消息进行撤回操作。" # 文章简单描述，会展示在主页
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
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

在IM系统的消息交互设计中，我们支持了文件，视频，图片，语音等消息类型，并支持已发送消息的回复和跳转以及消息的点赞和点踩，用户可以艾特群成员，并弹出消息提醒对应的群成员有艾特消息。用户还可以在时间线展示的情况下查看历史消息列表，以及对消息进行撤回操作。



## 支持多类型消息-文件、视频、图片、语音

在用户发送消息界面，用户可以发送多种消息类型，比如文件、视频、图片、语音等消息，我们可以将这些消息抽象起来，将它们都看作是文件，并存储在oss的url中。

因此我们只需要添加一个上传的接口，前端只需要记录上传这些类型的消息后得到的url，提交给后端即可。

![IM不同的消息类型展示](/posts_imgs/IM-Message-display.png)


其中的表设计中，包含两个重要的字段，一个是type，也就是指定消息是什么类型，一个是extra，放置不同类型消息的详情，如果是特别重要的消息，可以通过设置关联表的方式，扩展出去，比如红包类型消息。

![IM消息类型所对应的表字段](/posts_imgs/IM-Message-type.png)

不同类型的消息在被回复时效果不同，因此可以设计一个策略模式，定义消息的几个不同的策略。在保存消息的时候，先保存基础的消息数据，再根据不同的消息类型保存自己特殊的元素。



```java
    @Override
    @Transactional
    public Long sendMsg(ChatMessageReq request, Long uid) {
        check(request, uid);
        //根据消息类型来获取不同的策略类
        AbstractMsgHandler<?> msgHandler = MsgHandlerFactory.getStrategyNoNull(request.getMsgType());
        Long msgId = msgHandler.checkAndSaveMsg(request, uid);
        //发布消息发送事件
        applicationEventPublisher.publishEvent(new MessageSendEvent(this, msgId));
        return msgId;
    }

    public class MsgHandlerFactory {
        public static final Map<Integer,AbstractMsgHandler> STRATEGY_MAP = new HashMap<>();
        public static void register(Integer code,AbstractMsgHandler strategy){
            STRATEGY_MAP.put(code,strategy);
        }
        public static AbstractMsgHandler getStrategyNoNull(Integer code){
            AbstractMsgHandler strategy = STRATEGY_MAP.get(code);
            AssertUtil.isNotEmpty(strategy, CommonErrorEnum.PARAM_INVALID);
            return strategy;
        }
    }
```

### 艾特群成员

由于前端存了一个成员列表库，可以将其作为艾特好友的数据源，用户艾特一次只需要拉取一次群成员列表。

后续的更新都依赖于群成员列表上下线的推送，以及发送新消息的人，都会触发前端用户库的更新，这样库中就含有所有活跃成员的信息，前端可以自行根据用户库去做名称匹配，本地运行速度也更快。

我们可以通过艾特列表，消息页面，成员页面去进行好友的艾特。前端艾特后，保存一个被艾特的所有好友uidlist，传给后端，这样后端就非常方便。

我们的消息展示比较简单，本质就是一串文本，不像qq那样能够反解析出用户信息。如果要反解析的话也容易，只需要根据返回的展示结果，给出艾特成员的uid和用户名，这样前端就能很容易地去匹配出艾特用户，高亮出用户信息。


## 消息列表交互

### 消息发送

我们统一将消息推送，消息列表，消息发送3个功能都复用消息详情展示方法getMsgRespBatch。在发消息时，需要用户态，通过拦截器解析出uid，也就是RequestHolder.get().getUid()获得。

在发送消息时，调用service的sendMsg消息发送，结束后返回消息id，然后复用消息展示的接口getMsgResp将消息组装成前端可展示的消息对象返回给前端，这样用户发完消息后就能够通过接口直接快速看见自己的消息，不用等待后台的主动推送了。

```java
    @Override
    public ChatMessageResp getMsgResp(Message message, Long receiveUid) {
        return CollUtil.getFirst(getMsgRespBatch(Collections.singletonList(message), receiveUid));
    }

    private List<ChatMessageResp> getMsgRespBatch(List<Message> messages, Long receiveUid) {
        if (CollectionUtil.isEmpty(messages)) {
            return new ArrayList<>();
        }
        //查询消息标志
        List<MessageMark> msgMark = messageMarkDao.getValidMarkByMsgIdBatch(messages.stream().map(Message::getId).collect(Collectors.toList()));
        return MessageAdapter.buildMsgResp(messages, msgMark, receiveUid);
    }

    public static List<ChatMessageResp> buildMsgResp(List<Message> messages, List<MessageMark> msgMark,Long receiveUid){
        Map<Long, List<MessageMark>> markMap = msgMark.stream().collect(Collectors.groupingBy(MessageMark::getMsgId));
        return messages.stream().map(a->{
            ChatMessageResp resp = new ChatMessageResp();
            resp.setFromUser(buildFromUser(a.getFromUid()));
            resp.setMessage(buildMessage(a,markMap.getOrDefault(a.getId(),new ArrayList<>()),receiveUid));
            return resp;
        })
                .sorted(Comparator.comparing(a->a.getMessage().getSendTime()))//排好序发给前端
                .collect(Collectors.toList());
    }
```

### 消息回复和跳转

在回复消息的时候，本质也是发消息，只不过带上了一个回复消息的id。

#### 这时如何做到点击回复消息，能跳转到原消息的位置呢？

分两种情况讨论

1）原消息已拉取：如果原始消息在之前就已经被客户端拉取加载过了，那点击回复消息的时候就已经获取到回复消息的id了，此时前端直接根据消息id跳转到原消息即可。

2）原消息未拉取：如果原消息不存在客户端中，想要跳转的话，就需要去后端加载历史消息，直到加载出原消息为止。但是重点是不能让原消息无限制的跳转，也就是当用户翻到历史记录中的远古消息，并艾特回复时，大家都去点击原文，这样会直接把服务器拉爆。

我们可以设置一定的规则来判断原消息是否可以跳转。比如两条消息间隔超过多少条时，就不让跳转了。此时又出现了问题，那就是具体啥时候去查间隔条数呢？此时也分两种情况。

1）拉取消息时计算：当用户每次拉取消息列表时，判断如果有回复消息，就去计算间隔条数，就像是读放大场景，当有一千个用户拉取时，就需要算一千次，这种方法对于服务器不友好。

2）发送消息时计算：也就是谁回复，谁计算，计算花费的时间由消息发送者承担，这样只需要计算一次消息间隔，然后记录在数据库中，以后读取消息就不用再计算了，只需要获取数据库中该消息对应的消息间隔数即可，当超过一定条数时，不进行跳转。节省服务器性能。

#### 当获取到原消息id后，如何加载原消息呢？

首先后端会返回消息间隔的条数，前端需要计算与原消息间还有多少条消息未加载，然后直接复用后端的历史消息列表，将size=间隔记录条数为参数，即可直接一次性加载到原消息。

### 消息推送
用户发送消息，主要是将消息入库，通过发送spring事件的方式，进行一层解耦。

事件的监听者会将消息id组装成前端展示的内容，推送给前端所有在线用户。监听者里面调用了websocket的发送所有在线用户消息的方法，底层通过一个异步线程池的批量发送实现。
```java
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT,classes = MessageSendEvent.class,fallbackExecution = true)
    public void messageRoute(MessageSendEvent event){
        Long msgId = event.getMsgId();
        mqProducer.sendSecureMsg(MQConstant.SEND_MSG_TOPIC,new MsgSendMessageDTO(msgId),msgId);
    }

    /**
     * 发送可靠消息，在事务提交后保证发送成功
     * @param topic
     * @param body
     * @param key
     */
    @SecureInvoke(async = false)
    public void sendSecureMsg(String topic,Object body,Object key){
        Message<Object> build = MessageBuilder
                .withPayload(body)
                .setHeader("KEYS", key)
                .build();
        rocketMQTemplate.send(topic,build);
    }
```

## 历史消息列表

用户进入页面，首先会拉取一页最新的消息，访问的就是历史消息列表。

使用游标翻页能接近深翻页的问题，且适合不翻页的情况，也就很适合消息列表的查看。

在我们的项目中，设计了一个用户访问消息列表的接口，入参继承了一个基础的游标翻页对象，并且还有一个参数roomId,用于表明是哪个会话下的消息。

```java
    @GetMapping("/public/msg/page")
    @ApiOperation("消息列表")
    public ApiResult<CursorPageBaseResp<ChatMessageResp>> getMsgPage(@Valid ChatMessagePageReq request) {
        CursorPageBaseResp<ChatMessageResp> msgPage = chatService.getMsgPage(request, RequestHolder.get().getUid());
        filterBlackMsg(msgPage);
        return ApiResult.success(msgPage);
    }

    public class ChatMessagePageReq extends CursorPageBaseReq {
        @NotNull
        @ApiModelProperty("会话id")
        private Long roomId;
    }    
```

```java
@Data
@ApiModel("游标翻页请求")
@AllArgsConstructor
@NoArgsConstructor
public class CursorPageBaseReq {

    @ApiModelProperty("页面大小")
    @Min(0)
    @Max(100)
    private Integer pageSize = 10;

    @ApiModelProperty("游标（初始为null，后续请求附带上次翻页的游标）")
    private String cursor;

    //将前端的游标请求转换为内部数据库的请求,将当前的起始页转为分页第一页，相当于limit 0，再向后查一页，实现翻页
    //isSearchCount为false表示不查结果的总数，对于数据量大的情况下有好处
    public Page plusPage() {
        return new Page(1, this.pageSize,false);
    }

    @JsonIgnore
    public Boolean isFirstPage() {
        return StringUtils.isEmpty(cursor);
    }
}

```

查询方法较为简单，有两个功能，一个是先在数据库中查出一页的消息，然后再加载成前端需要展示的结构。
```java
    @Override
    public CursorPageBaseResp<ChatMessageResp> getMsgPage(ChatMessagePageReq request, Long receiveUid) {
        //用最后一条消息的id，来限制被踢出的人能看到的最大一条消息
        Long lastMsgId = getLastMsgId(request.getRoomId(), receiveUid);
        // 游标查询原始信息
        CursorPageBaseResp<Message> cursorPage = messageDao.getCursorPage(request.getRoomId(), request, lastMsgId);
        if(cursorPage.isEmpty()){
            return CursorPageBaseResp.empty();
        }
        // 对获取到的消息详情进行组装
        return CursorPageBaseResp.init(cursorPage,getMsgRespBatch(cursorPage.getList(),receiveUid));
    }
```

1.游标查询：通过游标分页的工具类就很方便的实现这个功能，我们的消息目前都在一个表，主键自增，暂时可以通过唯一的id来做游标查询，再限制会话id，以及限制消息需要是一条正常消息。不需要查询`禁用`的消息，或`撤回`的消息。并设置查询消息的方向，是往前面查还是往后面查。
```java
    public CursorPageBaseResp<Message> getCursorPage(Long roomId, CursorPageBaseReq request, Long lastMsgId) {
        return CursorUtils.getCursorPageByMysql(this, request, wrapper->{
            wrapper.eq(Message::getRoomId,roomId);
            wrapper.eq(Message::getStatus, NormalOrNoEnum.NORMAL.getStatus());
            wrapper.le(Objects.nonNull(lastMsgId),Message::getId,lastMsgId);
        },Message::getId);
    }

    public static <T> CursorPageBaseResp<T> getCursorPageByMysql(IService<T> mapper, CursorPageBaseReq request, Consumer<LambdaQueryWrapper<T>> initWrapper, SFunction<T,?> cursorColumn){
    //游标字段类型
    Class<?> cursorType = LambdaUtils.getReturnType(cursorColumn);
    LambdaQueryWrapper<T> wrapper = new LambdaQueryWrapper<>();
    //wrapper是查询的构造对象
    //额外条件，将当前用户uid作为额外条件
    initWrapper.accept(wrapper);
    //游标条件
    if(StrUtil.isNotBlank(request.getCursor())){
        wrapper.lt(cursorColumn,parseCursor(request.getCursor(),cursorType));
    }
    //游标方向
    wrapper.orderByDesc(cursorColumn);
    //plusPage表示查询后一页记录
    Page<T> page = mapper.page(request.plusPage(), wrapper);

    //取出游标，先判断非空，每次翻页都以上次的最后一个记录作为起始游标
    String cursor = Optional.ofNullable(CollectionUtil.getLast(page.getRecords()))
            .map(cursorColumn)
            .map(CursorUtils::toCursor)
            .orElse(null);
    //判断是否最后一页，通过当前查到的记录数是否等于请求的记录数来判断，这样可能会存在当前查询的结果刚好是最后一页，同时还是要查询记录数的倍数。这导致还需要多查一页才能判断是最后一页。
    //可以通过多查一条记录，但不将该记录展示，实现一次查询就能判断是否为最后一页
    Boolean isLast = page.getRecords().size() != request.getPageSize();
    return new CursorPageBaseResp<>(cursor,isLast,page.getRecords());
    }
```

2.消息组装将消息列表查到的原始数据，再聚合上`用户信息`，`消息标记（点赞点踩）`，`消息url识别`等前端需要展示的信息就行了。消息列表的展示就很像`商品的列表页`，也需要加载很多的依赖的资源。如果发现缺啥就一条一条的加载啥其实是效率很低的。

```java
public class ChatMessageResp {
    @ApiModelProperty("发送者消息")
    private UserInfo fromUser;
    @ApiModelProperty("消息详情")
    private Message message;

    @Data
    public static class UserInfo{
        @ApiModelProperty("用户id")
        private Long uid;
    }
    @Data
    public static class Message{
        @ApiModelProperty("消息id")
        private Long id;
        @ApiModelProperty("房间id")
        private Long roomId;
        @ApiModelProperty("消息发送时间")
        private Date sendTime;
        @ApiModelProperty("消息类型 1正常文本 2.撤回消息")
        private Integer type;
        @ApiModelProperty("消息内容不同的消息类型，内容体不同，见https://www.yuque.com/snab/mallcaht/rkb2uz5k1qqdmcmd")
        private Object body;
        @ApiModelProperty("消息标记")
        private MessageMark messageMark;
    }
    @Data
    public static class MessageMark {
        @ApiModelProperty("点赞数")
        private Integer likeCount;
        @ApiModelProperty("该用户是否已经点赞 0否 1是")
        private Integer userLike;
        @ApiModelProperty("举报数")
        private Integer dislikeCount;
        @ApiModelProperty("该用户是否已经举报 0否 1是")
        private Integer userDislike;
    }
}

```

一般都需要批量去查询出所有的资源，然后丢进适配器进行组装。

```java
    public static List<ChatMessageResp> buildMsgResp(List<Message> messages, List<MessageMark> msgMark,Long receiveUid){
        Map<Long, List<MessageMark>> markMap = msgMark.stream().collect(Collectors.groupingBy(MessageMark::getMsgId));
        return messages.stream().map(a->{
            ChatMessageResp resp = new ChatMessageResp();
            resp.setFromUser(buildFromUser(a.getFromUid()));
            resp.setMessage(buildMessage(a,markMap.getOrDefault(a.getId(),new ArrayList<>()),receiveUid));
            return resp;
        })
                .sorted(Comparator.comparing(a->a.getMessage().getSendTime()))//排好序发给前端
                .collect(Collectors.toList());
    }
```
## 消息撤回
我们通过对消息类型进行标识，标识不同的消息，1正常文本 2.撤回消息，在展示时，根据不同的消息类型，选择不同的展示。

## 时间线展示
前端的时间展示，是仿微信的时间展示。是根据每20条消息，或者是上下两条消息间隔5分钟就展示时间。

主要的目的就是，时间跨度大才展示时间。

## 表情包功能

我们的功能和微信类似，存储的适合考虑复用性，只在用户第一次上传时，保存到minio中，后续发送时直接指向对应的图片。当其他用户也保存了该图片文件时，也是指向同一地址，减少冗余。

对于较大的表情包，在上传时，自动通过算法对表情包进行压缩，减少存储空间占用。

保存表情包的时间点主要有两个，一个是用户自己主动上传，一个是添加其他人发送的表情包，还有一个是将其他人发送的图片保存成表情包，这个过程中，前端会拉取图片到本地，再自动进行压缩算法保存到minio中。

## 消息点赞点踩

我们可以把点击的动作，包括点赞和点踩，统称为消息的标记，消息标记之后，还需要推送给前端，让所有用户实时看到标记值变化。

数据库设计：将点赞和点踩抽象成对消息的标记，并专门为消息做一张标记表。每条消息对应多条消息标记的记录。

接口设计：入参包括markType：标记消息类型1点赞2点踩，msgId：消息id，actType：动作类型1确认2取消。
```java
public class ChatMessageMarkReq {
    @NotNull
    @ApiModelProperty("消息id")
    private Long msgId;

    @NotNull
    @ApiModelProperty("标记类型 1点赞 2举报")
    private Integer markType;

    @NotNull
    @ApiModelProperty("动作类型 1确认 2取消")
    private Integer actType;
}

```

实现细节：由于需要通知到所有在线用户，因此需要进行频控限制，防止出现乱刷接口的情况。
```java
    @PutMapping("/msg/mark")
    @ApiOperation("消息标记")
    @FrequencyControl(time = 10,count = 5,target = FrequencyControl.Target.UID)
    public ApiResult<Void> setMsgMark(@Valid @RequestBody ChatMessageMarkReq request){
        chatService.setMsgMark(RequestHolder.get().getUid(),request);
        return ApiResult.success();
    }
```

由于消息的标记涉及到对用户的消息进行操作，因此需要使用分布式锁来进行资源的隔离。在加上锁之后，就可以直接对消息进行标记，并发布一个消息标记事件。
```java
    @Override
    @RedissonLock(key = "#uid")
    public void setMsgMark(Long uid, ChatMessageMarkReq request) {
        AbstractMsgMarkStrategy strategy = MsgMarkFactory.getStrategyNoNull(request.getMarkType());
        switch (MessageMarkActTypeEnum.of(request.getActType())) {
            case MARK:
                strategy.mark(uid,request.getMsgId());
                break;
            case UN_MARK:
                strategy.unMark(uid,request.getMsgId());
                break;
        }
    }
```

传统的消息标记，通过查看历史有没有标记，有的话就改动标记，没有的话就新增一条标记记录。

但因为有点赞和点踩的互斥，使用if-else实现起来较复杂。

可以使用设计模式将消息的标记做成一个抽象类，包含标记和取消标记两种方法，并实现了两个子类，包括点赞和点踩，这样就能够覆盖2 * 2种动作。其中抽象类实现了基本功能，对于有扩展需求的，可以重写方法，自己实现。

比如点赞的时候需要点踩。我们重写`doMark`方法，直接调用父类公共方法实现`点赞`的标记，再去额外的调用点踩的`取消标记`方法。

```java
@Component
public class LikeStrategy extends AbstractMsgMarkStrategy{
    @Override
    protected MessageMarkTypeEnum getTypeEnum() {
        return MessageMarkTypeEnum.LIKE;
    }
    @Override
    public void doMark(Long uid,Long msgId){
        super.doMark(uid, msgId);
        //同时取消点踩的动作
        MsgMarkFactory.getStrategyNoNull(MessageMarkTypeEnum.DISLIKE.getType()).unMark(uid, msgId);
    }

}
@Component
public class DisLikeStrategy extends AbstractMsgMarkStrategy{

    @Override
    protected MessageMarkTypeEnum getTypeEnum() {
        return MessageMarkTypeEnum.DISLIKE;
    }

    @Override
    protected void doMark(Long uid, Long msgId) {
        super.doMark(uid, msgId);
        // 同时取消点赞的动作
        MsgMarkFactory.getStrategyNoNull(MessageMarkTypeEnum.LIKE.getType()).unMark(uid,msgId);
    }
}

```

具体的执行代码在抽象类中，
```java
    protected void exec(Long uid, Long msgId, MessageMarkActTypeEnum actTypeEnum) {
        Integer markType = getTypeEnum().getType();
        Integer actType = actTypeEnum.getType();
        MessageMark oldMark = messageMarkDao.get(uid, msgId, markType);
        if (Objects.isNull(oldMark) && actTypeEnum == MessageMarkActTypeEnum.UN_MARK) {
            //取消的类型，数据库一定有记录，没有的话直接跳过
            return;
        }
        //插入一条新消息，或者修改一条消息
        MessageMark insertOrUpdate = MessageMark.builder()
                .id(Optional.ofNullable(oldMark).map(MessageMark::getId).orElse(null))
                .uid(uid)
                .msgId(msgId)
                .type(markType)
                .status(transformAct(actType))
                .build();
        boolean modify = messageMarkDao.saveOrUpdate(insertOrUpdate);
        if (modify) {
            //修改成功，发送消息标记事件
            ChatMessageMarkDTO dto = new ChatMessageMarkDTO(uid, msgId, markType, actType);
            applicationEventPublisher.publishEvent(new MessageMarkEvent(this, dto));
        }
    }
```
当修改成功后，才会发放消息标记事件。消息标记的监听者，会执行几个操作
1. 判断消息标记到10条，给用户发徽章（幂等）
2. 推送给所有在线用户，消息标记改动了。
这里有一个细节，如果告诉前端消息被点赞后，由前端给标记数+1，这样遇到丢消息的情况时容易错乱，最好是后端计算好后，直接告诉后端目前的标记数量，哪怕某次消息数据丢失，最终还是能够趋于一致的。

---END---


> 参考文献

> https://cloud.tencent.com/document/product/269/2720

> https://www.yuque.com/snab/mallchat/ce2pu00d3m9iywpx
