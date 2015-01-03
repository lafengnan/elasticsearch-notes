ES模块
=======

## HTTP Server模块
___

HTTP模块定义在org.elasticsearch.http目录中。Guice模块定义在HttpServerModule.java文件中，定义如下：
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

在NettyHttpServerTransport.java文件中，doStart()方法用于启动ES的web服务器。doStart()根据用户配置中是否设置了*blockingserver*启动不同的soket线程。  
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

## Rest模块
____

Rest模块用于接收REST API请求并将其分发到相应的Handler处理。Guice模块文件定义在org.elasticsearch.rest目录中。根据RestModule.java定义：  
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
在*configure*方法中，**RestController**被绑定为单例，并调用RestAction模块bind操作。  

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
根据定义，如果用户配置了回调处理函数则直接创建一个响应返回。否则，判断请求中是否有过滤器。如果存在滤器则在过滤器链中依次处理，否则直接调用*executeHandler(request, channel)*方法。在*executeHandler*方法中，首先获取当前的handler，如果存在已注册的handler则调用handler处理请求，否则判断是否是**OPTIONS**请求，再作相应处理。*executeHandler*方法定义如下：  
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
