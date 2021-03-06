Elasticsearch请求处理流程
=======
## Bootstrap
___
Elasticsearch启动模块定义在org.elasticsearch.bootstrap目录中。启动顺序如下：  

main() -> init settings() -> bootstrap.setup() -> bootstrap.start() -> keepalive thread start()  
其中bootstrap.start()调用node.start()启动node service。node service接口在org.elasticsearc.node.internal中的NodeModule.java文件中被bind到InternalNode()类。因此node.start()实际调用的是InternalNode.start()方法。在bootstrap.setup()方法中调用NodeBuilder.build()方法初始化Node相关变量。相关代码如下：   
```java
NodeBuilder nodeBuilder = NodeBuilder.nodeBuilder().settings(tuple.v1()).loadConfigSettings(false);
node = nodeBuilder.build();
if (addShutdownHook) {
    Runtime.getRuntime().addShutdownHook(new Thread() {
    @Override
    public void run() {
                    node.close();
                }
            });
        }
```  
其中nodeBuilder.build()将会初始化InternalNode，InternalNode初始化时将所需的Module全都加载到Guice的ModuleBuilder中。代码如下：    
```java
try {
            ModulesBuilder modules = new ModulesBuilder();
            modules.add(new Version.Module(version));
            modules.add(new PageCacheRecyclerModule(settings));
            modules.add(new CircuitBreakerModule(settings));
            modules.add(new BigArraysModule(settings));
            modules.add(new PluginsModule(settings, pluginsService));
            modules.add(new SettingsModule(settings));
            modules.add(new NodeModule(this));
            modules.add(new NetworkModule());
            modules.add(new ScriptModule(settings));
            modules.add(new EnvironmentModule(environment));
            modules.add(new NodeEnvironmentModule(nodeEnvironment));
            modules.add(new ClusterNameModule(settings));
            modules.add(new ThreadPoolModule(settings));
            modules.add(new DiscoveryModule(settings));
            modules.add(new ClusterModule(settings));
            modules.add(new RestModule(settings));
            modules.add(new TransportModule(settings));
            if (settings.getAsBoolean(HTTP_ENABLED, true)) {
                modules.add(new HttpServerModule(settings));
            }
            modules.add(new RiversModule(settings));
            modules.add(new IndicesModule(settings));
            modules.add(new SearchModule());
            modules.add(new ActionModule(false));
            modules.add(new MonitorModule(settings));
            modules.add(new GatewayModule(settings));
            modules.add(new NodeClientModule());
            modules.add(new ShapeModule());
            modules.add(new PercolatorModule());
            modules.add(new ResourceWatcherModule());
            modules.add(new RepositoriesModule());
            modules.add(new TribeModule());
            modules.add(new BenchmarkModule(settings));

            injector = modules.createInjector();

            client = injector.getInstance(Client.class);
            success = true;
        } finally {
            if (!success) {
                nodeEnvironment.close();
            }
        }
```
在Node初始化完成后，调用bootstrap.start()启动服务，而bootstrap.start()实际上调用的是Node接口中的start()方法。Node接口被绑定到InternalNode实例对象上，从而真正调用的是InternalNode.start()方法。该方法首先轮询所有已安装的plugin实例并启动，然后依次启动以下service：  
```java
 injector.getInstance(MappingUpdatedAction.class).start();
        injector.getInstance(IndicesService.class).start();
        injector.getInstance(IndexingMemoryController.class).start();
        injector.getInstance(IndicesClusterStateService.class).start();
        injector.getInstance(IndicesTTLService.class).start();
        injector.getInstance(RiversManager.class).start();
        injector.getInstance(SnapshotsService.class).start();
        injector.getInstance(TransportService.class).start();
        injector.getInstance(ClusterService.class).start();
        injector.getInstance(RoutingService.class).start();
        injector.getInstance(SearchService.class).start();
        injector.getInstance(MonitorService.class).start();
        injector.getInstance(RestController.class).start();
        DiscoveryService discoService = injector.getInstance(DiscoveryService.class).start();
        discoService.waitForInitialState();

        // gateway should start after disco, so it can try and recovery from gateway on "start"
        injector.getInstance(GatewayService.class).start();

        if (settings.getAsBoolean("http.enabled", true)) {
            injector.getInstance(HttpServer.class).start();
        }
        injector.getInstance(ResourceWatcherService.class).start();
        injector.getInstance(TribeService.class).start();
```
关于HTTP Server是否启动需要根据用户的配置文件是否设置*http.enabled*参数为真来确定。该参数默认为真，如果用户将某个Node设置为Data Node并将该参数设置为False则可以让Data Node使用**Transport**模块提供的服务进行内部通信。详细信息参考该链接 [modules-node](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-node.html#modules-node).  

HTTPServer启动start()方法，start()方法调用doStart()方法，在其中调用transport.start()启动真正的HTTP Server的doStart()方法，即NettyHttpServerTransport.start()。HTTPServer类的doStart()方法定义如下：  
```java
@Override
    protected void doStart() throws ElasticsearchException {
        transport.start();
        if (logger.isInfoEnabled()) {
            logger.info("{}", transport.boundAddress());
        }
        nodeService.putAttribute("http_address", transport.boundAddress().publishAddress().toString());
    }
```

## HTTP Server模块

HTTP模块定义在org.elasticsearch.http目录中。Guice Module定义在HttpServerModule.java文件中，定义如下：
```java  
public class HttpServerModule extends AbstractModule {

    private final Settings settings;
    private final ESLogger logger;

    private Class<? extends HttpServerTransport> configuredHttpServerTransport;
    private String configuredHttpServerTransportSource;

    public HttpServerModule(Settings settings) {
        this.settings = settings;
        this.logger = Loggers.getLogger(getClass(), settings);
    }

    @SuppressWarnings({"unchecked"})
    @Override
    protected void configure() {
        if (configuredHttpServerTransport != null) {
            logger.info("Using [{}] as http transport, overridden by [{}]", configuredHttpServerTransport.getName(), configuredHttpServerTransportSource);
            bind(HttpServerTransport.class).to(configuredHttpServerTransport).asEagerSingleton();
        } else {
            Class<? extends HttpServerTransport> defaultHttpServerTransport = NettyHttpServerTransport.class;
            Class<? extends HttpServerTransport> httpServerTransport = settings.getAsClass("http.type", defaultHttpServerTransport, "org.elasticsearch.http.", "HttpServerTransport");
            bind(HttpServerTransport.class).to(httpServerTransport).asEagerSingleton();
        }

        bind(HttpServer.class).asEagerSingleton();
    }

    public void setHttpServerTransport(Class<? extends HttpServerTransport> httpServerTransport, String source) {
        checkNotNull(httpServerTransport, "Configured http server transport may not be null");
        checkNotNull(source, "Plugin, that changes transport may not be null");
        this.configuredHttpServerTransport = httpServerTransport;
        this.configuredHttpServerTransportSource = source;
    }
```
根据Guice绑定规则，如果用户在配置中设置了Http Server类型则使用用户配置的服务器，否则使用默认的NettyHttpServerTransport。  
在NettyHttpServerTransport.java文件中，doStart()方法用于启动ES的Web服务器。doStart()根据用户配置中是否设置了*blockingserver*启动不同的soket线程。  
```java
@Override
    protected void doStart() throws ElasticsearchException {
        this.serverOpenChannels = new OpenChannelsHandler(logger);

        if (blockingServer) {
            serverBootstrap = new ServerBootstrap(new OioServerSocketChannelFactory(
                    Executors.newCachedThreadPool(daemonThreadFactory(settings, "http_server_boss")),
                    Executors.newCachedThreadPool(daemonThreadFactory(settings, "http_server_worker"))
            ));
        } else {
            serverBootstrap = new ServerBootstrap(new NioServerSocketChannelFactory(
                    Executors.newCachedThreadPool(daemonThreadFactory(settings, "http_server_boss")),
                    Executors.newCachedThreadPool(daemonThreadFactory(settings, "http_server_worker")),
                    workerCount));
        }

        serverBootstrap.setPipelineFactory(configureServerChannelPipelineFactory());
```  
当HTTP Server启动之后开始等待请求，当接收到HTTP请求时，ES调用org.elasticsearch.http.netty.HttpRequestHandler.java中的HttpRequestHandler.messageReceived()方法。  
```java
@Override
    public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
        HttpRequest request;
        OrderedUpstreamMessageEvent oue = null;
        if (this.httpPipeliningEnabled && e instanceof OrderedUpstreamMessageEvent) {
            oue = (OrderedUpstreamMessageEvent) e;
            request = (HttpRequest) oue.getMessage();
        } else {
            request = (HttpRequest) e.getMessage();
        }

        // the netty HTTP handling always copy over the buffer to its own buffer, either in NioWorker internally
        // when reading, or using a cumalation buffer
        NettyHttpRequest httpRequest = new NettyHttpRequest(request, e.getChannel());
        if (oue != null) {
            serverTransport.dispatchRequest(httpRequest, new NettyHttpChannel(serverTransport, httpRequest, corsPattern, oue));
        } else {
            serverTransport.dispatchRequest(httpRequest, new NettyHttpChannel(serverTransport, httpRequest, corsPattern));
        }
        super.messageReceived(ctx, e);
    }
```  
在该方法中调用serverTransport.dispatchRequest()方法，将请求分发到相应的Handler处理。该方法的调用链如下：    
1. NettyHttpServerTransport.dispatchRequest ->  
2. HttpServerAdapter.dispatchRequest() ->   
3. HttpServer.Dispatcher.dispatchRequest() ->   
4. HttpServer.internalDispatchRequest() ->   
5. RestController.dispatchRequest(request, chanel)  
经过上述调用链的处理，一个HTTP请求被发送到RestController等待分发处理。

## Rest模块
____

Rest模块用于接收REST API请求并将其分发到相应的Handler处理。Guice Module定义在org.elasticsearch.rest目录中。根据RestModule.java定义：  
```java
public class RestModule extends AbstractModule {

    private final Settings settings;
    private List<Class<? extends BaseRestHandler>> restPluginsActions = Lists.newArrayList();

    public void addRestAction(Class<? extends BaseRestHandler> restAction) {
        restPluginsActions.add(restAction);
    }

    public RestModule(Settings settings) {
        this.settings = settings;
    }


    @Override
    protected void configure() {
        bind(RestController.class).asEagerSingleton();
        new RestActionModule(restPluginsActions).configure(binder());
    }
}
```  
在*configure*方法中，**RestController**被绑定为单例模式，并调用RestAction模块的bind操作。  

在**RestController**类中定义了*registerHandler*方法注册所有已经支持的RESTful API方法：*GET*, *DELETE*, *POST*, *PUT*, *OPTIONS*, *HEAD*。  同时还定义了*registerFilter*方法注册在请求真正执行之前进行的预处理操作。  

所有的请求在dispatchRequest方法中被分发到具体的Handler处理。dispatchRequest定义如下：  
```java
public void dispatchRequest(final RestRequest request, final RestChannel channel) {
        // If JSONP is disabled and someone sends a callback parameter we should bail out before querying
        if (!settings.getAsBoolean(HTTP_JSON_ENABLE, false) && request.hasParam("callback")){
            try {
                XContentBuilder builder = channel.newBuilder();
                builder.startObject().field("error","JSONP is disabled.").endObject().string();
                RestResponse response = new BytesRestResponse(FORBIDDEN, builder);
                response.addHeader("Content-Type", "application/javascript");
                channel.sendResponse(response);
            } catch (IOException e) {
                logger.warn("Failed to send response", e);
                return;
            }
            return;
        }
        if (filters.length == 0) {
            try {
                executeHandler(request, channel);
            } catch (Throwable e) {
                try {
                    channel.sendResponse(new BytesRestResponse(channel, e));
                } catch (Throwable e1) {
                    logger.error("failed to send failure response for uri [" + request.uri() + "]", e1);
                }
            }
        } else {
            ControllerFilterChain filterChain = new ControllerFilterChain(handlerFilter);
            filterChain.continueProcessing(request, channel);
        }
    }
```  
根据定义，如果用户配置了回调处理函数则直接创建一个响应返回。否则判断请求中是否有过滤器。如果存在滤器则对过滤器链依次调用过滤器处理，否则直接调用*executeHandler(request, channel)*方法。在*executeHandler*方法中，首先获取当前的handler，如果存在已注册的handler则调用handler处理请求，否则判断是否是**OPTIONS**请求，再作相应处理。*executeHandler*方法定义如下：  
```java
void executeHandler(RestRequest request, RestChannel channel) throws Exception {
        final RestHandler handler = getHandler(request);
        if (handler != null) {
            handler.handleRequest(request, channel);
        } else {
            if (request.method() == RestRequest.Method.OPTIONS) {
                // when we have OPTIONS request, simply send OK by default (with the Access Control Origin header which gets automatically added)
                channel.sendResponse(new BytesRestResponse(OK));
            } else {
                channel.sendResponse(new BytesRestResponse(BAD_REQUEST, "No handler found for uri [" + request.uri() + "] and method [" + request.method() + "]"));
            }
        }
    }
```
executeHandler方法首先根据request获取其相应的RestHandler。在getHandler()方法中定义了如下处理器：  
* getHandler ---> GET
* postHandler ---> POST
* putHandler ---> PUT
* deleteHandler ---> DELETE
* headHandler ---> HEAD
* optionsHandler ---> OPTIONS
此处的RestHandler是一个接口，定义如下：  
```java
public interface RestHandler {

    void handleRequest(RestRequest request, RestChannel channel) throws Exception;
}
```
根据RestModule.java的configure()方法可以发现，Rest Module将所有的RestHandler放入一个action list中，该list中的所有handler都会在RestActionMoudule中被绑定，除此之外还将所有Rest处理接口进行bind操作。具体代码参考： org.elasticsearch.rest.action.RestActionModule.java文件。  

