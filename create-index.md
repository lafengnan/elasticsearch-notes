# Create Index操作

当一个创建索引的请求被ES接收到之后，该请求经过分发被发送到RestCreateIndexAction.java中的handleRequest方法处理。
RestCreateIndexAction定义如下：  
```java
public class RestCreateIndexAction extends BaseRestHandler {

    @Inject
    public RestCreateIndexAction(Settings settings, RestController controller, Client client) {
        super(settings, controller, client);
        controller.registerHandler(RestRequest.Method.PUT, "/{index}", this);
        controller.registerHandler(RestRequest.Method.POST, "/{index}", this);
    }

    @SuppressWarnings({"unchecked"})
    @Override
    public void handleRequest(final RestRequest request, final RestChannel channel, final Client client) {
        CreateIndexRequest createIndexRequest = new CreateIndexRequest(request.param("index"));
        createIndexRequest.listenerThreaded(false);
        if (request.hasContent()) {
            createIndexRequest.source(request.content());
        }
        createIndexRequest.timeout(request.paramAsTime("timeout", createIndexRequest.timeout()));
        createIndexRequest.masterNodeTimeout(request.paramAsTime("master_timeout", createIndexRequest.masterNodeTimeout()));
        client.admin().indices().create(createIndexRequest, new AcknowledgedRestListener<CreateIndexResponse>(channel));
    }
}
```  
首先将接收到的请求组装成一个**CreateIndexRequest**然后将其作为参数调用client.admin().indices().create()方法。

client.admin().indices()的返回值是一个IndicesAdminClient接口，该接口定义在org.elasticsearch.client.IndicesAdminClient.java中。该接口以一个抽象类AbstractIndicesAdminClient将其实现，并被InternalTransportIndicesAdminClient继承并实例化。这里的create()方法定义如下：  
```java
 @Override
    public ActionFuture<CreateIndexResponse> create(final CreateIndexRequest request) {
        return execute(CreateIndexAction.INSTANCE, request);
    }
```
create()方法真正执行的是execute()方法，该方法定义在ElasticsearchClient接口中(org.elasticsearch.client.ElasticsearchClient.java)。  ElasticsearchClient接口的实现层次如下： 

**interface**  ElasticsearchClient, 成员方法：  
* ActionFuture<Response> execute(final Action<Request, Response, RequestBuilder, Client> action, final Request request)  
* void execute(final Action<Request, Response, RequestBuilder, Client> action, final Request request, ActionListener<Response> listener);  

**interface** IndicesAdmClient *extends* ElasticsearchClient, 成员方法：  
* ActionFuture<CreateIndexResponse> create(CreateIndexRequest request);  
* 其他  
* 
**interface** Client *extends* ElasticsearchClient，成员方法略  

**abstract class** AbstractClient *implements* Client，成员方法：  
* 略  

**abstract class** AbsctractIndicesAdminClient *implements* IndicesAdmClient, 成员方法如下：  
* create  
```java
 public ActionFuture<CreateIndexResponse> create(final CreateIndexRequest request) {
        return execute(CreateIndexAction.INSTANCE, request);
    }
```
* 其他  

**class** InternalTransportIndicesAdminClient *extends* AbstractIndicesAdminClient *implements* IndicesAdminClient，成员方法：  
* ActionFuture create(action, request)  
```java
@SuppressWarnings("unchecked")
@Override
public <Request extends ActionRequest, Response extends ActionResponse, RequestBuilder extends ActionRequestBuilder<Request, Response, RequestBuilder, IndicesAdminClient>> ActionFuture<Response> execute(final Action<Request, Response, RequestBuilder, IndicesAdminClient> action, final Request request) {
        PlainActionFuture<Response> actionFuture = PlainActionFuture.newFuture();
        execute(action, request, actionFuture);
        return actionFuture;
    }
```
* void create(action, request, listener)  
```java
@SuppressWarnings("unchecked")
@Override
public <Request extends ActionRequest, Response extends ActionResponse, RequestBuilder extends ActionRequestBuilder<Request, Response, RequestBuilder, IndicesAdminClient>> void execute(final Action<Request, Response, RequestBuilder, IndicesAdminClient> action, final Request request, ActionListener<Response> listener) {
        headers.applyTo(request);
        final TransportActionNodeProxy<Request, Response> proxy = actions.get(action);
        nodesService.execute(new TransportClientNodesService.NodeListenerCallback<Response>() {
            @Override
            public void doWithNode(DiscoveryNode node, ActionListener<Response> listener) {
                proxy.execute(node, request, listener);
            }
        }, listener);
    }
```
由上述代码流程可以跟踪到create()方法最终调用TransportClientNodesService的execute()方法。该方法定义如下：  
```java
public <Response> void execute(NodeListenerCallback<Response> callback, ActionListener<Response> listener) throws ElasticsearchException {
        ImmutableList<DiscoveryNode> nodes = this.nodes;
        ensureNodesAreAvailable(nodes);
        int index = getNodeNumber();
        RetryListener<Response> retryListener = new RetryListener<>(callback, listener, nodes, index);
        DiscoveryNode node = nodes.get((index) % nodes.size());
        try {
            callback.doWithNode(node, retryListener);
        } catch (Throwable t) {
            //this exception can't come from the TransportService as it doesn't throw exception at all
            listener.onFailure(t);
        }
    }
```
该方法的逻辑相对简单：  
1. 首先获取当前cluster中的所有节点并将其存入一个列表，然后验证节点列表是否为空
2. 获取一个递增的随机数作为节点索引号index
3. 创建一个RetryListener实例
4. 从节点列表中根据index % nodes.length获取一个节点  
5. 调用回调函数，即TransportClientNodesService.NodeListenerCallback.doWithNode()
6. 在doWithNode()中调用proxy.execute()方法
关step6， proxy在execute()方法中实例化：  
```java
final TransportActionNodeProxy<Request, Response> proxy = actions.get(action);
```  
即，最终调用TransportActionNodeProxy的execute()方法。  该方法定义如下：  
```java
 public void execute(DiscoveryNode node, final Request request, final ActionListener<Response> listener) {
        ActionRequestValidationException validationException = request.validate();
        if (validationException != null) {
            listener.onFailure(validationException);
            return;
        }
        transportService.sendRequest(node, action.name(), request, transportOptions, new BaseTransportResponseHandler<Response>() {
            @Override
            public Response newInstance() {
                return action.newResponse();
            }

            @Override
            public String executor() {
                if (request.listenerThreaded()) {
                    return ThreadPool.Names.LISTENER;
                }
                return ThreadPool.Names.SAME;
            }

            @Override
            public void handleResponse(Response response) {
                listener.onResponse(response);
            }

            @Override
            public void handleException(TransportException exp) {
                listener.onFailure(exp);
            }
        });
    }
```
