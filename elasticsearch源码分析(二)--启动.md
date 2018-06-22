---
title: elasticsearch源码分析(二)--启动
date: 2018-05-26 12:12:57
categories: 
- Elasticsearch源码分析专题
---

由于最近半年来一直在使用Elasticsearch来做全文检索和ELK统一日志工作，对于ES还是觉得需要细细研究，才能感受到它的魅力，才能有所提高。

我们先提出几个问题：

* 启动入口在哪个类？
* 启动需要做哪些初始化工作？
* 如何加载配置文件？


#### 一、怎么找启动入口在哪个类

看源码最头疼的事情就是找入口，相信很多刚开始也是这样，面对那么多模块中的类，很难找到一个切入点，我刚开始看也是这样，对于这样的问题，其实还是自己的积累不够，多学习就是了。

我们先来看看启动的脚本elasticsearch.bat或者elasticsearch.sh

~~~bat
@echo off

忽略其他

%JAVA% %ES_JAVA_OPTS% %ES_PARAMS% -cp "%ES_CLASSPATH%" "org.elasticsearch.bootstrap.Elasticsearch" !newparams!

ENDLOCAL

~~~

看到了org.elasticsearch.bootstrap.Elasticsearch这个类，不用想就是它的启动类。

#### 二、Elasticsearch类做了什么事情

我们先来猜想一下，我们下载完Elasticsearch的安装包，一般有两种部署方式：单机部署和集群部署

##### 2.1 单机部署

一般我们会修改{Elasticsearch_home}\config下的elasticsearch.yml文件和jvm.options

在elasticsearch.yml中配置集群名称、节点名称、日志存放路径、数据存放路径、网络IP、http端口（9200）、Netty端口（9300）等

同时还会去初始化一些module，如下图

![:\hexo\source\images\es\es单机启动.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/es%E5%8D%95%E6%9C%BA%E5%90%AF%E5%8A%A8.png)



##### 2.2 集群部署

我们会在单机部署的基础上，增加Discovery模块（集群发现）的配置、

有哪些节点参与到集群当中：discovery.zen.ping.unicast.hosts: ["host1", "host2"]

需要有几个皇子在场才可以选举投票出master：discovery.zen.minimum_master_nodes: 3



##### 2.3 启动流程猜想

通过上述分析我们知道，ES集群启动会做一些初始化工作、加载配置文件，加载一下扩展插件，如果是集群启动，还会进行master选举，master选举需要有足够多的节点参与投票，这个参数是可以指定。



#### 三、启动源码分析

##### 3.1 Elasticsearch类图

![:\hexo\source\images\es\Elasticsearch类图.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/Elasticsearch%E7%B1%BB%E5%9B%BE.png)



##### 3.2 Elasticsearch#main()方法

我们先来看看org.elasticsearch.bootstrap.Elasticsearch#main()方法

~~~Java
public static void main(final String[] args) throws Exception {
        // we want the JVM to think there is a security manager installed so that if internal policy decisions that would be based on the
        // presence of a security manager or lack thereof act as if there is a security manager present (e.g., DNS cache policy)
        System.setSecurityManager(new SecurityManager() {
            @Override
            public void checkPermission(Permission perm) {
                // grant all permissions so that we can later set the security manager to the one that we want
            }
        });
        LogConfigurator.registerErrorListener();
        // 调用构造器
        final Elasticsearch elasticsearch = new Elasticsearch();
        // 调用main方法，执行完后返回一个状态
        int status = main(args, elasticsearch, Terminal.DEFAULT);
        // 判断状态是否启动成功
        if (status != ExitCodes.OK) {
            exit(status);
        }
    }
 static int main(final String[] args, final Elasticsearch elasticsearch, final Terminal terminal) throws Exception {
        return elasticsearch.main(args, terminal);
    }
~~~

通过上面的类图关系，我们知道Elasticsearch是一个Command，就是一开始先设置了一个SecurityManager，做一些检查checkPermission(Permission perm)，因此主要还是增加一些启停的hook，配置日志输出，用意看注释吧，接着打印了一些基本参数后则进入`init`方法，在Command#execute(terminal, options)方法里会调用`Bootstrap.init(!daemonize, pidFile, quiet, initialEnv);`

