---
title: elasticsearch源码分析(三)Discover模块
date: 2018-05-27 12:12:57
categories: 
- Elasticsearch源码分析专题
---

通过上一篇对Elasticsearch启动的分析，我们知道了ES启动的大致流程，还遗留下几个问题

* master选举是在什么模块进行的
* ES集群是如何进行Master选举的？
* ES是如何维护这些节点的？

要想进行Master选举，必然要有一套算法机制，以及节点之前的通信连接、判断节点存活状态等。

通过查阅官网资料，我们知道这些功能是在Elasticsearch的发现协议Discovery里面进行的，在官网上，Elasticsearch的Discovery Module有下面几种实现：

* Azure Classic Discovery：https://www.elastic.co/guide/en/elasticsearch/reference/5.6/modules-discovery-azure-classic.html

* EC2 Discovery：https://www.elastic.co/guide/en/elasticsearch/reference/5.6/modules-discovery-ec2.html#modules-discovery-ec2

*  Google Compute Engine Discovery：https://www.elastic.co/guide/en/elasticsearch/reference/5.6/modules-discovery-gce.html

* Zen Discovery：https://www.elastic.co/guide/en/elasticsearch/reference/5.6/modules-discovery-zen.html

  ​

#### 一、Zen Discovery模块介绍

这里基本上是官网的翻译，建议还是查看官网文档，翻译不准。。。

Zen Discovery是内置在elasticsearch的默认发现模块。它提供单播发现，但可扩展到支持云环境和其他形式的发现。

禅发现集成了其它模块，例如，节点之间的所有通信是使用transport模块。

它被分离成多个子模块，其解释如下：

##### 1.1 Ping

这是一个节点使用发现机制来查找其他节点的过程。

##### 1.2 Unicast 

单播发现需要一个主机列表，用于将作为GossipRouter。这些宿主可被指定为主机名或IP地址;指定主机名的主机每一轮Ping过程中解析为IP地址。请注意，如果您处于DNS解析度随时间变化的环境中，则可能需要调整[JVM安全设置](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/networkaddress-cache-ttl.html "Title")。

建议将单播主机列表维护为集群中符合主节点的节点列表。

单播发现提供以下设置和`discovery.zen.ping.unicast`前缀：

