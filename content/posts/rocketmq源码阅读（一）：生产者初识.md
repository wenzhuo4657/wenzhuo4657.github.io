+++
date = '2025-06-29T10:00:14+08:00'
draft = false
title = 'rocketmq源码阅读（一）：生产者初识'
+++













#  源码阅读（一）




## 搭建环境


rocketmq的java-sdk并没有什么特殊的构建的，直接加载maven即可。



节点部署：
[Docker 部署 RocketMQ \| RocketMQ](https://rocketmq.apache.org/zh/docs/quickStart/02quickstartWithDocker)

面板部署：
[apache/rocketmq-dashboard](https://github.com/apache/rocketmq-dashboard)


hhh，遵从官网部署，点点点就可以了。





## 生产者初识


阅读源码首先要明确的目标，或者说我们想要了解什么？


在我看来生产者的职责分为以下几个
- 1,与`namesrv`沟通，维护本地的`broker`通信队列
- 2,发送`topic`消息到远程`broker`
- 3,根据响应结果判断是否发送成功，以及失败情况下的策略






每一个生产者对于`broker`节点来说是消息的来源，但是`broker`并不关心究竟有多少个生产者在为它提供服务，对于消费者来说同样如此！



首先定位到源码示例

![]([https://blog.wenzhuo4657.org/img/2025/04/ce5a9ce6d095400e1dfefe77486c2729.png)








示例的逻辑并不复杂，但问题是怎么处理的？尝试追溯`DefaultMQProducer`,找到客户端的sdk模块`rocketmq-client`




观察`DefaultMQProducer`的父类路径发现`extends ClientConfig implements MQProducer`,对于`MQProducer`没啥好说的，定义了生产者方法规范，而且没有注释。小白一只的我将目光对准`ClientConfig`,



`private String clientIP = NetworkUtil.getLocalAddress();`  寻找ip的方法。

`private String instanceName = System.getProperty("rocketmq.client.name", "DEFAULT");`  生产者名称

```
    # 客户端id，这个要注意与生产者名称区分，在我看来是类似与多元组一样的， ip+instanceName+......
   
    public String buildMQClientId() {
        StringBuilder sb = new StringBuilder();
        sb.append(this.getClientIP());

        sb.append("@");
        sb.append(this.getInstanceName());
        if (!UtilAll.isBlank(this.unitName)) {
            sb.append("@");
            sb.append(this.unitName);
        }

        if (enableStreamRequestType) {
            sb.append("@");
            sb.append(RequestType.STREAM);
        }

        return sb.toString();
    }
    
```









需要注意的是对于修改`clentIP`可以直接通过`producer.setClientIP(DEFAULT_NAMESRVADDR);`,`clientID`则并没有这个字段，他是在生产者启动时自动生成的的一个标识符，或许可以这么说 `ip+instanceName`表示了这个主机上的所有rocketmq的消费者和生产者。

追溯`buildMQClientId`即可发现，start的调用链路中存在这个方法。





### 与`namesrv`沟通，维护本地的`broker`通信队列



`DefaultMQProducer#start() -> DefaultMQProducerImpl#start() -> MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHoo)  -> MQClientInstance # new MQClientInstance(ClientConfig clientConfig, int instanceIndex, String clientId, RPCHook rpcHook)`


追溯到`MQClientInstance`的初始化，在这里类中可以看到非常重要的几个属性
```

## broker节点列表 ket为id,value为 ip地址
private final ConcurrentMap<String, HashMap<Long, String>> brokerAddrTable = new ConcurrentHashMap<>();

## 心跳检查开关，源码中显示用于管理broker节点的连接是否实效
private boolean enableHeartbeatChannelEventListener = true;

##  消费者状态代理
this.consumerStatsManager

##  netty连接客户端配置
private final NettyClientConfig nettyClientConfig;
```



断点查看参数后发现`new MQClientInstance(ClientConfig clientConfig, int instanceIndex, String clientId, RPCHook rpcHook)`初始化的是远程namesrv的客户端。
![[Pasted image 20250409182100.png]]





#### namesrv的配置

结论： 存在不一致则直接覆盖更新，

```
NettyRemotingClient{
private final AtomicReference<List<String>> namesrvAddrList = new AtomicReference<>();//这个原子引用在方法中充当了互斥锁

 public void updateNameServerAddressList(List<String> addrs) {  
        List<String> old = this.namesrvAddrList.get();  
        boolean update = false;  
  
  
//        1,如果存在不一致，则全部更新  
        if (!addrs.isEmpty()) {  
            if (null == old) {//olg不存在  
                update = true;  
            } else if (addrs.size() != old.size()) {//数量不一致  
                update = true;  
            } else {  
                for (String addr : addrs) {  
                    if (!old.contains(addr)) {//存在差异  
                        update = true;  
                        break;                    }  
                }  
            }  
  
            if (update) {  
            this.namesrvAddrList.set(addrs);
            } 
        }  
    }


}
```



此处引申出两个新问题,
1,如何复用`MQClientInstance `?
调用链路中直接上级是
`MQClientManager`,关键成员变量为
```
    private ConcurrentMap<String/* clientId */, MQClientInstance> factoryTable =
        new ConcurrentHashMap<>();
```




2,`namesrv`存在更新，看上去是MQClientInstance 重新初始化的缘故？那么为什么会重新初始化？`namesrv`应当写死在配置文件当中，但此处似乎允许动态更新？


追溯`ClientConfig`信息的来源发现在`DefaultMQProducerImpl`成员变量
```
    private final DefaultMQProducer defaultMQProducer;
public class DefaultMQProducer extends ClientConfig implements MQProducer {
....
}
```

从接口角度上来说，DefaultMQProducer确实是生产者实现，但可以将`DefaultMQProducerImpl`理解为生产者的启动类，而并非实现类。

此外这个实现类会默认使用环境变量（或者是其他配置），然后使用编码角度的上层配置覆盖，这也是常规的编码>环境变量的实现。




####  维护broker队列


追溯发送消息的方法找到关键方法`sendKernelImpl`
```
class DefaultMQProducerImpl{
 private SendResult sendKernelImpl(。。。 ) {
        
String brokerName = this.mQClientFactory.getBrokerNameFromMessageQueue(mq);
String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(brokerName);

        if (null == brokerAddr) {
            tryToFindTopicPublishInfo(mq.getTopic());
            brokerName = this.mQClientFactory.getBrokerNameFromMessageQueue(mq);
            brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(brokerName);
            }
        
}
```



1,获取brokerAddr地址时并不会检验是否有效，只要有对应缓存即可
2,` tryToFindTopicPublishInfo(mq.getTopic());` 会在本地没有对应topic条目时更新


基于以上两点可以看出，默认实现里对于broker的维护似乎是通过发送消息错误时重新拉取实现的？

```
DefaultMQProducerImpl#sendKernelImpl  -》
this.mQClientFactory.getMQClientAPIImpl().sendMessage   -》


```



在`sendmessgae`中发送消息有三种走向，通过`switch`区分
```
   switch (communicationMode) {
            case ONEWAY:  //单向发送
                this.remotingClient.invokeOneway(addr, request, timeoutMillis);

            case ASYNC:  //异步发送
            
                this.sendMessageAsync(addr, brokerName, msg, timeoutMillis - costTimeAsync, request, sendCallback, topicPublishInfo, instance,
                    retryTimesWhenSendFailed, times, context, producer);
    
            case SYNC:   //同步发送
            
                return this.sendMessageSync(addr, brokerName, msg, timeoutMillis - costTimeSync, request);
            default:
                assert false;
                break;
        }
```

单向发送： 不会判断是否发送成功
异步发送/同步发送：会根据ack或者响应超时等其他指标判断消息是否发送成功

因此需要注意，假设使用了单向发送，那么先前的推论`根据相应判断broker是否有效`就不成立，或者说不会单单根据这个判断broker是否有效。


##### 心跳模式
重新阅读源码，追溯`brokerAddrTable`调用发现心跳模式，
```
 if (clientConfig.isEnableHeartbeatChannelEventListener()) {
            channelEventListener = new ChannelEventListener() {
                
                private final ConcurrentMap<String, HashMap<Long, String>> brokerAddrTable = MQClientInstance.this.brokerAddrTable;

                @Override
                public void onChannelActive(String remoteAddr, Channel channel) {
                    for (Map.Entry<String, HashMap<Long, String>> addressEntry : brokerAddrTable.entrySet()) {
                        for (Map.Entry<Long, String> entry : addressEntry.getValue().entrySet()) {
                            String addr = entry.getValue();
                            if (addr.equals(remoteAddr)) {
                                long id = entry.getKey();
                                String brokerName = addressEntry.getKey();
                                if (sendHeartbeatToBroker(id, brokerName, addr, false)) {
                                    rebalanceImmediately();
                                }
                                break;
                            }
                        }
                    }
                }
            }
```



核心判断` if (addr.equals(remoteAddr)) {。。。  break;}`  

结合心跳模式的判断，
1,此处应当为borker地址的监听判断，或者说维护。
2,这里仍然不是拉去broker的地址



##### 定时任务： 清除离线broker

再一次追溯调用找到定时任务，每秒执行一次
```
MQClientInstance#startScheduledTask
		this.scheduledExecutorService.scheduleAtFixedRate(() -> {
            try {
                MQClientInstance.this.cleanOfflineBroker();
                MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
            } catch (Throwable t) {
                log.error("ScheduledTask sendHeartbeatToAllBroker exception", t);
            }
        }, 1000, this.clientConfig.getHeartbeatBrokerInterval(), TimeUnit.MILLISECONDS);
```

```    
MQClientInstance.this.cleanOfflineBroker();  -》
this.isBrokerAddrExistInTopicRouteTable(addr)
```


该方法内部依赖`TopicRouteData`的维护，并且仅仅判断是否存在对应的`broker`地址，而不关心是否有对应topic
```
 private boolean isBrokerAddrExistInTopicRouteTable(final String addr) {
        for (Entry<String, TopicRouteData> entry : this.topicRouteTable.entrySet()) {
            TopicRouteData topicRouteData = entry.getValue();
            List<BrokerData> bds = topicRouteData.getBrokerDatas();
            for (BrokerData bd : bds) {
                if (bd.getBrokerAddrs() != null) {
                    boolean exist = bd.getBrokerAddrs().containsValue(addr);
                    if (exist)
                        return true;
                }
            }
        }
```


根据方法的注释，可以判断`TopicRouteData`的数据是需要作为依据存在，准确性要比`brokerAddrTable` 高。


##### updateTopicRouteInfoFromNameServer

更新路由拉区信息，无同步
```
（final String topic, boolean isDefault,
        DefaultMQProducer defaultMQProducer)
```


因此该方法用于更新指定生产者下的topic的地址，


调用时机： 
- 1, sendKernelImpl下没有topic对应的`MASTER_ID`的节点时。
- 


阅读源码发现，生产者作用似乎仅仅是指定了队列数量，但是其他数量都使用了当前MQClientInstance实例的数据，那么就引申出一个新问题，就是生产者和实例的对应关系？MQClientInstance似乎可以复用，在前面的方法调用当中存在`getOrCreateMQClientInstance`，仅仅从方法名称中也可以发现似乎存在某个变量维护着一个公共的`MQClientInstance``？



##### MQClientManager和MQClientInstance

首先说结论，这两对象都是能复用就复用，不会做额外的判断。


MQClientManager在调用时，直接使用的是静态成员变量
`   private static MQClientManager instance = new MQClientManager();`

MQClientInstance通过上面的静态成员变量维护了一个成员变量，
```
   private ConcurrentMap<String/* clientId */, MQClientInstance> factoryTable =
        new ConcurrentHashMap<>();
```



回顾clientiD的创建方法
```
    public String buildMQClientId() {
        StringBuilder sb = new StringBuilder();
        sb.append(this.getClientIP());

        sb.append("@");
        sb.append(this.getInstanceName());
        if (!UtilAll.isBlank(this.unitName)) {
            sb.append("@");
            sb.append(this.unitName);
        }

        if (enableStreamRequestType) {
            sb.append("@");
            sb.append(RequestType.STREAM);
        }

        return sb.toString();
    }
```


四元组为： ip： 实例名称: 单元名称： 流式请求



尝试使用相同clientId后成功启动。



##### TopicRouteInfo
目前来讲brokerAddrTable有只有一个直接来源`topicRouteTable`



追溯调用找到关键方法`MQClientInstance#updateTopicRouteInfoFromNameServer`

```
public boolean updateTopicRouteInfoFromNameServer(，，，）{
 topicRouteData = this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(clientConfig.getMqClientApiTimeout());
 // 中间省略的操作大致作用为old、young合并
this.topicRouteTable.put(topic, cloneTopicRouteData);
}
```



```
追溯找到路由信息的请求定义
InterruptedException, RemotingTimeoutException, RemotingSendRequestException, RemotingConnectException {  
    GetRouteInfoRequestHeader requestHeader = new GetRouteInfoRequestHeader();  
    requestHeader.setTopic(topic);  
    RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.GET_ROUTEINFO_BY_TOPIC, requestHeader);
```


注意核心请求code
```
RequestCode.GET_ROUTEINFO_BY_TOPIC,

public static final int GET_ROUTEINFO_BY_TOPIC = 105;
```

##### 发送消息失败


发送消息失败，会抛出自定义错误`RemotingException e` 交给`DefaultMQProducerImpl#sendDefaultImpl`的catch处理，断点查看得知，在默认情况下，会尝试隔离broker

```
} catch (RemotingException e) {  
    endTimestamp = System.currentTimeMillis();  
    if (this.mqFaultStrategy.isStartDetectorEnable()) {  
        // Set this broker unreachable when detecting schedule task is running for RemotingException.  
        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true, false);  
    } else {  
        // Otherwise, isolate this broker.  
        在内部方法的调用链路中，发现他会重新检测broker的状态，
        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true, true);  
    }
```

### 总结

生产者网络通信： `remotingClient`
生产者通信api封装： `MQClientAPIImpl`
生产者动态信息保存： `MQClientInstance`
生产者动态信息- 经理：`MQClientManager`
默认生产者实现：`DefaultMQProducerImpl`
默认生产者的定义： `DefaultMQProducer`




初始化`remotingClient`会以当前编码配置为主。
```
//        1,如果存在不一致，则全部更新  
        if (!addrs.isEmpty()) {  
            if (null == old) {//olg不存在  
                update = true;  
            } else if (addrs.size() != old.size()) {//数量不一致  
                update = true;  
            } else {  
                for (String addr : addrs) {  
                    if (!old.contains(addr)) {//存在差异  
                        update = true;  
                        break;                    }  
                }  
            }
```

broker的更新和维护

1,心跳模式
2,定时任务根据路由信息更新
3,不存在对应的topic信息时，会发送code为105的请求到namesrv中
4,默认情况下，发送消息失败会尝试隔离对应的broker

![](https://blog.wenzhuo4657.org/img/2025/04/20244c5203f9570357a426db7ebc3a23.png)







客户端所用的code类
`org.apache.rocketmq.remoting.protocol.RequestCode`

```

```