~~~Java
public final int main(String[] args, Terminal terminal) throws Exception {
        if (addShutdownHook()) {
            shutdownHookThread.set(new Thread(() -> {
                try {
                    this.close();
                } catch (final IOException e) {
                    try (
                        StringWriter sw = new StringWriter();
                        PrintWriter pw = new PrintWriter(sw)) {
                        e.printStackTrace(pw);
                        terminal.println(sw.toString());
                    } catch (final IOException impossible) {
                        // StringWriter#close declares a checked IOException from the Closeable interface but the Javadocs for StringWriter
                        // say that an exception here is impossible
                        throw new AssertionError(impossible);
                    }
                }
            }));
            // 当JVM关闭时，会执行系统中已经设置的所有通过方法addShutdownHook添加的钩子，
            // 当系统执行完这些钩子后，jvm才会关闭
            Runtime.getRuntime().addShutdownHook(shutdownHookThread.get());
        }
~~~

配置日志输出Command#main()方法中

~~~
// 配置日志输出
// initialize default for es.logger.level because we will not read the log4j2.properties
final String loggerLevel = System.getProperty("es.logger.level", Level.INFO.name());
final Settings settings = Settings.builder().put("logger.level", loggerLevel).build();
LogConfigurator.configureWithoutConfig(settings);
~~~

LogConfigurator#configureWithoutConfig()方法

~~~java
   public static void configureWithoutConfig(final Settings settings) {
        Objects.requireNonNull(settings);
        // we initialize the status logger immediately otherwise Log4j will complain when we try to get the context
        configureStatusLogger();
        configureLoggerLevels(settings);
    }
~~~

在Command#mainWithoutErrorHandling(args, terminal)中执行Command，同时会抛出所有的异常给Command#main()方法，真正调用execute(terminal, options)方法执行操作，这是一个抽象方法，通过我们的类图,它的实现类应该是EnvironmentAwareCommand#execute()

![:\hexo\source\images\es\Command-execute实现类.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/Command-execute%E5%AE%9E%E7%8E%B0%E7%B1%BB.png)

~~~Java
@Override
    protected void execute(Terminal terminal, OptionSet options) throws Exception {
        // 将配置信息设置到HashMap中
        final Map<String, String> settings = new HashMap<>();
        for (final KeyValuePair kvp : settingOption.values(options)) {
            if (kvp.value.isEmpty()) {
                throw new UserException(ExitCodes.USAGE, "setting [" + kvp.key + "] must not be empty");
            }
            if (settings.containsKey(kvp.key)) {
                final String message = String.format(
                        Locale.ROOT,
                        "setting [%s] already set, saw [%s] and [%s]",
                        kvp.key,
                        settings.get(kvp.key),
                        kvp.value);
                throw new UserException(ExitCodes.USAGE, message);
            }
            settings.put(kvp.key, kvp.value);
        }
        // 检查了elasticsearch的三个环境参数：
        putSystemPropertyIfSettingIsMissing(settings, "path.conf", "es.path.conf");
        putSystemPropertyIfSettingIsMissing(settings, "path.data", "es.path.data");
        putSystemPropertyIfSettingIsMissing(settings, "path.home", "es.path.home");
        putSystemPropertyIfSettingIsMissing(settings, "path.logs", "es.path.logs");
        // 调用execute方法
        execute(terminal, options, createEnv(terminal, settings));
    }
~~~

该方法也是一个抽象方法，它有很多实现类

![:\hexo\source\images\es\EnvironmentAwareCommand-execute方法实现类.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/EnvironmentAwareCommand-execute%E6%96%B9%E6%B3%95%E5%AE%9E%E7%8E%B0%E7%B1%BB.png)

在该方法中，会先调用createEnv(terminal, settings)设置环境参数，使用该方法来加载配置文件信息

~~~Java
  /** Create an {@link Environment} for the command to use. Overrideable for tests. */
    protected Environment createEnv(Terminal terminal, Map<String, String> settings) {
        return InternalSettingsPreparer.prepareEnvironment(Settings.EMPTY, terminal, settings);
    }
~~~

那么这些配置信息怎么跟节点信息关联呢？

##### 3.3 Elasticsearch#execute()方法

直接来看Elasticsearch#execute()方法做了什么？

~~~Java
protected void execute(Terminal terminal, OptionSet options, Environment env) throws UserException {
        // 检查参数是否为空
        if (options.nonOptionArguments().isEmpty() == false) {
            throw new UserException(ExitCodes.USAGE, "Positional arguments not allowed, found " + options.nonOptionArguments());
        }
        if (options.has(versionOption)) {
            if (options.has(daemonizeOption) || options.has(pidfileOption)) {
                throw new UserException(ExitCodes.USAGE, "Elasticsearch version option is mutually exclusive with any other option");
            }
            terminal.println("Version: " + org.elasticsearch.Version.CURRENT
                    + ", Build: " + Build.CURRENT.shortHash() + "/" + Build.CURRENT.date()
                    + ", JVM: " + JvmInfo.jvmInfo().version());
            return;
        }
        // 是否以守护线程启动（后台启动 -d）
        final boolean daemonize = options.has(daemonizeOption);
        // 进程文件
        final Path pidFile = pidfileOption.value(options);
        // 
        final boolean quiet = options.has(quietOption);

        try {
            // 执行初始化方法
            init(daemonize, pidFile, quiet, env);
        } catch (NodeValidationException e) {
            throw new UserException(ExitCodes.CONFIG, e.getMessage());
        }
    }
~~~

该方法主要是检查一些参数，然后调用Elasticsearch#init(daemonize, pidFile, quiet, env)方法，在方法里会调用`Bootstrap.init(!daemonize, pidFile, quiet, initialEnv)`，而这个方法才是Elasticsearch真正去启动ES。

~~~java 
/**
     * This method is invoked by {@link Elasticsearch#main(String[])} to startup elasticsearch.
     */
    static void init(
            final boolean foreground,
            final Path pidFile,
            final boolean quiet,
            final Environment initialEnv) throws BootstrapException, NodeValidationException, UserException {
        // Set the system property before anything has a chance to trigger its use
        initLoggerPrefix();

        // force the class initializer for BootstrapInfo to run before
        // the security manager is installed
        BootstrapInfo.init();

        INSTANCE = new Bootstrap();
        final SecureSettings keystore = loadSecureSettings(initialEnv);
        Environment environment = createEnvironment(foreground, pidFile, keystore, initialEnv.settings());
        try {
            // 配置日志输出
            LogConfigurator.configure(environment);
        } catch (IOException e) {
            throw new BootstrapException(e);
        }
        // 检查自定义配置文件
        checkForCustomConfFile();
        // 检查是否配置错误
        checkConfigExtension(environment.configExtension());
        // 如果pidFile文件不为空，则创建pid文件，会在磁盘上持久化一个记录应用pid的文件
        if (environment.pidFile() != null) {
            try {
                PidFile.create(environment.pidFile(), true);
            } catch (IOException e) {
                throw new BootstrapException(e);
            }
        }
        //通过参数foreground和quiet来控制日志输出
        final boolean closeStandardStreams = (foreground == false) || quiet;
        try {
            if (closeStandardStreams) {
                final Logger rootLogger = ESLoggerFactory.getRootLogger();
                final Appender maybeConsoleAppender = Loggers.findAppender(rootLogger, ConsoleAppender.class);
                if (maybeConsoleAppender != null) {
                    Loggers.removeAppender(rootLogger, maybeConsoleAppender);
                }
                closeSystOut();
            }

            // fail if somebody replaced the lucene jars
            checkLucene();

            // install the default uncaught exception handler; must be done before security is
            // initialized as we do not want to grant the runtime permission
            // setDefaultUncaughtExceptionHandler
            // 初始化节点信息
            Thread.setDefaultUncaughtExceptionHandler(
                new ElasticsearchUncaughtExceptionHandler(() -> Node.NODE_NAME_SETTING.get(environment.settings())));
            // 调用Bootstrap的setup方法和start方法
            INSTANCE.setup(true, environment);

            try {
                // any secure settings must be read during node construction
                IOUtils.close(keystore);
            } catch (IOException e) {
                throw new BootstrapException(e);
            }
            // 调用Bootstrap的start方法
            INSTANCE.start();

            if (closeStandardStreams) {
                closeSysError();
            }
            ... 略
~~~

参数详解

- foreground：标识elasticsearch是否是作为后台守护进程启动的，
- pidFile：通过parser解析args后得到，实际是解析了默认命令行参数（verbose，E,silent，version，help，quiet，daemonize，pidfile）
- quiet：同上
- initialEnv：Environment实例化的环境参数对象，保存了一些类似于repoFile，configFile，pluginsFile，binFile，libFile等参数。

通过上述的源码阅读，我们发现在该方法中：

主要工作

- 首先会实例化一个Bootstrap对象
- 配置log输出器
- 创建pid文件，会在磁盘上持久化一个记录应用pid的文件
- 通过参数foreground和quiet来控制日志输出
- 调用Bootstrap的setup方法和start方法

##### 3.5 Bootstrap#setup()方法

~~~Java
setup(boolean addShutdownHook, Environment environment)throws BootstrapException 
~~~

该方法主要工作

* 通过environment生成本地插件控制器

~~~Java
Settings settings = environment.settings();

        try {
            // Spawner类是一个Environment本地插件控制器
            spawner.spawnNativePluginControllers(environment);
        } catch (IOException e) {
            throw new BootstrapException(e);
        }
~~~

* 初始化本地资源

  ~~~Java
  initializeNatives(
                  environment.tmpFile(),
                  BootstrapSettings.MEMORY_LOCK_SETTING.get(settings),
                  BootstrapSettings.SYSTEM_CALL_FILTER_SETTING.get(settings),
                  BootstrapSettings.CTRLHANDLER_SETTING.get(settings));
  ~~~

  ​

* 在安全管理器安装之前初始化探针

  ~~~Java
  initializeProbes();
  ~~~

  ​

* 添加关闭钩子

  ~~~Java
  if (addShutdownHook) {
              Runtime.getRuntime().addShutdownHook(new Thread() {
                  @Override
                  public void run() {
                      try {
                          IOUtils.close(node, spawner);
                          LoggerContext context = (LoggerContext) LogManager.getContext(false);
                          Configurator.shutdown(context);
                      } catch (IOException ex) {
                          throw new ElasticsearchException("failed to stop node", ex);
                      }
                  }
              });
          }
  ~~~

  ​

* 检查jar重复

  ~~~Java
  try {
              // look for jar hell,检查jar重复
              JarHell.checkJarHell();
          } catch (IOException | URISyntaxException e) {
              throw new BootstrapException(e);
          }
  ~~~

  ​

* 在安全管理器安装之前配置日志输出器

  ~~~Java
  // install SM after natives, shutdown hooks, etc.
          // 安装安全管理器
          try {
              Security.configure(environment, BootstrapSettings.SECURITY_FILTER_BAD_DEFAULTS_SETTING.get(settings));
          } catch (IOException | NoSuchAlgorithmException e) {
              throw new BootstrapException(e);
          }
  ~~~

  ​

* 安装安全管理器

  ~~~Java
  // install SM after natives, shutdown hooks, etc.
          // 安装安全管理器
          try {
              Security.configure(environment,               BootstrapSettings.SECURITY_FILTER_BAD_DEFAULTS_SETTING.get(settings));
          } catch (IOException | NoSuchAlgorithmException e) {
              throw new BootstrapException(e);
          }
  ~~~

  ​

* 通过参数environment实例化Node

  ~~~Java
  // 通过参数environment实例化Node
          node = new Node(environment) {
              @Override
              protected void validateNodeBeforeAcceptingRequests(
                  final Settings settings,
                  final BoundTransportAddress boundTransportAddress, List<BootstrapCheck> checks) throws NodeValidationException {
                  BootstrapChecks.check(settings, boundTransportAddress, checks);
              }
          };
  ~~~

  ​

##### 3.6 Bootstrap#start()方法

~~~Java
 private void start() throws NodeValidationException {
        node.start();
        keepAliveThread.start();
    }
~~~

主要工作

- 启动已经实例化的Node

- 启动keepAliveThread 线程，这个线程在Bootstrap初始化的时候就已经实例化了，该线程创建了一个计数为1的CountDownLatch，目的是在启动完成后能顺利添加关闭钩子，而这句：

  ~~~java 
  Runtime.getRuntime().addShutdownHook(new Thread())
  ~~~

  意思就是在jvm中增加一个关闭的钩子，当jvm关闭的时候，会执行系统中已经设置的所有通过方法addShutdownHook添加的钩子，当系统执行完这些钩子后，jvm才会关闭。所以这些钩子可以在jvm关闭的时候进行内存清理、对象销毁等操作。
  可以看到启动的重点在setup方法中，启动过后就是Node的事了。

  keepAliveThhread线程

  ~~~java 
  private final CountDownLatch keepAliveLatch = new CountDownLatch(1);
  /** creates a new instance */
      Bootstrap() {
          // 在构造器中就创建keepAliveThread线程
          keepAliveThread = new Thread(new Runnable() {
              @Override
              public void run() {
                  try {
                      keepAliveLatch.await();
                  } catch (InterruptedException e) {
                      // bail out
                  }
              }
          }, "elasticsearch[keepAlive/" + Version.CURRENT + "]");
          keepAliveThread.setDaemon(false);
          // keep this thread alive (non daemon thread) until we shutdown
          Runtime.getRuntime().addShutdownHook(new Thread() {
              @Override
              public void run() {
                  // 这里的钩子执行完毕，才会执行完keepAliveThread线程的run()方法
                  keepAliveLatch.countDown();
              }
          });
      }
  ~~~



##### 3.4 Node类源码解读

我们先不看源码，如果是你，会怎么去设计这个Node类？会怎么去加载配置文件信息？

猜想，我们启动ES都是一个节点Node，如果是集群，会有多个Node，那么我们应该也是通过Node来加载配置文件，加载完配置文件构造一个Config对象，最后初始化一个Node对象。

继续猜想，Node应该是包含一些基本信息、全局环境配置Setting和Environment，节点环境NodeEnvironment、是否为master、是否可以参与投票等。

问题：这些信息设置完毕，如何启动、如何停止？如何加载插件？



验证猜想，查看类的定义信息

![:\hexo\source\images\es\Node定义1.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/Node%E5%AE%9A%E4%B9%891.png)

![:\hexo\source\images\es\Node定义.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/Node%E5%AE%9A%E4%B9%89.png)

###### 3.4.1 Node初始化

我们前面通过分析Bootstrap#setup()方法知道，Node的实例化是在该方法中调用 new Node(environment)进行的，节点的启动是在Bootstrap#start()方法中调用Node#start()方法进行启动的。

~~~Java
// 通过参数environment实例化Node
        node = new Node(environment) {
            @Override
            protected void validateNodeBeforeAcceptingRequests(
                final Settings settings,
                final BoundTransportAddress boundTransportAddress, List<BootstrapCheck> checks) throws NodeValidationException {
                BootstrapChecks.check(settings, boundTransportAddress, checks);
            }
        };
~~~

使用google的注入框架Guice的Injector进行注入与获取实例。elasticsearch里面的组件都是用上面的方法进行模块化管理，elasticsearch对guice进行了封装，通过ModulesBuilder类构建elasticsearch的模块：

~~~Java
ModulesBuilder modules = new ModulesBuilder();
            // plugin modules must be added here, before others or we can get crazy injection errors...
            for (Module pluginModule : pluginsService.createGuiceModules()) {
                modules.add(pluginModule);
            }
            final MonitorService monitorService = new MonitorService(settings, nodeEnvironment, threadPool);
            modules.add(new NodeModule(this, monitorService));
            ClusterModule clusterModule = new ClusterModule(settings, clusterService,
                pluginsService.filterPlugins(ClusterPlugin.class));
            modules.add(clusterModule);
            IndicesModule indicesModule = new IndicesModule(pluginsService.filterPlugins(MapperPlugin.class));
            modules.add(indicesModule);

            SearchModule searchModule = new SearchModule(settings, false, pluginsService.filterPlugins(SearchPlugin.class));
            CircuitBreakerService circuitBreakerService = createCircuitBreakerService(settingsModule.getSettings(),
                settingsModule.getClusterSettings());
            resourcesToClose.add(circuitBreakerService);
...
~~~

Node的实例化主要工作：

* 设置初始化信息：nodeEnvironment

~~~java 
try {
            Settings tmpSettings = Settings.builder().put(environment.settings())
                .put(Client.CLIENT_TYPE_SETTING_S.getKey(), CLIENT_TYPE).build();

            tmpSettings = TribeService.processSettings(tmpSettings);

            // create the node environment as soon as possible, to recover the node id and enable logging
            try {
                nodeEnvironment = new NodeEnvironment(tmpSettings, environment);
                resourcesToClose.add(nodeEnvironment);
            } catch (IOException ex) {
                throw new IllegalStateException("Failed to create node environment", ex);
            }
            final boolean hadPredefinedNodeName = NODE_NAME_SETTING.exists(tmpSettings);
            Logger logger = Loggers.getLogger(Node.class, tmpSettings);
            final String nodeId = nodeEnvironment.nodeId();
            tmpSettings = addNodeNameIfNeeded(tmpSettings, nodeId);
            if (DiscoveryNode.nodeRequiresLocalStorage(tmpSettings)) {
                checkForIndexDataInDefaultPathData(tmpSettings, nodeEnvironment, logger);
            }
            // this must be captured after the node name is possibly added to the settings
            final String nodeName = NODE_NAME_SETTING.get(tmpSettings);
            if (hadPredefinedNodeName == false) {
                logger.info("node name [{}] derived from node ID [{}]; set [{}] to override", nodeName, nodeId, NODE_NAME_SETTING.getKey());
            } else {
                logger.info("node name [{}], node ID [{}]", nodeName, nodeId);
            }
~~~



* 打印JVM信息

  ​

* 初始化pluginsService类

~~~Java
this.pluginsService = new PluginsService(tmpSettings, environment.modulesFile(), environment.pluginsFile(), classpathPlugins);
~~~

* environment(这里会加载配置文件)

  ~~~Java
  this.environment = new Environment(this.settings);
  Environment.assertEquivalent(environment, this.environment);
  ~~~

* Executors 和threadPool

~~~Java
final List<ExecutorBuilder<?>> executorBuilders = pluginsService.getExecutorBuilders(settings);

final ThreadPool threadPool = new ThreadPool(settings, executorBuilders.toArray(new ExecutorBuilder[0]));
resourcesToClose.add(() -> ThreadPool.terminate(threadPool, 10, TimeUnit.SECONDS));
// adds the context to the DeprecationLogger so that it does not need to be injected everywhere
DeprecationLogger.setThreadContext(threadPool.getThreadContext());
resourcesToClose.add(() -> DeprecationLogger.removeThreadContext(threadPool.getThreadContext()));
~~~

我们来看es线程池做了什么？

~~~Java
public ThreadPool(final Settings settings, final ExecutorBuilder<?>... customBuilders) {
        super(settings);

        assert Node.NODE_NAME_SETTING.exists(settings);
        // 将构造好的线程池添加到HashMap中，key是线程池的名称，value是ExecutorBuilder
        // 每一个线程都是通过ExecutorBuilder来构造
        final Map<String, ExecutorBuilder> builders = new HashMap<>();
        final int availableProcessors = EsExecutors.boundedNumberOfProcessors(settings);
        final int halfProcMaxAt5 = halfNumberOfProcessorsMaxFive(availableProcessors);
        final int halfProcMaxAt10 = halfNumberOfProcessorsMaxTen(availableProcessors);
        final int genericThreadPoolMax = boundedBy(4 * availableProcessors, 128, 512);
        builders.put(Names.GENERIC, new ScalingExecutorBuilder(Names.GENERIC, 4, genericThreadPoolMax, TimeValue.timeValueSeconds(30)));
        builders.put(Names.INDEX, new FixedExecutorBuilder(settings, Names.INDEX, availableProcessors, 200));
        builders.put(Names.BULK, new FixedExecutorBuilder(settings, Names.BULK, availableProcessors, 200)); // now that we reuse bulk for index/delete ops
        builders.put(Names.GET, new FixedExecutorBuilder(settings, Names.GET, availableProcessors, 1000));
        builders.put(Names.SEARCH, new FixedExecutorBuilder(settings, Names.SEARCH, searchThreadPoolSize(availableProcessors), 1000));
        builders.put(Names.MANAGEMENT, new ScalingExecutorBuilder(Names.MANAGEMENT, 1, 5, TimeValue.timeValueMinutes(5)));
        // no queue as this means clients will need to handle rejections on listener queue even if the operation succeeded
        // the assumption here is that the listeners should be very lightweight on the listeners side
        builders.put(Names.LISTENER, new FixedExecutorBuilder(settings, Names.LISTENER, halfProcMaxAt10, -1));
        builders.put(Names.FLUSH, new ScalingExecutorBuilder(Names.FLUSH, 1, halfProcMaxAt5, TimeValue.timeValueMinutes(5)));
        builders.put(Names.REFRESH, new ScalingExecutorBuilder(Names.REFRESH, 1, halfProcMaxAt10, TimeValue.timeValueMinutes(5)));
        builders.put(Names.WARMER, new ScalingExecutorBuilder(Names.WARMER, 1, halfProcMaxAt5, TimeValue.timeValueMinutes(5)));
        builders.put(Names.SNAPSHOT, new ScalingExecutorBuilder(Names.SNAPSHOT, 1, halfProcMaxAt5, TimeValue.timeValueMinutes(5)));
        builders.put(Names.FETCH_SHARD_STARTED, new ScalingExecutorBuilder(Names.FETCH_SHARD_STARTED, 1, 2 * availableProcessors, TimeValue.timeValueMinutes(5)));
        builders.put(Names.FORCE_MERGE, new FixedExecutorBuilder(settings, Names.FORCE_MERGE, 1, -1));
        builders.put(Names.FETCH_SHARD_STORE, new ScalingExecutorBuilder(Names.FETCH_SHARD_STORE, 1, 2 * availableProcessors, TimeValue.timeValueMinutes(5)));
        for (final ExecutorBuilder<?> builder : customBuilders) {
            if (builders.containsKey(builder.name())) {
                throw new IllegalArgumentException("builder with name [" + builder.name() + "] already exists");
            }
            builders.put(builder.name(), builder);
        }
        this.builders = Collections.unmodifiableMap(builders);

        threadContext = new ThreadContext(settings);

        final Map<String, ExecutorHolder> executors = new HashMap<>();
        for (@SuppressWarnings("unchecked") final Map.Entry<String, ExecutorBuilder> entry : builders.entrySet()) {
            final ExecutorBuilder.ExecutorSettings executorSettings = entry.getValue().getSettings(settings);
            final ExecutorHolder executorHolder = entry.getValue().build(executorSettings, threadContext);
            if (executors.containsKey(executorHolder.info.getName())) {
                throw new IllegalStateException("duplicate executors with name [" + executorHolder.info.getName() + "] registered");
            }
            logger.debug("created thread pool: {}", entry.getValue().formatInfo(executorHolder.info));
            executors.put(entry.getKey(), executorHolder);
        }

        executors.put(Names.SAME, new ExecutorHolder(DIRECT_EXECUTOR, new Info(Names.SAME, ThreadPoolType.DIRECT)));
        this.executors = unmodifiableMap(executors);
        // 最后创建一个1线程的scheduler来执行定时任务
        this.scheduler = new ScheduledThreadPoolExecutor(1, EsExecutors.daemonThreadFactory(settings, "scheduler"), new EsAbortPolicy());
        this.scheduler.setExecuteExistingDelayedTasksAfterShutdownPolicy(false);
        this.scheduler.setContinueExistingPeriodicTasksAfterShutdownPolicy(false);
        this.scheduler.setRemoveOnCancelPolicy(true);

        TimeValue estimatedTimeInterval = ESTIMATED_TIME_INTERVAL_SETTING.get(settings);
        // 最后创建一个执行timer的线程
        this.cachedTimeThread = new CachedTimeThread(EsExecutors.threadName(settings, "[timer]"), estimatedTimeInterval.millis());
        this.cachedTimeThread.start();
    }
~~~

原来在ES的threadPool中，根据不同的类型分别分配了不同线程数的一个线程池，而executor由一个executorBuilder来提供，所以submit task的时候也需要指定不同的Name。最后创建一个1线程的scheduler来执行定时任务。最后创建一个执行timer的线程。

再继续往下看Node的构造方法就会看到接下来会new 一堆的services和modules，这里就不一一过了，其共性就是都会绑定刚刚创建的threadPool，已经也会绑定必要的services，某些module本身具有后台线程的话，初始化完成需要调用`.start()`去启动这些后台线程。



* 初始化modules实例，通过Guice的Injector进行注入各个Module实例

~~~java 
ModulesBuilder modules = new ModulesBuilder();
***
modules.add(b -> {
                    b.bind(NodeService.class).toInstance(nodeService);
                    b.bind(NamedXContentRegistry.class).toInstance(xContentRegistry);
                    b.bind(PluginsService.class).toInstance(pluginsService);
                    b.bind(Client.class).toInstance(client);
                    b.bind(NodeClient.class).toInstance(client);
                    b.bind(Environment.class).toInstance(this.environment);
                    b.bind(ThreadPool.class).toInstance(threadPool);
                    b.bind(NodeEnvironment.class).toInstance(nodeEnvironment);
                    b.bind(TribeService.class).toInstance(tribeService);
                    b.bind(ResourceWatcherService.class).toInstance(resourceWatcherService);
                    b.bind(CircuitBreakerService.class).toInstance(circuitBreakerService);
                    b.bind(BigArrays.class).toInstance(bigArrays);
                    b.bind(ScriptService.class).toInstance(scriptModule.getScriptService());
                    b.bind(AnalysisRegistry.class).toInstance(analysisModule.getAnalysisRegistry());
                    b.bind(IngestService.class).toInstance(ingestService);
                    b.bind(NamedWriteableRegistry.class).toInstance(namedWriteableRegistry);
                    b.bind(MetaDataUpgrader.class).toInstance(metaDataUpgrader);
                    b.bind(MetaStateService.class).toInstance(metaStateService);
                    b.bind(IndicesService.class).toInstance(indicesService);
                    b.bind(SearchService.class).toInstance(newSearchService(clusterService, indicesService,
                        threadPool, scriptModule.getScriptService(), bigArrays, searchModule.getFetchPhase()));
                    b.bind(SearchTransportService.class).toInstance(searchTransportService);
                    b.bind(SearchPhaseController.class).toInstance(new SearchPhaseController(settings, bigArrays,
                            scriptModule.getScriptService()));
                    b.bind(Transport.class).toInstance(transport);
                    b.bind(TransportService.class).toInstance(transportService);
                    b.bind(NetworkService.class).toInstance(networkService);
                    b.bind(UpdateHelper.class).toInstance(new UpdateHelper(settings, scriptModule.getScriptService()));
                    b.bind(MetaDataIndexUpgradeService.class).toInstance(new MetaDataIndexUpgradeService(settings, xContentRegistry,
                        indicesModule.getMapperRegistry(), settingsModule.getIndexScopedSettings(), indexMetaDataUpgraders));
                    b.bind(ClusterInfoService.class).toInstance(clusterInfoService);
                    b.bind(Discovery.class).toInstance(discoveryModule.getDiscovery());
                    {
                        RecoverySettings recoverySettings = new RecoverySettings(settings, settingsModule.getClusterSettings());
                        processRecoverySettings(settingsModule.getClusterSettings(), recoverySettings);
                        b.bind(PeerRecoverySourceService.class).toInstance(new PeerRecoverySourceService(settings, transportService,
                                indicesService, recoverySettings, clusterService));
                        b.bind(PeerRecoveryTargetService.class).toInstance(new PeerRecoveryTargetService(settings, threadPool,
                                transportService, recoverySettings, clusterService));
                    }
                    httpBind.accept(b);
                    pluginComponents.stream().forEach(p -> b.bind((Class) p.getClass()).toInstance(p));
                }
            );
            injector = modules.createInjector();
~~~

这里面会注入Discovery，ClusterService，Transport Service，还创建了NodeClient用来接收全部其他节点请求。这些都会在往后重点剖析。

###### 3.4.2 启动Node

通过在Bootstrap#start()方法中调用Node.start()来启动节点

我们知道，在Node的初始化方法中，Model组件会被添加到绑定的线程当中，那么启动这些只需要调用相应组件的.start()方法即可完成组件的加载

~~~Java
public Node start() throws NodeValidationException {
        if (!lifecycle.moveToStarted()) {
            return this;
        }

        Logger logger = Loggers.getLogger(Node.class, NODE_NAME_SETTING.get(settings));
        logger.info("starting ...");
        // hack around dependency injection problem (for now...)
        injector.getInstance(Discovery.class).setAllocationService(injector.getInstance(AllocationService.class));
        pluginLifecycleComponents.forEach(LifecycleComponent::start);

        injector.getInstance(MappingUpdatedAction.class).setClient(client);
        injector.getInstance(IndicesService.class).start();
        injector.getInstance(IndicesClusterStateService.class).start();
        injector.getInstance(IndicesTTLService.class).start();
        injector.getInstance(SnapshotsService.class).start();
        injector.getInstance(SnapshotShardsService.class).start();
        injector.getInstance(RoutingService.class).start();
        injector.getInstance(SearchService.class).start();
        injector.getInstance(MonitorService.class).start();
~~~



3.4.3 Node节点停止

该方法跟node启动差不多，也是调用相关组件的stop方法即可，这里就不再分析了



###### 3.4.4 加载配置文件信息

- 入口

通过Node的构造方法

```java
public Node(Settings preparedSettings) {
   this(InternalSettingsPreparer.prepareEnvironment(preparedSettings, null));
}
```

这就是加载配置文件的入口，它有三个方法

```java
public static Settings prepareSettings(Settings input) {
        Settings.Builder output = Settings.builder();
        initializeSettings(output, input, Collections.emptyMap());
        finalizeSettings(output, null);
        return output.build();
    } 
public static Environment prepareEnvironment(Settings input, Terminal terminal) {
        return prepareEnvironment(input, terminal, Collections.emptyMap());
    }
public static Environment prepareEnvironment(Settings input, Terminal terminal, Map<String, String> properties) {}

```

在InternalSettingsPreparer类的prepareEnvironment(org.elasticsearch.common.settings.Settings, org.elasticsearch.cli.Terminal, java.util.Map<java.lang.String,java.lang.String>, java.nio.file.Path)方法中进行了配置文件的加载。

- 加载配置文件的方法

  ```java
  public static Environment prepareEnvironment(Settings input, Terminal terminal, Map<String, String> properties) {
          // just create enough settings to build the environment, to get the config dir
          Settings.Builder output = Settings.builder();
          // 初始化输入输出流信息
          initializeSettings(output, input, properties);
          // 构造Environment实例
          Environment environment = new Environment(output.build());
          // 这个很关键，保证elasticsearch.yml文件中配置的日志路径path.logs生效
          output = Settings.builder(); // start with a fresh output
          boolean settingsFileFound = false;
          Set<String> foundSuffixes = new HashSet<>();
          for (String allowedSuffix : ALLOWED_SUFFIXES) {
              Path path = environment.configFile().resolve("elasticsearch" + allowedSuffix);
              if (Files.exists(path)) {
                  if (!settingsFileFound) {
                      try {
                          output.loadFromPath(path);
                      } catch (IOException e) {
                          throw new SettingsException("Failed to load settings from " + path.toString(), e);
                      }
                  }
                  settingsFileFound = true;
                  foundSuffixes.add(allowedSuffix);
              }
          }
          if (foundSuffixes.size() > 1) {
              throw new SettingsException("multiple settings files found with suffixes: "
                  + Strings.collectionToDelimitedString(foundSuffixes, ","));
          }

          // re-initialize settings now that the config file has been loaded
          initializeSettings(output, input, properties);
          finalizeSettings(output, terminal);
          // 再次获取Environment实例
          environment = new Environment(output.build());

          // we put back the path.logs so we can use it in the logging configuration file
          output.put(Environment.PATH_LOGS_SETTING.getKey(), cleanPath(environment.logsFile().toAbsolutePath().toString()));
          String configExtension = foundSuffixes.isEmpty() ? null : foundSuffixes.iterator().next();
          // 返回Environment实例
          return new Environment(output.build(), configExtension);
  ```

  构建一个默认的Settings的实例

  然后用构造出来的新的Settings来加载给定或默认路径下的*elasticsearch.yml*

  然后将方法接受的参数Settings实例也加载到这个新的Settings中。

  最后才将日志文件的路径加载进Settings中，这样就保证了*elasticsearch.yml*文件中配置的日志路径*path.logs*生效（覆盖该方法参数中的配置）。

  最后返回一个Environment的实例，使得Node开始构建

#### 四、总结：

通过上述的源码分析，我们知道Elasticsearch节点启动的入口是Elasticsearch#main()方法，在该方法中会进行一些安全管理的设置，去调用Command的main()方法，整个方法执行没有任何异常，则返回ok状态。

Command#main()：会去添加一些钩子、配置日志输出、调用mainWithoutErrorHandling()去执行EnvironmentAwareCommand#execute(terminal, options)方法。

EnvironmentAwareCommand#execute(terminal, options)方法：只是将配置信息设置到HashMap中，检查了elasticsearch的参数path.conf、path.data、path.home、path.logs，最后调用Elasticsearch#execute()方法，execute(terminal, options, createEnv(terminal, settings))会先调用EnvironmentAwareCommand# createEnv(terminal, settings)

Elasticsearch#execute()方法：主要是处理参数，调用init(daemonize, pidFile, quiet, env)，真正执行启动的是Bootstrap.init(!daemonize, pidFile, quiet, initialEnv)方法。

Bootstrap.init(!daemonize, pidFile, quiet, initialEnv)：主要是调用setup()方法和start()方法，在setup()方法中主要通过environment生成本地插件控制器spawner、添加钩子、添加安全管理器、检查jar包、创建Node节点。而start()通过启动初始化好的Node和keepAliveThread线程，这个keepAliveThread使用了CountdownLatch计数器为1来保证钩子一定能够关闭。



Node类的初始化：通过设置好的environment来初始化节点，设置nodEnvironment、Environment、设置Node_name、设置线程池（其实是一个HashMap<String,ExecutorBuilder>） ，根据不同的类型分别分配了不同线程数的一个线程池。将创建好的module绑定到创建的ThreadPool。

大致的时序图如下：

![:\hexo\source\images\es\Elasticsearch源码启动时序图.pn](https://github.com/Zhanggdong/Zhanggdong.github.io/raw/master/images/es/Elasticsearch%E6%BA%90%E7%A0%81%E5%90%AF%E5%8A%A8%E6%97%B6%E5%BA%8F%E5%9B%BE.png)



现在还遗留着几个问题：

* master选举是在什么模块进行的


* 怎么进行master选举
* 怎么进行节点监控、维护的

留到下一篇再进行分析。。。