| 设置                    | 描述                                                         |
| ----------------------- | ------------------------------------------------------------ |
| `hosts`                 | 数组设置或逗号分隔的设置。每个值的形式应该是`host:port`或`host`（如果没有设置，`port`默认设置会`transport.profiles.default.port` 回落到`transport.tcp.port`）。请注意，IPv6主机必须放在括号内。默认为`127.0.0.1, [::1]` |
| `hosts.resolve_timeout` | 在每轮ping中等待DNS查找的时间量。指定为 [时间单位](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/common-options.html#time-units)。默认为5秒。 |

单播发现使用[传输](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/modules-transport.html)模块执行发现。

##### 1.3 master选举

作为Ping过程的一部分，集群的主节点要么当选要么加入假期。这是自动完成的。ping的默认超时为3秒

~~~sh
discovery.zen.ping_timeout（默认为3s）
~~~

如果在超时后没有做出决定，则重新启动ping程序。在缓慢或拥塞的网络中，在作出选举决定之前，三秒可能不足以让节点意识到其环境中的其他节点。在这种情况下，应该谨慎地增加超时时间，因为这会减慢选举进程。一旦一个节点决定加入一个现有的已形成的集群，它将发送一个加入请求给主设备（`discovery.zen.join_timeout`）的超时默认值是ping超时的20倍。

当主节点停止或遇到问题时，群集节点会再次启动ping并选择新的主节点。这种ping测试也可以作为防止（部分）网络故障的保护，其中一个节点可能会不公正地认为主站发生故障。在这种情况下，节点将简单地从其他节点听到关于当前活动的主节点的信息。

如果`discovery.zen.master_election.ignore_non_master_pings`是`true`，没有参与资格（节点，其中节点坪`node.master`是`false`）的主选期间忽略; 默认值是 `false`。

可以通过设置`node.master`来排除节点成为主节点`false`。

该`discovery.zen.minimum_master_nodes`套需要加入新当选主为了选举完成并当选节点接受其主控权掌握合格节点的最小数量。相同的设置控制应该成为任何活动集群一部分的活动主节点合格节点的最小数量。如果不满足这个要求，活动的主节点将下台，新的主节点选举将开始。

此设置必须设置为您的主要合格节点的[法定人数](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/important-settings.html#minimum_master_nodes)。建议避免只有两个主节点，因为两个法定人数是两个。因此，任何主节点的损失都将导致无法运行的群集。



##### 1.4 故障检测

有两个故障检测进程正在运行。第一种方法是通过主设备对群集中的所有其他节点进行ping操作，并验证它们是否处于活动状态。另一方面，每个节点都会主动确认它是否仍然存在或需要启动选举过程。

以下设置使用`discovery.zen.fd`前缀控制故障检测过程 ：

| 设置            | 描述                                                 |
| --------------- | ---------------------------------------------------- |
| `ping_interval` | 一个节点多久发作一次。默认为`1s`。                   |
| `ping_timeout`  | 等待ping响应需要多长时间，默认为 `30s`。             |
| `ping_retries`  | 有多少ping故障/超时会导致节点被视为失败。默认为`3`。 |

##### 1.5 群集状态更新Cluster state updates

主节点是群集中，可以使改变集群状态的唯一节点。主节点一次处理一个集群状态更新，状态改变和发布更新的到集群中的所有其他节点。每个节点接收发布消息，确认它，但还没有立即应用它。如果主节点没有接收来自节点确认的数量至少为discovery.zen.minimum_master_nodes，在时间（由受控discovery.zen.commit_timeout设置，默认值为30秒）内。节点集群状态改变被拒绝。

一旦足够的节点已作出回应，集群状态改变被提交然后消息将被发送到所有结点。然后节点然后进行新的群集状态适用于他们的内部状态。主节点等待所有节点响应，在去队列处理下一个状态更新之前，直到超时，超时时间是在discovery.zen.publish_timeout默认情况下设置为30秒，时间从发布开始时测量h超时设置可以通过动态的改变集群更新设置API

##### 1.6 无主块No master block

要使群集完全可操作，它必须具有活动的主节点和一些有主资格的节点，并且主资格的节点必须满足的数目必须满足`discovery.zen.minimum_master_nodes`设置的值。如果设置 `discovery.zen.no_master_block` ，那么设置控制在没有活动的主设备时应拒绝哪些操作。

该`discovery.zen.no_master_block`设置有两个有效选项：

| `all`   | 节点上的所有操作（即读取和写入操作）都将被拒绝。这也适用于api集群状态读取或写入操作，如get索引设置，put映射和集群状态api。 |
| ------- | ------------------------------------------------------------ |
| `write` | （默认）写入操作将被拒绝。基于最后一次已知的群集配置，读取操作将成功。这可能会导致部分读取过时的数据，因为此节点可能与群集的其余部分隔离。 |

该`discovery.zen.no_master_block`设置不适用于基于节点的apis（例如，群集统计信息，节点信息和节点统计信息apis）。对这些apis的请求不会被阻止，并且可以在任何可用的节点上运行。

##### 1.7 Fault Delection

用ping的方式来确定node是否在集群里面



#### 二、Discovery源码分析

##### 2.1 Discovery类图

![:\hexo\source\images\es\ZenDiscovery类图.pn](F:\hexo\source\images\es\ZenDiscovery类图.png)

##### 2.2 与Discovery相关的几个类

ZenDiscovery.java 模块的主类，也是启动这个模块的入口，由Node.java调用并初始化，几乎涵盖了全部的发现协议的逻辑，是一个高度内聚了类，它有一些成员变量，需要明白他们的意思：

* pingTimeout：取自discovery.zen.ping_timeout（默认为3s）允许调整选举时间来处理网络慢或拥塞的情况（更高的值确保更少的失败机会）
* joinTimeout：取自discovery.zen.join_timeout（默认值为ping超时的20倍）。当一个新的node加入集群时，将会发个join的request到master，这个request的timeout即joinTimeout。
* joinRetryAttempts：join重试的次数，默认为3次。
* joinRetryDelay：重试的间隔，默认为100ms。
* maxPingsFromAnotherMaster：容忍其他master发出的,在强制其他或是本地master rejoin之前的次数。
* masterElectionIgnoreNonMasters：用来控制在主节点选举时候的ping响应，只有在极端情况下才会使用这个参数，平时一般不用配置，默认值为false

~~~yaml
有人说，选举master时，node.master为false的节点的投票是不起作用的，这个说法不完全正确：如果discovery.zen.master_election.ignore_non_master_pings设置为true，那么以上说法正确，但是默认是false，也就是说，它们的投票是起作用的，只是它们不可能成为master。所以我觉得，集群机器数不大的话，除了负担特别重的机器，都设置为node.master为true比较妥当。

设置需要加入新一轮master选举的“master”候选人的最小数量
也就是说，集群中，该值是针对那些node.master=true的来设置的，建议>=num(node.master=true)/2+1.并不是有的朋友解释的，集群机器数量的除以2再加1，当然默认情况下是，因为默认情况下，discovery.zen.master_election.ignore_non_master_pings为false

~~~

* masterElectionWaitForJoinsTimeout：master选举时等待join的timeout,默认是joinTimeout的一半。

其中joinRetryAttempts和maxPingsFromAnotherMaster是一定要大于等于1的。



UnicastZenPing.java 是一个ZenPing 实现类，主要是负责底层和其他Nodes建立并维护连接的任务

PublishClusterStateAction.java 在`ZenDiscovery`中的变量名是`publishClusterState`，之前讲过，这些`**Action` 都是对`**Service`的封装，因此它主要是用来处理发送事件和处理事件的接口，比如发送一个`clusterStateChangeEvent` 和处理这个event，都是通过这个类调用

MasterFaultDetection.java 构建完cluster后所有的node用来检测master存活状态的类

NodeFaultDetection.java 构建完cluster后master用来检测其他node存活状态的类



##### 2.3 如何运行

我们通过上一篇的分析知道，在ES启动的时候会去实例化Node，然后调用Node#start()方法启动各个module，Discovery是在实例化Node的时候通过guice进行注入的，在Node启动的时候去启动的，代码如下：

Node的构造函数中实例化

~~~Java
...
final DiscoveryModule discoveryModule = new DiscoveryModule(this.settings, threadPool, transportService,
                namedWriteableRegistry, networkService, clusterService, pluginsService.filterPlugins(DiscoveryPlugin.class));
...
b.bind(Discovery.class).toInstance(discoveryModule.getDiscovery());
~~~

###### 2.3.1 ZenDiscover的初始化

初始化的时候会加载我上段ZenDiscovery模块介绍提到的几个模块，我就不再重复了，值得注意的是Fault Delection的分为两个masterFD和nodesFD；其次还加载了一些对于discover的配置

###### 2.3.2 ZenDiscovery运行

其实ZenDiscover的运行就是几个子模块的运行；它是通过Node#start()方法启动的。

在Node#start()方法中：我们可以看到Discovery相关的代码

~~~Java
Discovery discovery = injector.getInstance(Discovery.class);
clusterService.setDiscoverySettings(discovery.getDiscoverySettings());
clusterService.addInitialStateBlock(discovery.getDiscoverySettings().getNoMasterBlock());
clusterService.setClusterStatePublisher(discovery::publish);
...
discovery.startInitialJoin();
~~~

在DiscoveryModule类中，

~~~java 
 Map<String, Supplier<Discovery>> discoveryTypes = new HashMap<>();
        discoveryTypes.put("zen",
            () -> new ZenDiscovery(settings, threadPool, transportService, namedWriteableRegistry, clusterService, hostsProvider));
        discoveryTypes.put("none", () -> new NoneDiscovery(settings, clusterService, clusterService.getClusterSettings()));
        discoveryTypes.put("single-node", () -> new SingleNodeDiscovery(settings, clusterService));
        for (DiscoveryPlugin plugin : plugins) {
            plugin.getDiscoveryTypes(threadPool, transportService, namedWriteableRegistry,
                clusterService, hostsProvider).entrySet().forEach(entry -> {
                if (discoveryTypes.put(entry.getKey(), entry.getValue()) != null) {
                    throw new IllegalArgumentException("Cannot register discovery type [" + entry.getKey() + "] twice");
                }
            });
        }

String discoveryType = DISCOVERY_TYPE_SETTING.get(settings);
        // 这里是函数式编程的用法，详情请百度或者Google
        Supplier<Discovery> discoverySupplier = discoveryTypes.get(discoveryType);
        if (discoverySupplier == null) {
            throw new IllegalArgumentException("Unknown discovery type [" + discoveryType + "]");
        }
        Loggers.getLogger(getClass(), settings).info("using discovery type [{}]", discoveryType);
        discovery = Objects.requireNonNull(discoverySupplier.get());
~~~

由上面的代码可以看出，这里Discovery的实例是由DisdcoveryModule的suppiler 提供。



