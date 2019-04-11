---
title: netty-socketio对于多节点连接的尝试
date: 2019-04-11 16:00:00
categories:
- java
tags: netty, socketio
---

最近公司有个项目使用了netty-socketio来做实时聊天。但是在部署过程中发现当部署多台机器的时候连接会自动断开，无法建立连接。无奈只能只启动一台机器。但是只有一台机器会有一定风险，所以尝试一下能否解决多台机器断开连接的问题。

而且多节点目前只支持轮询（此处吐槽一下某某云，如果能像nginx那样支持sticky就不用那么麻烦了）

所以新建了一个简单的项目来测试。

项目地址在github上[https://github.com/BOSSzz/socketio-redis-test](https://github.com/BOSSzz/socketio-redis-test)

# 开始
## java代码
首先引入依赖, 使用了最新版本的netty-socketio
```xml
<dependency>
    <groupId>com.corundumstudio.socketio</groupId>
    <artifactId>netty-socketio</artifactId>
    <version>1.7.17</version>
</dependency>
```
定义一个消息实体类：`HelloUid`
```java
public class HelloUid {

    private String uid;

    private String message;

    public String getUid() {
        return uid;
    }

    public void setUid(String uid) {
        this.uid = uid;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```

用来与浏览器传递数据

然后建立socketio服务器
```java
public class SocketIOMain {

    private final static Logger logger = LoggerFactory.getLogger(SocketIOMain.class);

    public static void main(String[] args) throws InterruptedException {
        Configuration config = new Configuration();
        config.setHostname("localhost");
        config.setPort(Integer.parseInt(args[0]));
        SocketIOServer server = new SocketIOServer(config);
        server.addConnectListener(new ConnectListener() {// 添加客户端连接监听器
            @Override
            public void onConnect(SocketIOClient client) {
                logger.info(client.getRemoteAddress() + " web客户端接入");
                client.sendEvent("helloPush", "hello");
            }
        });
        // 握手请求
        server.addEventListener("helloevent", HelloUid.class, new DataListener<HelloUid>() {
            @Override
            public void onData(final SocketIOClient client, HelloUid data, AckRequest ackRequest) {
                // 握手
                String userid = data.getUid();
                logger.info(Thread.currentThread().getName() + "web读取到的userid：" + userid);


                // send message back to client with ack callback
                // WITH data
                client.sendEvent("hellopush", new AckCallback<String>(String.class) {
                    @Override
                    public void onSuccess(String result) {
                        logger.info("ack from client: " + client.getSessionId() + " data: " + result);
                    }
                }, data);


            }
        });

        server.start();

        Thread.sleep(Integer.MAX_VALUE);

        server.stop();
    }
}
```
这样我们的一个简易socketio服务器就完成了。

记得在启动的时候在args参数里指定一下端口号

## 前端代码
在`resource`文件夹中新建html文件
```html
<!DOCTYPE html>
<html>
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <title>Socketio chat</title>
    <script src="./jquery-3.3.1.min.js" type="text/javascript"></script>
    <script type="text/javascript" src="./socket.io/socket.io.js"></script>
    <style>
        body {
            padding: 20px;
        }
        #console {
            height: 400px;
            overflow: auto;
        }
        .username-msg {
            color: orange;
        }
        .connect-msg {
            color: green;
        }
        .disconnect-msg {
            color: red;
        }
        .send-msg {
            color: #888
        }
    </style>
</head>
<body>
<h1>Netty-socketio chat demo</h1>
<br />
<div id="console" class="well"></div>
<form class="well form-inline" onsubmit="return false;">
    <input id="name" class="input-xlarge" type="text" placeholder="用户名称. . . " />
    <input id="msg" class="input-xlarge" type="text" placeholder="发送内容. . . " />
    <button type="button" onClick="sendMessage()" class="btn">Send</button>
    <button type="button" onClick="sendDisconnect()" class="btn">Disconnect</button>
</form>
</body>
<script type="text/javascript">
    var socket = io.connect('http://localhost:8099');
    socket.on('connect',function() {
        output('<span class="connect-msg">Client has connected to the server!</span>');
    });

    socket.on('hellopush', function(data) {
        output('<span class="username-msg">' + data.uid + ' : </span>'
            + data.message);
    });

    socket.on('disconnect',function() {
        output('<span class="disconnect-msg">The client has disconnected! </span>');
    });

    function sendDisconnect() {
        socket.disconnect();
    }

    function sendMessage() {
        var userName = $("#name").val()
        var message = $('#msg').val();
        $('#msg').val('');
        socket.emit('helloevent', {
            uid : userName,
            message : message
        });
    }

    function output(message) {
        var currentTime = "<span class='time' >" + new Date() + "</span>";
        var element = $("<div>" + currentTime + " " + message + "</div>");
        $('#console').prepend(element);
    }
</script>
</html>

```

记得在`resource`目录里放上`jquery`和`socketio`的js文件

![resource情况](https://github.com/BOSSzz/BOSSzz.github.io/blob/master/_posts/images/socketio/resource.png?raw=true)

然后在idea里直接通过浏览器打开即可

![打开html](https://github.com/BOSSzz/BOSSzz.github.io/blob/master/_posts/images/socketio/open.png?raw=true)

之后运行服务端再打开页面就能完成建议的通信了

![效果](https://github.com/BOSSzz/BOSSzz.github.io/blob/master/_posts/images/socketio/effect.png?raw=true)

## 通过nginx模拟多节点轮询
接下来我们使用nginx来模拟多个节点的轮询，安装nginx的过程就不写了

nginx的配置过程参考[nginx+sticky安装及负载均衡使用](https://blog.csdn.net/yyysylvia/article/details/80198021)

注意我们不需要使用sticky，所以可以忽略sticky的安装

最终我的配置如下
```conf

#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    upstream socket_nodes {
        server localhost:10015;
        server localhost:10016;
        #session_sticky;
    }

    server {
        listen       8099;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        location /socket.io {
            proxy_redirect off;
            proxy_buffering off;
            proxy_set_header       Host $host:$server_port;
            proxy_set_header        X-Real-IP $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
            client_max_body_size    10m;
            client_body_buffer_size 128k;
            proxy_connect_timeout   1000;
            proxy_send_timeout      1000;
            proxy_read_timeout      1000;
            proxy_buffer_size       256k;
            proxy_buffers           128 256k;
            proxy_busy_buffers_size 256k;
            proxy_temp_file_write_size 256k;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";
            proxy_pass      http://socket_nodes;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}
    include servers/*;
}
```

有几点需要注意
```conf
upstream socket_nodes {
        server localhost:10015;
        server localhost:10016;
        #session_sticky;
    }
```
这里两个地址是服务端模拟多节点启动的两个端口

```conf
server {
        listen       8099;
        server_name  localhost;
```
这里是nginx监听的端口，也就是页面建立socketio连接所填的端口

## 多服务器启动
之后分别用10015和10016两个端口启动服务器，再打开页面会发现连接一直失败和项目遇到的场景一致

![断开连接](https://github.com/BOSSzz/BOSSzz.github.io/blob/master/_posts/images/socketio/disconnect.png?raw=true)

查阅了socketio连接原理和查看了服务端打印的日志后发现是两次xhr请求发送到了两个服务器导致其中一台在尝试用过sessionId获取client信息时获取不到，从而主动断开了连接。

查阅日志发现了这样一段
```
17:35:50.231 [nioEventLoopGroup-3-12] WARN com.corundumstudio.socketio.transport.WebSocketTransport - Unauthorized client with sessionId: 4933e61f-5df1-4c5e-a223-de22e876b9e0 with ip: /127.0.0.1:58668. Channel closed!
```
而打印这日志的调用在`WebSocketTransport`类中，可见如果没有在`clientsBox`中获取到`ClientHead`，就会直接关闭连接。
```java
private void connectClient(final Channel channel, final UUID sessionId) {
        ClientHead client = clientsBox.get(sessionId);
        if (client == null) {
            log.warn("Unauthorized client with sessionId: {} with ip: {}. Channel closed!",
                        sessionId, channel.remoteAddress());
            channel.close();
            return;
        }

        client.bindChannel(channel, Transport.WEBSOCKET);

        authorizeHandler.connect(client);

        if (client.getCurrentTransport() == Transport.POLLING) {
            SchedulerKey key = new SchedulerKey(SchedulerKey.Type.UPGRADE_TIMEOUT, sessionId);
            scheduler.schedule(key, new Runnable() {
                @Override
                public void run() {
                    ClientHead clientHead = clientsBox.get(sessionId);
                    if (clientHead != null) {
                        if (log.isDebugEnabled()) {
                            log.debug("client did not complete upgrade - closing transport");
                        }
                        clientHead.onChannelDisconnect();
                    }
                }
            }, configuration.getUpgradeTimeout(), TimeUnit.MILLISECONDS);
        }

        log.debug("сlient {} handshake completed", sessionId);
    }
```

而`ClientsBox`代码是这样的
```java
public class ClientsBox {

    private final Map<UUID, ClientHead> uuid2clients = PlatformDependent.newConcurrentHashMap();
    private final Map<Channel, ClientHead> channel2clients = PlatformDependent.newConcurrentHashMap();

    // TODO use storeFactory
    public HandshakeData getHandshakeData(UUID sessionId) {
        ClientHead client = uuid2clients.get(sessionId);
        if (client == null) {
            return null;
        }

        return client.getHandshakeData();
    }

    public void addClient(ClientHead clientHead) {
        uuid2clients.put(clientHead.getSessionId(), clientHead);
    }

    public void removeClient(UUID sessionId) {
        uuid2clients.remove(sessionId);
    }

    public ClientHead get(UUID sessionId) {
        return uuid2clients.get(sessionId);
    }

    public void add(Channel channel, ClientHead clientHead) {
        channel2clients.put(channel, clientHead);
    }

    public void remove(Channel channel) {
        channel2clients.remove(channel);
    }


    public ClientHead get(Channel channel) {
        return channel2clients.get(channel);
    }

}
```
这个类将`uuid`和`ClientHead`的对应关系保存在`uuid2clients`这个map中，所以另一个服务器肯定是获取不到的。

## 解决
那么思路就很简单了，我们将`uuid2clients`这个map保存在redis中不就好了吗。

但是很可惜，`ClientHead`这个类没有实现序列化接口，所以虽然思路不变，但是序列化保存到redis的过程需要修改一下。

所以我们将`ClientHead`这个类拆开，一些成员变量其实是socketio server初始化的，可以不需要保存在redis，将跟随uuid变化的变量保存起来

但是很不幸，我发现其中一个成员变量`HandshakeData`虽然实现了`Serializable`接口，但是他的成员变量`HttpHeaders`是没法序列化的, 无奈只好把header转成map保存

我们通过一个类`ClientHeadData`来保存这些需要存入redis的信息
```java
public class ClientHeadData implements Serializable {

    private static final long serialVersionUID = 238790253873656662L;

    private UUID sessionId;

    private Map<String, List<String>> headers;
    private InetSocketAddress address;
    private Date time;
    private InetSocketAddress local;
    private String url;
    private Map<String, List<String>> urlParams;
    private boolean xdomain;

    private String transport;

    public ClientHeadData(UUID sessionId, Map<String, List<String>> headers, Map<String, List<String>> urlParams, InetSocketAddress address, Date time, InetSocketAddress local, String url, boolean xdomain, String transport) {
        this.sessionId = sessionId;
        this.address = address;
        this.time = time;
        this.local = local;
        this.url = url;
        this.urlParams = urlParams;
        this.headers = new HashMap<>();
        this.headers = headers;
        this.transport = transport;
    }

    public UUID getSessionId() {
        return sessionId;
    }

    public void setSessionId(UUID sessionId) {
        this.sessionId = sessionId;
    }

    public String getTransport() {
        return transport;
    }

    public void setTransport(String transport) {
        this.transport = transport;
    }

    public Map<String, List<String>> getHeaders() {
        return headers;
    }

    public void setHeaders(Map<String, List<String>> headers) {
        this.headers = headers;
    }

    public InetSocketAddress getAddress() {
        return address;
    }

    public void setAddress(InetSocketAddress address) {
        this.address = address;
    }

    public Date getTime() {
        return time;
    }

    public void setTime(Date time) {
        this.time = time;
    }

    public InetSocketAddress getLocal() {
        return local;
    }

    public void setLocal(InetSocketAddress local) {
        this.local = local;
    }

    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public Map<String, List<String>> getUrlParams() {
        return urlParams;
    }

    public void setUrlParams(Map<String, List<String>> urlParams) {
        this.urlParams = urlParams;
    }

    public boolean isXdomain() {
        return xdomain;
    }

    public void setXdomain(boolean xdomain) {
        this.xdomain = xdomain;
    }
}
```

然后我们新建一个类`ClientsRedisBox`继承`ClientsBox`, 里面使用jedis来保存和获取数据

```java
public class ClientsRedisBox extends ClientsBox {

    private final static String CLIENT_HEAD = "clientheads";

    private AckManager ackManager;

    private DisconnectableHub disconnectable;

    private StoreFactory storeFactory;

    private CancelableScheduler disconnectScheduler;

    private Configuration configuration;

    private Jedis jedis;

    public ClientsRedisBox() {
        jedis = new Jedis("localhost", 6379);
    }

    public ClientsRedisBox(AckManager ackManager, DisconnectableHub disconnectable, StoreFactory storeFactory, CancelableScheduler disconnectScheduler, Configuration configuration) {
        this.ackManager = ackManager;
        this.disconnectable = disconnectable;
        this.storeFactory = storeFactory;
        this.disconnectScheduler = disconnectScheduler;
        this.configuration = configuration;
        jedis = new Jedis("localhost", 6379);

    }

    public AckManager getAckManager() {
        return ackManager;
    }

    public void setAckManager(AckManager ackManager) {
        this.ackManager = ackManager;
    }

    public DisconnectableHub getDisconnectable() {
        return disconnectable;
    }

    public void setDisconnectable(DisconnectableHub disconnectable) {
        this.disconnectable = disconnectable;
    }

    public StoreFactory getStoreFactory() {
        return storeFactory;
    }

    public void setStoreFactory(StoreFactory storeFactory) {
        this.storeFactory = storeFactory;
    }

    public CancelableScheduler getDisconnectScheduler() {
        return disconnectScheduler;
    }

    public void setDisconnectScheduler(CancelableScheduler disconnectScheduler) {
        this.disconnectScheduler = disconnectScheduler;
    }

    public Configuration getConfiguration() {
        return configuration;
    }

    public void setConfiguration(Configuration configuration) {
        this.configuration = configuration;
    }

    @Override
    public HandshakeData getHandshakeData(UUID sessionId) {
        return super.getHandshakeData(sessionId);
    }

    @Override
    public void addClient(ClientHead clientHead) {
        super.addClient(clientHead);
        Map<String, List<String>> headers = new HashMap<>();
        for (String name : clientHead.getHandshakeData().getHttpHeaders().names()) {
            List<String> values = new ArrayList<>(clientHead.getHandshakeData().getHttpHeaders().getAll(name));
            headers.put(name, values);
        }
        ClientHeadData clientHeadData = new ClientHeadData(clientHead.getSessionId(), headers,
                clientHead.getHandshakeData().getUrlParams(),
                clientHead.getHandshakeData().getAddress(),
                clientHead.getHandshakeData().getTime(),
                clientHead.getHandshakeData().getLocal(),
                clientHead.getHandshakeData().getUrl(),
                clientHead.getHandshakeData().isXdomain(),
                clientHead.getCurrentTransport().getValue());
        try {
            ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
            ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
            objectOutputStream.writeObject(clientHeadData);
            jedis.hset(CLIENT_HEAD.getBytes(), clientHead.getSessionId().toString().getBytes(), byteArrayOutputStream.toByteArray());
            byteArrayOutputStream.close();
            objectOutputStream.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void removeClient(UUID sessionId) {
        super.removeClient(sessionId);
    }

    @Override
    public ClientHead get(UUID sessionId) {
        ClientHead clientHead = super.get(sessionId);
        if (clientHead == null) {
            try {
                if (!jedis.hexists(CLIENT_HEAD.getBytes(), sessionId.toString().getBytes())) {
                    return null;
                }
                byte[] bytes = jedis.hget(CLIENT_HEAD.getBytes(), sessionId.toString().getBytes());
                ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bytes);
                ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);
                ClientHeadData clientHeadData = (ClientHeadData) objectInputStream.readObject();
                HttpHeaders httpHeaders = new DefaultHttpHeaders();
                clientHeadData.getHeaders().forEach((name, values) -> {
                    httpHeaders.add(name, values);
                });
                HandshakeData handshakeData = new HandshakeData(httpHeaders, clientHeadData.getUrlParams(), clientHeadData.getAddress(),
                        clientHeadData.getLocal(), clientHeadData.getUrl(), clientHeadData.isXdomain());
                clientHead = new ClientHead(sessionId, ackManager, disconnectable, storeFactory, handshakeData,
                        this, Transport.byName(clientHeadData.getTransport()), disconnectScheduler, configuration);
                super.addClient(clientHead);

            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return clientHead;
    }

    @Override
    public void add(Channel channel, ClientHead clientHead) {
        super.add(channel, clientHead);
    }

    @Override
    public void remove(Channel channel) {
        super.remove(channel);
    }

    @Override
    public ClientHead get(Channel channel) {
        return super.get(channel);
    }
}
```

在内存中get不到时，会尝试从redis再获取一次

然后我们新建一个继承了`SocketIOChannelInitializer`的类来替换服务器的初始化过程
```java
@ChannelHandler.Sharable
public class MyPipeLineFactory extends SocketIOChannelInitializer {

    public MyPipeLineFactory() {
        super();
        try {
            Field field = SocketIOChannelInitializer.class.getDeclaredField("clientsBox");
            field.setAccessible(true);
            field.set(this, new ClientsRedisBox());
        } catch (Exception e) {

        }
    }

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) {
        super.handlerAdded(ctx);
    }

    @Override
    public void start(Configuration configuration, NamespacesHub namespacesHub) {
        super.start(configuration, namespacesHub);
        try {
            AckManager ackManager;
            CancelableScheduler disconnectScheduler;
            Field field = SocketIOChannelInitializer.class.getDeclaredField("ackManager");
            field.setAccessible(true);
            ackManager = (AckManager) field.get(this);
            field = SocketIOChannelInitializer.class.getDeclaredField("scheduler");
            field.setAccessible(true);
            disconnectScheduler = (CancelableScheduler) field.get(this);
            field = SocketIOChannelInitializer.class.getDeclaredField("clientsBox");
            field.setAccessible(true);
            ClientsRedisBox clientsRedisBox = (ClientsRedisBox) field.get(this);
            clientsRedisBox.setAckManager(ackManager);
            clientsRedisBox.setConfiguration(configuration);
            clientsRedisBox.setDisconnectable(this);
            clientsRedisBox.setDisconnectScheduler(disconnectScheduler);
            clientsRedisBox.setStoreFactory(configuration.getStoreFactory());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        super.initChannel(ch);
    }

    @Override
    protected void addSslHandler(ChannelPipeline pipeline) {
        super.addSslHandler(pipeline);
    }

    @Override
    protected void addSocketioHandlers(ChannelPipeline pipeline) {
        super.addSocketioHandlers(pipeline);
    }

    @Override
    public void onDisconnect(ClientHead client) {
        super.onDisconnect(client);
    }

    @Override
    public void stop() {
        super.stop();
    }
}
```

可以看到我使用了反射的方法来替换成员变量，因为clientsBox是private的

然后我们在`SocketIOMain`中添加一行代码来使用我们自定义的`PipeLineFactory`
```
server.setPipelineFactory(new MyPipeLineFactory());
```

至此，大功告成！

在启动两个端口，打开页面，就可以看到已经不会自动断开连接了，发送消息也正常，可喜可贺。

## 其他问题
过程中还遇到了一个序列化问题

```
java.io.NotSerializableException: io.netty.handler.codec.HeadersUtils$1
```
这让我非常奇怪，因为这个类根本就没有再需要序列化的过程中。

查阅了资料后发现这是指在`HeadersUtils`类中的第一个匿名内部类（参考[Java 里 Hashmap 序列化的一个坑](https://richardcao.me/2017/05/15/Java-Hashmap-Serializable/)）

也就是源码的这一段
```java
    public static <K, V> List<String> getAllAsString(Headers<K, V, ?> headers, K name) {
        final List<V> allNames = headers.getAll(name);
        return new AbstractList<String>() {
            public String get(int index) {
                V value = allNames.get(index);
                return value != null ? value.toString() : null;
            }

            public int size() {
                return allNames.size();
            }
        };
    }
```
可以看到返回了一个AbstractList，就是这个list不能序列化
索性这是个list，只要new 一个ArrayList就行了
```java
List<String> values = new ArrayList<>(clientHead.getHandshakeData().getHttpHeaders().getAll(name));
```
之前我是没有new ArrayList的，所以会序列化失败

# 参考链接
+ [【websocket】与Spring集成 Netty-SocketIO：最好用的Java版即时消息推送](https://blog.csdn.net/flyaimo/article/details/80107031)
+ [socket.io 原理详解](https://blog.csdn.net/u013243347/article/details/86669988)
+ [基于netty-socketio的web推送服务](https://www.cnblogs.com/luxiaoxun/p/4279997.html)
+ [Java 里 Hashmap 序列化的一个坑](https://richardcao.me/2017/05/15/Java-Hashmap-Serializable/)
+ [nginx+sticky安装及负载均衡使用](https://blog.csdn.net/yyysylvia/article/details/80198021)