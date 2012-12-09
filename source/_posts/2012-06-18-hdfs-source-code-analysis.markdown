---
layout: post
title: "HDFS Source Code Analysis"
date: 2012-06-18 19:28
comments: true
categories: 
published: true
---

1. [创建FileSystem](#instantiate_FileSystem)
2. [HDFS客户端向服务器发送rpc请求](#hdfs_client_send_request)
3. [HDFS服务器对客户端rpc请求的处理](#hdfs_server_handle_client_request)
4. [DataNode中和数据相关的类](#hdfs_datanode_storage_related_class)
5. [DataNode的启动](#hdfs_datanode_start)
5. [参考资料](#reference)
<!-- more -->

## <a id="instantiate_FileSystem"></a>创建FileSystem
FileSystem在Hadoop中只是一个抽象类，它屏蔽了各种底层文件系统的实现，这有点Linux VFS的味道。在Hadoop中各个文件系统的实现都是继承了FileSystem这个抽象类的，因此用户只需要通过FileSystem的接口来操作底层的文件系统就可以了。Hadoop中具体的文件系统实现有很多，如DistributedFileSystem和LocalFileSystem。如何告知Hadoop要使用哪一个具体的文件系统实现是这节要讨论的。

在这里我们可以通过在Hadoop的配置文件里指定fs.default.name来实现，当然也可以在程序中显式指定。假设我们指定fs.default.name的值为hdfs://localhost:9000，其中的*hdfs*就是告诉Hadoop要使用DistributedFileSystem。Hadoop会从fs.default.name中抽出hdfs，然后去配置文件里找fs.*hdfs*.impl，这个可以在core-default.xml中找到，值为org.apache.hadoop.hdfs.DistributedFileSystem，可以看出Hadoop已经成功找到具体要创建的文件系统实现是什么了，接下来Hadoop通过反射就能由类名创建出类实例来，然后返回给用户使用。类似的如果我们指定fs.default.name的值为file:///，那么Hadoop就会去配置文件里找fs.*file*.impl，这个的值就是org.apache.hadoop.fs.LocalFileSystem。当然我们还能给fs.default.name指定很多其他的值来告诉Hadoop要使用的底层文件系统实现是什么。在这里还要说明的是fs.default.name的默认值就是file:///。接下来给出一些关键代码：
``` java
/** FileSystem.java **/

/* 创建FileSystem的工厂方法 */
public static FileSystem get(Configuration conf) throws IOException {
    return get(getDefaultUri(conf), conf);
}

/* 获取配置文件中fs.default.name的值，并转化成一个URI对象 */
public static URI getDefaultUri(Configuration conf) {
    return URI.create(fixName(conf.get(FS_DEFAULT_NAME_KEY, "file:///"))); // FS_DEFAULT_NAME_KEY的值就是fs.default.name
}

FileSystem get(URI uri, Configuration conf) throws IOException{
    Key key = new Key(uri, conf);
    FileSystem fs = null;
    synchronized (this) {
        fs = map.get(key);
    }
    if (fs != null) {
        return fs;
    }

    fs = createFileSystem(uri, conf); // 只关注这个函数
    synchronized (this) {  // refetch the lock again
        FileSystem oldfs = map.get(key);
        if (oldfs != null) { // a file system is created while lock is releasing
            fs.close(); // close the new file system
            return oldfs;  // return the old file system
        }

        // now insert the new file system into the map
        if (map.isEmpty() && !clientFinalizer.isAlive()) {
            Runtime.getRuntime().addShutdownHook(clientFinalizer);
        }
        fs.key = key;
        map.put(key, fs);
        return fs;
    }
}

private static FileSystem createFileSystem(URI uri, Configuration conf
        ) throws IOException {
    // 从fs.default.name的值中抽取出scheme部分，也就是开头的那部分。然后从配置文件中找fs.**.impl的值，也就是要实例的类的名字
    Class<?> clazz = conf.getClass("fs." + uri.getScheme() + ".impl", null); 
    LOG.debug("Creating filesystem for " + uri);
    if (clazz == null) {
        throw new IOException("No FileSystem for scheme: " + uri.getScheme());
    }
    // 反射，根据类名创建类实例
    FileSystem fs = (FileSystem)ReflectionUtils.newInstance(clazz, conf);
    fs.initialize(uri, conf);
    return fs;
}
```

## <a id="hdfs_client_send_request"></a>HDFS客户端向服务器发送rpc请求
对于HDFS来说我们是通过DistributedFileSystem来进行文件系统的操作的，在这里DistributedFileSystem就是client，而server就跑在NameNode上面。我们通过DistributedFileSystem发出的所有操作都会通过rpc传递到NameNode的server上，然后再执行最后将结果返回。这就是client和server之间的rpc。

这节谈论client如何将用户对DistributedFileSystem的方法调用传递到server端。在这里我们以`mkdirs()`方法为例，假设用户调用了DistributedFileSystem的`mkdirs()`方法，代码如下：
``` java
public boolean mkdirs(Path f, FsPermission permission) throws IOException {
    statistics.incrementWriteOps(1);
    return dfs.mkdirs(getPathName(f), permission);
}
```

可以看到DistributedFileSystem将方法调用转交给了实例变量dfs，这个就是DFSClient的实例，这个是rpc的一个关键点，官方文档对这个类的描述是

> DFSClient can connect to a Hadoop Filesystem and perform basic file tasks. It uses the ClientProtocol to communicate with a NameNode daemon, and connects directly to DataNodes to read/write block data. Hadoop DFS users should obtain an instance of DistributedFileSystem, which uses DFSClient to handle filesystem tasks.

可以看出DistributedFileSystem将所有对文件系统的操作都转交给了DFSClient来处理。接下来我们就看看DFSClient是怎么处理的。代码如下：
``` java
/* DistributedFileSystem.java */
public void initialize(URI uri, Configuration conf) throws IOException {
    super.initialize(uri, conf);
    setConf(conf);

    String host = uri.getHost();
    if (host == null) {
        throw new IOException("Incomplete HDFS URI, no host: "+ uri);
    }

    InetSocketAddress namenode = NameNode.getAddress(uri.getAuthority());
    this.dfs = new DFSClient(namenode, conf, statistics); // 创建了DFSClient的实例
    this.uri = URI.create(uri.getScheme()+"://"+uri.getAuthority());
    this.workingDir = getHomeDirectory();
}

/* DFSClient.java */
public boolean mkdirs(String src, FsPermission permission)throws IOException{
    // omitted
    return namenode.mkdirs(src, masked);
    // omitted
}

/** 
* Create a new DFSClient connected to the given nameNodeAddr or rpcNamenode.
* Exactly one of nameNodeAddr or rpcNamenode must be null.
*/
DFSClient(InetSocketAddress nameNodeAddr, ClientProtocol rpcNamenode,
  Configuration conf, FileSystem.Statistics stats)
throws IOException {
    // omitted
    this.rpcNamenode = createRPCNamenode(nameNodeAddr, conf, ugi);
    this.namenode = createNamenode(this.rpcNamenode);   
    // omitted
}

private static ClientProtocol createRPCNamenode(InetSocketAddress nameNodeAddr,
  Configuration conf, UserGroupInformation ugi) 
    throws IOException {
    return (ClientProtocol)RPC.getProxy(ClientProtocol.class,
        ClientProtocol.versionID, nameNodeAddr, ugi, conf,
        NetUtils.getSocketFactory(conf, ClientProtocol.class));
}

/* RPC.java */
/** Construct a client-side proxy object that implements the named protocol,
 * talking to a server at the named address. */
public static VersionedProtocol getProxy(
        Class<? extends VersionedProtocol> protocol,
        long clientVersion, InetSocketAddress addr, UserGroupInformation ticket,
        Configuration conf, SocketFactory factory, int rpcTimeout) throws IOException {

    if (UserGroupInformation.isSecurityEnabled()) {
        SaslRpcServer.init(conf);
    }
    VersionedProtocol proxy =
        (VersionedProtocol) Proxy.newProxyInstance(
                protocol.getClassLoader(), new Class[] { protocol },
                new Invoker(protocol, addr, ticket, conf, factory, rpcTimeout));
    long serverVersion = proxy.getProtocolVersion(protocol.getName(), 
            clientVersion);
    if (serverVersion == clientVersion) {
        return proxy;
    } else {
        throw new VersionMismatch(protocol.getName(), clientVersion, 
                serverVersion);
    }
}

private static ClientProtocol createNamenode(ClientProtocol rpcNamenode)
    throws IOException {
    RetryPolicy createPolicy = RetryPolicies.retryUpToMaximumCountWithFixedSleep(
            5, LEASE_SOFTLIMIT_PERIOD, TimeUnit.MILLISECONDS);

    Map<Class<? extends Exception>,RetryPolicy> remoteExceptionToPolicyMap =
        new HashMap<Class<? extends Exception>, RetryPolicy>();
    remoteExceptionToPolicyMap.put(AlreadyBeingCreatedException.class, createPolicy);

    Map<Class<? extends Exception>,RetryPolicy> exceptionToPolicyMap =
        new HashMap<Class<? extends Exception>, RetryPolicy>();
    exceptionToPolicyMap.put(RemoteException.class, 
            RetryPolicies.retryByRemoteException(
                RetryPolicies.TRY_ONCE_THEN_FAIL, remoteExceptionToPolicyMap)); // 控制retry的策略 TRY_ONCE_THEN_FAIL
    RetryPolicy methodPolicy = RetryPolicies.retryByException(
            RetryPolicies.TRY_ONCE_THEN_FAIL, exceptionToPolicyMap);
    Map<String,RetryPolicy> methodNameToPolicyMap = new HashMap<String,RetryPolicy>();

    methodNameToPolicyMap.put("create", methodPolicy);

    return (ClientProtocol) RetryProxy.create(ClientProtocol.class,
            rpcNamenode, methodNameToPolicyMap);
}
```
可以看到DFSClient又将方法调用转交给了实例变量namenode，于是我们就要了解namenode和与其有关的rpcNamenode这两个实例变量。对于DFSClient的构造函数，这里只显示最重要的两样东西分别是namenode和rpcNamenode。要了解这两样东西首先需要了解Java的[Dynamic Proxy](http://docs.oracle.com/javase/1.4.2/docs/guide/reflection/proxy.html)。我们可以看到rpcNamenode实现了ClientProtocol接口，这个接口就规定了client能向server调用的所有方法。按照Dynamic Proxy的机制我们知道所有对rpcNamenode的方法调用全部归结到了Invoker的`invoke()`方法上了，这个方法就是将client的调用信息如调用的方法，参数等通过socket传递给server，并获得返回结果。对于Invoker的`invoke()`方法的剖析这里就不在描述了，可以通过参考链接了解更多。至此client的方法调用就顺利传递到server了。照这样看DFSClient就只要调用rpcNamenode的`mkdirs()`方法就好了，为什么还要namenode呢？要了解这个就需要RetryProxy的代码了：
``` java
/* RetryProxy.java */
public static Object create(Class<?> iface, Object implementation,
        Map<String,RetryPolicy> methodNameToPolicyMap) {
    return Proxy.newProxyInstance(
            implementation.getClass().getClassLoader(),
            new Class<?>[] { iface },
            new RetryInvocationHandler(implementation, methodNameToPolicyMap)
            );
}

/* RetryInvocationHandler.java */
public Object invoke(Object proxy, Method method, Object[] args)
    throws Throwable {
    RetryPolicy policy = methodNameToPolicyMap.get(method.getName());
    if (policy == null) {
        policy = defaultPolicy;
    }

    int retries = 0;
    while (true) {
        try {
            return invokeMethod(method, args);
        } catch (Exception e) {
            if (!policy.shouldRetry(e, retries++)) {
                LOG.info("Exception while invoking " + method.getName()
                        + " of " + implementation.getClass() + ". Not retrying."
                        + StringUtils.stringifyException(e));
                if (!method.getReturnType().equals(Void.TYPE)) {
                    throw e; // non-void methods can't fail without an exception
                }
                return null;
            }
            LOG.debug("Exception while invoking " + method.getName()
                    + " of " + implementation.getClass() + ". Retrying."
                    + StringUtils.stringifyException(e));
        }
    }
}

private Object invokeMethod(Method method, Object[] args) throws Throwable {
    try {
        if (!method.isAccessible()) {
            method.setAccessible(true);
        }
        return method.invoke(implementation, args);
    } catch (InvocationTargetException e) {
        throw e.getCause();
    }
}
```
可以看到对于namenode的方法调用都归结到RetryInvocationHandler的`invoke()`上了，而这个方法就通过反射再调用rpcNamenode的对应方法，只不过它会retry，也就是说如果调用失败的话，它还会重新尝试调用，这么一来可靠性就增强了，毕竟网络传输很容易出问题的。如果让DFSClient直接调用rpcNamenode的话，一旦失败就会立刻告诉用户，而其实如果多尝试几次还是可能会成功的，这就是为什么要引入namenode的原因了。至此我们就知道client是如何将用户对DistributedFileSystem的方法调用传递到server端的，最后用一张图来总结。

{% img /images/post/hdfs-source-code-analysis/hdfs_client_send_request.png %}

参考链接：
<a href="http://blog.csdn.net/historyasamirror/article/details/6159248">1</a>
<a href="http://caibinbupt.iteye.com/blog/280790">2</a>
<a href="http://caibinbupt.iteye.com/blog/281281">3</a>
<a href="http://caibinbupt.iteye.com/blog/281476">4</a>
<a href="http://http://blog.csdn.net/xhh198781/article/details/7268298">5</a>
<a href="http://assets.en.oreilly.com/1/event/12/HDFS%20Under%20the%20Hood%20Presentation%201.pdf">6</a>

## <a id="hdfs_server_handle_client_request"></a> HDFS服务器对客户端请求的处理
上面我们提到了客户端如何将rpc方法调用的请求传送给了HDFS服务器也就是namenode，在这一节就要看一下服务器是如何接受请求并处理的。首先看的是org.apache.hadoop.hdfs.server.namenode的代码，这个类代表了HDFS中的namenode。
``` java
/* NameNode.java */
private void initialize(Configuration conf) throws IOException {
    // omitted 
    this.serviceRpcServer = RPC.getServer(this, dnSocketAddr.getHostName(), 
            dnSocketAddr.getPort(), serviceHandlerCount,
            false, conf, namesystem.getDelegationTokenSecretManager()); // handle request from datanode
            
    this.server = RPC.getServer(this, socAddr.getHostName(), // namenode将自己的实例传进去了，将来server就会调用namenode的方法来完成请求，比如namenode.mkdirs
        socAddr.getPort(), handlerCount, false, conf, namesystem
        .getDelegationTokenSecretManager()); // handle request from client
        
    startHttpServer(conf); // handle request from http 

    this.server.start();  //start RPC server   
    if (serviceRpcServer != null) {
      serviceRpcServer.start();      
    }
    // omitted
}
```
可以看到在初始化NameNode时一共开启了三个服务器，其中serviceRpcServer是用来处理datanode的请求，比如datanode要定期发送心跳包。server是用来处理client的请求。最后还有一个http服务器。因此我们这里只关心server服务器而忽略其他两个。
``` java
/* RPC.java */
/** Construct a server for a protocol implementation instance listening on a
 * port and address, with a secret manager. */
public static Server getServer(final Object instance, final String bindAddress, final int port,
        final int numHandlers,
        final boolean verbose, Configuration conf,
        SecretManager<? extends TokenIdentifier> secretManager) 
throws IOException {
    return new Server(instance, conf, bindAddress, port, numHandlers, verbose, secretManager);
}

public static class Server extends org.apache.hadoop.ipc.Server { // RPC的一个静态内部类
    public Server(Object instance, Configuration conf, String bindAddress,  int port,
            int numHandlers, boolean verbose, 
            SecretManager<? extends TokenIdentifier> secretManager) throws IOException {
        super(bindAddress, port, Invocation.class, numHandlers, conf,
                classNameBase(instance.getClass().getName()), secretManager);
        this.instance = instance;
        this.verbose = verbose;
    }
}
```
最终建立的是org.apache.hadoop.ipc.Server的一个子类，这个类只是实现了父类的抽象方法`call()`，其他的功能都由父类提供了。因此我们主要专注org.apache.hadoop.ipc.Server类
``` java
/* Server.java */
public abstract class Server {
  private Listener listener = null;
  private Responder responder = null;
  private int numConnections = 0;
  private Handler[] handlers = null;
  private BlockingQueue<Call> callQueue; // queued calls

  /** Starts the service.  Must be called before any calls will be handled. */
  public synchronized void start() {
      responder.start();
      listener.start();
      handlers = new Handler[handlerCount];

      for (int i = 0; i < handlerCount; i++) {
          handlers[i] = new Handler(i);
          handlers[i].start();
      }
  }
}
```
这里就截取了比较重要的代码片段，其中包含这一节要主要介绍的Listener，Responder，Handler，它们就是Server中的3大组件。这三个类都继承了Thread，也就是说它们的每个实例都将是一个线程。下面分别来介绍这三大组件。

### Listener
首先我们先来看一下Listener的一部分源码
``` java
/* Server.java */
/** Listens on the socket. Creates jobs for the handler threads*/
private class Listener extends Thread {
    private ServerSocketChannel acceptChannel = null; //the accept channel
    private Selector selector = null; //the selector that we use for the server
    private Reader[] readers = null;
    // omitted

    public Listener() throws IOException {
        address = new InetSocketAddress(bindAddress, port);
        // Create a new server socket and set to non blocking mode
        acceptChannel = ServerSocketChannel.open();
        acceptChannel.configureBlocking(false);

        // Bind the server socket to the local host and port
        bind(acceptChannel.socket(), address, backlogLength);
        port = acceptChannel.socket().getLocalPort(); //Could be an ephemeral port
        // create a selector;
        selector= Selector.open();
        readers = new Reader[readThreads];
        readPool = Executors.newFixedThreadPool(readThreads);
        for (int i = 0; i < readThreads; i++) {
            Selector readSelector = Selector.open();
            Reader reader = new Reader(readSelector);
            readers[i] = reader;
            readPool.execute(reader);
        }

        // Register accepts on the server socket with the selector.
        acceptChannel.register(selector, SelectionKey.OP_ACCEPT);
        this.setName("IPC Server listener on " + port);
        this.setDaemon(true);
    }

    public void run() {
        LOG.info(getName() + ": starting");
        SERVER.set(Server.this);
        while (running) {
            SelectionKey key = null;
            try {
                selector.select();
                Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
                while (iter.hasNext()) {
                    key = iter.next();
                    iter.remove();
                    try {
                        if (key.isValid()) {
                            if (key.isAcceptable())
                                doAccept(key);
                        }
                    } catch (IOException e) {
                    }
                key = null;
              }
            } catch (OutOfMemoryError e) {
                // omitted
            }
        }
        // omitted
    }
    
    void doAccept(SelectionKey key) throws IOException,  OutOfMemoryError {
        Connection c = null;
        ServerSocketChannel server = (ServerSocketChannel) key.channel();
        SocketChannel channel;
        while ((channel = server.accept()) != null) {
            channel.configureBlocking(false);
            channel.socket().setTcpNoDelay(tcpNoDelay);
            Reader reader = getReader(); // 为了让每个reader尽可能平均地分配到channel，目前采用的是round robin方法
            try {
                reader.startAdd();
                SelectionKey readKey = reader.registerChannel(channel); // 将这个channel交由reader来处理
                c = new Connection(readKey, channel, System.currentTimeMillis());
                readKey.attach(c);
                synchronized (connectionList) {
                    connectionList.add(numConnections, c);
                    numConnections++;
                }
            } finally {
                reader.finishAdd(); 
            }
        }
    }
}
```
Listener主要做的事就是处理客户端的连接请求，它在acceptChannel(相当于listening socket)上监听客户端的连接请求，然后创建一个新的channel(相当于connected socket)作为接下来该客户端和服务器的通讯通道，然后将这个channel分配到某个reader上，让其来处理后续的客户端请求。同时还创建了一个Connection来表示客户端和服务器建立起来的连接，这个类内部有channel的引用，因此其知道这个连接底层的channel(或者说socket)是什么。为了要完全理解上面的代码还需要知道的是Java NIO中的[Selector](http://tutorials.jenkov.com/java-nio/selectors.html)。  
我们现在知道真正获取(读取)客户端rpc请求的是Reader类，它实现了Runnable接口即作为一个线程运行，这个类在Listener中只有固定多的实例(可以通过配置文件修改)，因此需要一个reader处理多个客户端的请求，这个在Selector的帮助下将会很容易实现，Reader类中有一个readerSelector的实例来帮助reader处理请求。下面看一下Reader的部分源代码。
``` java
/* Server.java */
private class Reader implements Runnable {
    private Selector readSelector = null;
    public void run() {
        synchronized (this) {
            while (running) {
                SelectionKey key = null;
                try {
                    // 下面都是Selector的典型用法，关键是doRead()函数，这个函数会处理客户端传来的数据
                    readSelector.select(); // 当有channel可读后返回
                    Iterator<SelectionKey> iter = readSelector.selectedKeys().iterator();
                    while (iter.hasNext()) {
                        key = iter.next();
                        iter.remove();
                        if (key.isValid()) {
                            if (key.isReadable()) {
                                doRead(key);
                            }
                        }
                        key = null;
                    }
                } catch (InterruptedException e) {
                        // omitted
                } catch (IOException ex) {
                    LOG.error("Error in Reader", ex);
                }
            }
        }
    }

    public synchronized SelectionKey registerChannel(SocketChannel channel)
        throws IOException {
            return channel.register(readSelector, SelectionKey.OP_READ);
    }
}

void doRead(SelectionKey key) throws InterruptedException {
    int count = 0;
    // 找出目前处理连接对应的Connection，上面提到过对于每一个已建立的连接都会生成一个Connection来表示
    // 关于Connection类后面还会详细讨论
    Connection c = (Connection)key.attachment(); 

    try {
        count = c.readAndProcess(); // 交给对应的Connection来处理
    } catch (InterruptedException ieo) {
        // omitted
    } catch (Exception e) {
        count = -1; //so that the (count < 0) block is executed
    }
    if (count < 0) {
        closeConnection(c);
        c = null;
    }
    else {
        c.setLastContact(System.currentTimeMillis());
    }
}   
```
可以看到Reader做的就是不停地找它所管理的客户端连接中有数据可读的，然后找到这个连接对应的Connection对象，交由相应的connection来处理，于是目光就转向了Connection类。之前说过Connection类表示一个客户端已建立的连接，这个类有一个channel的引用，表示这个连接底层使用的channel(可以理解成socket)。其实Connection类的内容远不止这些，接下来就看看Connection类的代码。
``` java
/* Server.java */
public class Connection {
    private SocketChannel channel; // 这个连接底层的channel
    // 对于客户端的每个rpc函数调用都会生成Call对象，responseQueue保存的是已经处理完的call
    // call对象里有处理完后的结果，之后要传回给客户端的
    private LinkedList<Call> responseQueue; 

    // number of outstanding rpcs 还没处理完的rpc请求，这里的处理完指的是将结果发回给客户端
    // 客户端可能会一下子发送多个rpc请求，然后等待所有的结果
    private volatile int rpcCount = 0; 
    
    // readAndProcess()做的主要工作是读取一个rpc请求的所有数据
    public int readAndProcess() throws IOException, InterruptedException {
        // omitted
        processOneRpc(data.array()); // data就是从channel中获得的一个rpc请求的所有数据
        // omitted
    }

    private void processOneRpc(byte[] buf) throws IOException,InterruptedException {
        // omitted
        processData(buf);
        // omitted
    }

    private void processData(byte[] buf) throws  IOException, InterruptedException {
      DataInputStream dis =
        new DataInputStream(new ByteArrayInputStream(buf));
      int id = dis.readInt();                    // try to read an id
        
      Writable param = ReflectionUtils.newInstance(paramClass, conf);//read param
      param.readFields(dis);        
  
      Call call = new Call(id, param, this);
      callQueue.put(call);              // queue the call; maybe blocked here
      incRpcCount();  // Increment the rpc count
    }
```
代码跟踪到这就差不多结束了，这些代码大致做了如下的事：首先是`readAndProcess()`读取一个rpc请求的所有数据，然后一直传到`processData()`函数处理，`processData()`函数首先把这些数据转成一个Call对象，这其实可以和上一节介绍的客户端向服务器发送rpc请求对应起来看，可以看作一个序列化和解序列化的过程。这个Call对象就包含了调用的函数名和函数参数，同时之前也提到过Call对象还会保存函数调用过后的结果即返回值。最后将Call对象放入callQueue队列中等待被处理。如果上面还记得Server的代码的话就可以发现callQueue实际上是Server的一个实例变量。我们讨论到现在的Listener，Reader，Connection类和之后要讨论的Handler，Responder都是这个Server的内部类，所以对这些类来说callQueue就相当于一个全局变量了。之前我们提到Listener会有多个Reader来读取很多客户端的rpc请求，所有这些请求被转化成Call对象后都会放入这个全局的callQueue队列中等待被处理。好了，至此Listener组件就全部讲完了，下面是Handler组件。

### Handler
我们现在知道callQueue中包含了客户端对服务器的rpc调用请求，如何处理这些请求就是Handler的事了。看之前Server的源码我们可以知道Server中存在多个Handler，每个Handler都是一个线程，它们并发地从callQueue中取出一个一个的Call然后进行处理，下面来看看源代码。
``` java
/* Server.java  */
/** Handles queued calls . */
private class Handler extends Thread {
    public void run() {
        ByteArrayOutputStream buf = 
            new ByteArrayOutputStream(INITIAL_RESP_BUF_SIZE);
        while (running) {
            try {
                final Call call = callQueue.take(); // pop the queue; maybe blocked here 从callQueue中取出一个call进行处理 

                Writable value = null;

                CurCall.set(call);
                try {
                    // Make the call as the user via Subject.doAs, thus associating
                    // the call with the Subject
                    if (call.connection.user == null) {
                        value = call(call.connection.protocol, call.param, call.timestamp); // 处理这个call，也就是找到一个合适的对象调用客户端所要求的方法，并传入参数，最终获得结果
                    } else {
                        value = 
                            call.connection.user.doAs
                            (new PrivilegedExceptionAction<Writable>() {
                                 public Writable run() throws Exception {
                                 // make the call
                                 return call(call.connection.protocol, call.param, call.timestamp);
                                 }
                             }
                            );
                    }
                } catch (Throwable e) {
                    // omitted
                }
                CurCall.set(null);
                synchronized (call.connection.responseQueue) {
                    // setupResponse() needs to be sync'ed together with 
                    // responder.doResponse() since setupResponse may use
                    // SASL to encrypt response data and SASL enforces
                    // its own message ordering.
                    setupResponse(buf, call, 
                            (error == null) ? Status.SUCCESS : Status.ERROR, 
                            value, errorClass, error); // 设置好返回值，也就是说把返回值等信息保存到call对象里。还记得吗，call对象能保存这个call的执行结果
                    // Discard the large buf and reset it back to 
                    // smaller size to freeup heap
                    if (buf.size() > maxRespSize) {
                      buf = new ByteArrayOutputStream(INITIAL_RESP_BUF_SIZE);
                    }
                    responder.doRespond(call); // 处理返回结果
                }
            } catch (InterruptedException e) {
                if (running) {                          // unexpected -- log it
                    // omitted
                }
            } catch (Exception e) {
                // omitted
            }
        }
    }
}
```
可以看到每个Handler线程做的就是从callQueue中取一个call，然后调用Server的call方法处理并获得结果，我们之前提到过这个call方法是抽象方法由具体的子类来实现，这个具体实现的子类我们之前也提到过在RPC.java文件中，具体的代码就不列出来了，只说一下它做了什么。具体call方法做的就是利用反射调用NameNode类的具体方法，在这里就是调用了`mkdirs()`，大家可以看看是不是NameNode里就有这个方法。获得完函数调用的结果后就把它放入对应的call对象里，然后调用`responder.doRespond()`方法来处理，于是就引出了Responder这个第三大组件。

``` java
/* Server.java */
private class Responder extends Thread {
    private Selector writeSelector; // 和readSelector异曲同工

    void doRespond(Call call) throws IOException {
        synchronized (call.connection.responseQueue) {
            // 将这个call放入其所属的connection(也就是请求这个call的客户端所表示的connection)的responseQueue中等待被处理
            // responseQueue在之前的Connection的源代码中也有介绍，它和callQueue很类似，只不过一个是全局的，一个是每个Connection都会有一个
            call.connection.responseQueue.addLast(call); 
            if (call.connection.responseQueue.size() == 1) {
                processResponse(call.connection.responseQueue, true);
            }
        }
    }

    // processResponse()主要做的就是从responseQueue中取一个call，然后获得这个call的调用结果，还记得当时保存进去了吗
    // 将调用结果传回到客户端，至此一个rpc请求调用圆满完成
    // 这个函数可能会在两种情况下被调用：一种是在Handler线程中被调用，就像上面的代码所示，另一种就是在Responder线程中被调用，这个待会会见到
    // inHandler区别是在哪种情况下被调用
    private boolean processResponse(LinkedList<Call> responseQueue, boolean inHandler) throws IOException {
        boolean error = true;
        boolean done = false;       // there is more data for this channel.
        int numElements = 0;
        Call call = null;
        try {
            synchronized (responseQueue) {
                //
                // If there are no items for this channel, then we are done
                //
                numElements = responseQueue.size();
                if (numElements == 0) {
                    error = false;
                    return true;              // no more data for this channel.
                }
                //
                // Extract the first call
                //
                call = responseQueue.removeFirst();
                SocketChannel channel = call.connection.channel;
                //
                // Send as much data as we can in the non-blocking fashion
                //
                int numBytes = channelWrite(channel, call.response);
                if (numBytes < 0) {
                    return true;
                }
                if (!call.response.hasRemaining()) {
                    call.connection.decRpcCount();
                    if (numElements == 1) {    // last call fully processes.
                        done = true;             // no more data for this channel.
                    } else {
                        done = false;            // more calls pending to be sent.
                    }
                } else {
                    //
                    // If we were unable to write the entire response out, then 
                    // insert in Selector queue. 
                    //
                    call.connection.responseQueue.addFirst(call);

                    if (inHandler) {
                        // set the serve time when the response has to be sent later
                        call.timestamp = System.currentTimeMillis();

                        incPending();
                        try {
                            // Wakeup the thread blocked on select, only then can the call 
                            // to channel.register() complete.
                            writeSelector.wakeup();
                            channel.register(writeSelector, SelectionKey.OP_WRITE, call); // 让Responder来管理这个channel
                        } catch (ClosedChannelException e) {
                            //Its ok. channel might be closed else where.
                            done = true;
                        } finally {
                            decPending();
                        }
                    }
                }
                error = false;              // everything went off well
            }
        }
        finally {
            closeConnection(call.connection);
        }
        return done;
    }

    public void run() {
        long lastPurgeTime = 0;   // last check for old calls.

        while (running) {
            try {
                waitPending();     // If a channel is being registered, wait.
                writeSelector.select(PURGE_INTERVAL);
                Iterator<SelectionKey> iter = writeSelector.selectedKeys().iterator();
                while (iter.hasNext()) {
                    SelectionKey key = iter.next();
                    iter.remove();
                    try {
                        if (key.isValid() && key.isWritable()) {
                            doAsyncWrite(key);
                        }
                    } catch (IOException e) {
                        LOG.info(getName() + ": doAsyncWrite threw exception " + e);
                    }
                }

                // omitted
            } catch (OutOfMemoryError e) {
                // omitted
            } catch (Exception e) {
                // omitted
            }
        }
    }

    private void doAsyncWrite(SelectionKey key) throws IOException {
        Call call = (Call)key.attachment();
        if (call == null) {
            return;
        }
        if (key.channel() != call.connection.channel) {
            throw new IOException("doAsyncWrite: bad channel");
        }

        synchronized(call.connection.responseQueue) {
            if (processResponse(call.connection.responseQueue, false)) { // 调用processResponse的第二种情况，在Responder线程中调用
                try {
                    key.interestOps(0);
                } catch (CancelledKeyException e) {
                    // omitted
                }
            }
        }
    }
}
```
下面就把上面的代码综合起来讲讲。首先在Handler线程中会调用`doRespond()`，这个方法就把call放入responseQueue中，如果队列中就这一个call就亲自调用`processResponse()`，也就是说从Handler线程中将返回结果发回给客户端，否则的话就等着到时候由Responder线程来处理。Responder线程通过writeSelector来管理和客户端的连接，不像readSelector关注的是read ready，writeSelector自然关注的是write ready，一旦某个客户端的连接可写后，Responder就找出这个连接对应的Connection对象，然后从Connection的responseQueue中取出call，将结果发送回客户端。可以看到将rpc结果返回给客户端可能发生在Handler线程，也有可能发生在Responder线程。

至此整个流程就讲完了，经过三大组件轮番处理后，总算功德圆满。老规矩，最后附图一张。

{% img /images/post/hdfs-source-code-analysis/hdfs_server_handle_client_request.png %}

参考链接：
<a href="http://blog.csdn.net/xhh198781/article/details/7280084">1</a>
<a href="http://blog.csdn.net/xhh198781/article/details/7268298">2</a>
<a href="http://tutorials.jenkov.com/java-nio/selectors.html">3</a>

## <a id="hdfs_datanode_storage_related_class"></a> DataNode中和数据相关的类
我们知道DataNode中存放了具体的数据，也就是说HDFS文件系统中的数据(也就是文件)是分散存储在很多个DataNode之中的。但有一点要注意的是DataNode是以Block为单位来存放数据的，一个Block对应于DataNode本地硬盘上的一个文件。而我们认为的HDFS文件系统中的一个文件会被切割成多个Block进行存放，这里就有概念上的文件和实际上的文件的关联。对于client来说HDFS为我们抽象出了一个文件系统，这是个分布式的文件系统，但我们可以像普通文件系统一样来看待它和使用它。我们可以认为HDFS中有个文件A.txt(概念上的文件)，我们可以直接对这个文件进行读写操作，但在底层实现上这个文件会被拆成多个Block(实际上的文件)分散存放在多个DataNode的本地文件系统中。NameNode负责概念上的文件和实际上的文件的转化，比如NameNode知道一个概念上的文件由多少Block组成并且这些Block都在哪些DataNode上，同时NameNode也会维护概念上的目录结构，比如说A.txt在/jjyao/data/路径下。与NameNode相对的就是DataNode，它不知道任何概念上的东西，它不知道什么是A.txt，什么是/jjyao/data，它只知道一个个的Block文件，这些文件都存放在本地的文件系统中。  
Block文件存放在本地的什么地方由配置项dfs.data.dir指定。比如我可以指定这些Block文件都存放在/datanode/data1和/datanode/data2这两个目录下(当然除了存放Block文件DataNode还会存放一些其他文件)。对于每个dfs.data.dir指定的根目录里面有current，detach等目录，其中current目录可以看成是存放Block文件的根目录，这个目录里面可以看成一颗树，树中各级目录下的文件就是Block文件和对应的meta文件。可能有人会问为什么不把所有的Block文件都直接放在current目录下呢，答案是如果Block文件很多的话，本地的文件系统可能不能很好的支持在一个目录下有太多的文件。现在我们大体知道DataNode存了什么数据(更详细的解释参加[这里](http://india.paxcel.net:6060/LargeDataMatters/wp-content/uploads/2010/09/HDFS1.pdf))，那么DataNode中哪些类和这些数据相关就是我们这节要讨论的问题。

在介绍这些类之前写给出一张直观上的图。在这里假设dfs.data.dir规定了/datanode/data1和/datanode/data2两个地方来存放DataNode中的数据，那么就有如下的一张图

{% img /images/post/hdfs-source-code-analysis/hdfs_datanode_storage_abstract.png %}

可以看到DataNode通过DataStorage和FSDataset这两个类来管理着存放的数据。从这幅图中可以清晰看出来DataStorage和StorageDirectory属于一个阵营而FSDataset，FSVolumnSet，FSVolume和FSDir属于另一个阵营。它们虽然都用来管理DataNode中存放的数据，但是两块的功能不同。对于DataStorage和StorageDirectory来说主要完成的是HDFS的升级/回滚/提交操作。而对于另外一个阵营里面的类则是真正管理Block文件的，比如添加Block文件。其实只要看DataStorage和FSDataset的接口就可以知道这两个阵营所能做的事。

## <a id="hdfs_datanode_start"></a> DataNode的启动
对于DataNode集群的启动我们可以通过脚本start-all.sh来完成，这个脚本的命令会通过ssh传到各个DataNode所在的机器上，然后Java虚拟机会以DataNode.main()为入口启动DataNode。因此我们的分析也就是从DataNode.main()开始一层层深入，下面看源代码。
``` java
/* DataNode.java */
public static void main(String args[]) {
    secureMain(args, null);
}

public static void secureMain(String [] args, SecureResources resources) {
    try {
        DataNode datanode = createDataNode(args, null, resources);
        if (datanode != null)
            datanode.join(); // 等待DataNode的Daemon进程结束
    } catch (Throwable e) {
        System.exit(-1);
    } finally {
        System.exit(0);
    }
}

public static DataNode createDataNode(String args[],
        Configuration conf, SecureResources resources) throws IOException {
    DataNode dn = instantiateDataNode(args, conf, resources);
    runDatanodeDaemon(dn); // 把DataNode当作一个Daemon进程启动，运行DataNode.run()方法
    return dn;
}

public static DataNode instantiateDataNode(String args[],
        Configuration conf, SecureResources resources) throws IOException {
    String[] dataDirs = conf.getStrings(DATA_DIR_KEY); // 获取配置项dfs.data.dir的值，也就是DataNode中存放数据的目录
    return makeInstance(dataDirs, conf, resources);
}

public static DataNode makeInstance(String[] dataDirs, Configuration conf, 
        SecureResources resources) throws IOException {
    LocalFileSystem localFS = FileSystem.getLocal(conf);
    ArrayList<File> dirs = new ArrayList<File>();
    for (String dir : dataDirs) {
        try {
            // 检查配置项dfs.data.dir中给出的路径是否valid，比如确实是个目录而不是文件
            DiskChecker.checkDir(localFS, new Path(dir), dataDirPermission); 
            dirs.add(new File(dir));
        } catch(IOException e) {
            // omitted
        }
    }
    if (dirs.size() > 0) // 至少要存在一个valid的目录，不然。。。
        return new DataNode(conf, dirs, resources);
    return null;
}

DataNode(final Configuration conf,
        final AbstractList<File> dataDirs, SecureResources resources) throws IOException {
    super(conf);
    try {
        startDataNode(conf, dataDirs, resources);
    } catch (IOException ie) {
        shutdown();
        throw ie;
    }   
}

// 启动的主要工作都在这里
void startDataNode(Configuration conf, 
        AbstractList<File> dataDirs, SecureResources resources) throws IOException {

    // use configured nameserver & interface to get local hostname
    if (conf.get("slave.host.name") != null) {
        machineName = conf.get("slave.host.name");   
    }

    InetSocketAddress nameNodeAddr = NameNode.getServiceAddress(conf, true);

    InetSocketAddress socAddr = DataNode.getStreamingAddr(conf);
    int tmpPort = socAddr.getPort();
    storage = new DataStorage();
    // construct registration
    this.dnRegistration = new DatanodeRegistration(machineName + ":" + tmpPort);

    // connect to name node
    this.namenode = (DatanodeProtocol) // 有没有很熟悉，和之前客户端向NameNode发送rpc请求的原理差不多
        RPC.waitForProxy(DatanodeProtocol.class,
                DatanodeProtocol.versionID,
                nameNodeAddr, 
                conf);
    // get version and id info from the name-node
    NamespaceInfo nsInfo = handshake(); // 获取NameNode的版本信息，用来确保DataNode和NameNode的版本是一样的
    StartupOption startOpt = getStartupOption(conf);

    // read storage info, lock data dirs and transition fs state if necessary
    storage.recoverTransitionRead(nsInfo, dataDirs, startOpt);
    // adjust
    this.dnRegistration.setStorageInfo(storage);
    // 创建一个FSDataset实例，创建完成后FSDataset内部就会有像上一节那样的图了
    this.data = new FSDataset(storage, conf); 

    // find free port or use privileged port provide
    ServerSocket ss;
    if(secureResources == null) {
        ss = (socketWriteTimeout > 0) ? 
            ServerSocketChannel.open().socket() : new ServerSocket();
        Server.bind(ss, socAddr, 0);
    } else {
        ss = resources.getStreamingSocket();
    }
    // adjust machine name with the actual port
    tmpPort = ss.getLocalPort();
    selfAddr = new InetSocketAddress(ss.getInetAddress().getHostAddress(),
            tmpPort);

    this.threadGroup = new ThreadGroup("dataXceiverServer");
    // 这个服务器是用来处理Client或者其他DataNode对于Block文件的处理请求，比如要求发送某个Block文件的内容
    // 这个服务器上使用的不是RPC机制而是一种流式机制，因为要传输Block文件的内容嘛
    this.dataXceiverServer = new Daemon(threadGroup,  
            new DataXceiverServer(ss, conf, this));

    DataNode.nameNodeAddr = nameNodeAddr;

    //initialize periodic block scanner
    // 单独的一个线程，定期对所有的Block文件进行扫描校验
    blockScanner = new DataBlockScanner(this, (FSDataset)data, conf);

    //create a servlet to serve full-file content
    // 开启一个web服务器，运行通过浏览器来获得DataNode的状态
    this.infoServer = (secureResources == null) 
        ? new HttpServer("datanode", infoHost, tmpInfoPort, tmpInfoPort == 0, 
                conf, SecurityUtil.getAdminAcls(conf, DFSConfigKeys.DFS_ADMIN))
        : new HttpServer("datanode", infoHost, tmpInfoPort, tmpInfoPort == 0,
                conf, SecurityUtil.getAdminAcls(conf, DFSConfigKeys.DFS_ADMIN),
                secureResources.getListener());

    this.infoServer.addServlet(null, "/blockScannerReport", 
            DataBlockScanner.Servlet.class);

    this.infoServer.start();

    // BlockTokenSecretManager is created here, but it shouldn't be
    // used until it is initialized in register().
    this.blockTokenSecretManager = new BlockTokenSecretManager(false,
            0, 0);
    //init ipc server
    InetSocketAddress ipcAddr = NetUtils.createSocketAddr(
            conf.get("dfs.datanode.ipc.address"));
    // 这个RPC服务器和NameNode上的RPC服务器是一样的实现
    ipcServer = RPC.getServer(this, ipcAddr.getHostName(), ipcAddr.getPort(), 
            conf.getInt("dfs.datanode.handler.count", 3), false, conf,
            blockTokenSecretManager);
}
```
总结一下DataNode初始化的时候大致干了些什么事。首先初始化了一个NameNode的Remote Proxy，这样以后和NameNode进行通信就方便啦。然后就把上一节介绍的两大阵营的类给初始化好，这样以后DataNode就可以通过FSDataset和DataStorage来操作DataNode中存储的所有数据了。接下来初始化了三个服务器和一个DataBlockScanner，三个服务器分别是流式机制的dataXceiverServer，web服务器infoServer和RPC服务器ipcServer。  
所有这些都初始化好过后，`createDataNode()`方法就会调用DataNode的`run()`方法，然后DataNode就正式跑起来了，下面看源代码。
``` java
/* DataNode.java */
public void run() {
    dataXceiverServer.start();
    ipcServer.start();

    while (shouldRun) {
        try {
            startDistributedUpgradeIfNeeded();
            offerService(); // 主要是这个函数
        } catch (Exception ex) {
            if (shouldRun) {
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException ie) {
                }
            }
        }
    }
    shutdown();
}

/**
 * Main loop for the DataNode.  Runs until shutdown,
 * forever calling remote NameNode functions.
 */
public void offerService() throws Exception {
    while (shouldRun) {
        try {
            long startTime = now();

            // 过一段时间执行一次
            if (startTime - lastHeartbeat > heartBeatInterval) {
                //
                // All heartbeat messages include following info:
                // -- Datanode name
                // -- data transfer port
                // -- Total capacity
                // -- Bytes remaining
                //
                lastHeartbeat = startTime;
                // 发送心跳包表明DataNode还活着，同时获得NameNode传给DataNode的指令
                // NameNode和DataNode遵循严格的C/S架构，NameNode不会主动去联系DataNode的
                // 只有在DataNode联系它的时候才借机传输点命令给DataNode
                DatanodeCommand[] cmds = namenode.sendHeartbeat(dnRegistration,
                        data.getCapacity(),
                        data.getDfsUsed(),
                        data.getRemaining(),
                        xmitsInProgress.get(),
                        getXceiverCount());
                if (!processCommand(cmds))
                    continue;
            }

            // check if there are newly received blocks
            Block [] blockArray=null;
            String [] delHintArray=null;
            synchronized(receivedBlockList) {
                synchronized(delHints) {
                    int numBlocks = receivedBlockList.size();
                    if (numBlocks > 0) {
                        //
                        // Send newly-received blockids to namenode
                        //
                        blockArray = receivedBlockList.toArray(new Block[numBlocks]);
                        delHintArray = delHints.toArray(new String[numBlocks]);
                    }
                }
            }
            if (blockArray != null) {
                // 告诉NameNode最新接收到的Block文件
                namenode.blockReceived(dnRegistration, blockArray, delHintArray);
                synchronized (receivedBlockList) {
                    synchronized (delHints) {
                        for(int i=0; i<blockArray.length; i++) {
                            receivedBlockList.remove(blockArray[i]);
                            delHints.remove(delHintArray[i]);
                        }
                    }
                }
            }

            // Send latest blockinfo report if timer has expired.
            if (startTime - lastBlockReport > blockReportInterval) {
                if (data.isAsyncBlockReportReady()) {
                    // Create block report
                    long brCreateStartTime = now();
                    Block[] bReport = data.retrieveAsyncBlockReport();

                    // Send block report
                    long brSendStartTime = now();
                    // 报告Block文件的信息
                    DatanodeCommand cmd = namenode.blockReport(dnRegistration,
                            BlockListAsLongs.convertToArrayLongs(bReport));

                    // omitted
                    processCommand(cmd);
                } else {
                    data.requestAsyncBlockReport();
                    if (lastBlockReport > 0) { // this isn't the first report
                        // omitted
                    }
                }
            }

            // start block scanner
            // 只会执行一次
            if (blockScanner != null && blockScannerThread == null &&
                    upgradeManager.isUpgradeCompleted()) {
                blockScannerThread = new Daemon(blockScanner);
                blockScannerThread.start();
            }

            //
            // There is no work to do;  sleep until hearbeat timer elapses, 
            // or work arrives, and then iterate again.
            //
            long waitTime = heartBeatInterval - (System.currentTimeMillis() - lastHeartbeat);
            synchronized(receivedBlockList) {
                if (waitTime > 0 && receivedBlockList.size() == 0) {
                    try {
                        receivedBlockList.wait(waitTime);
                    } catch (InterruptedException ie) {
                    }
                    delayBeforeBlockReceived();
                }
            } // synchronized
        } catch(RemoteException re) {
            // omitted
        } catch (IOException e) {
        }
    } // while (shouldRun)
} // offerService
```
可以看到DataNode把那几个服务器和DataBlockScanner作为独立的线程启动后，自己也进入了一个while循环执行以下的操作：定期发送心跳包和Block信息报告，还发送最新接收到的Block文件信息。至此对于DataNode启动的分析就结束了，在这一过程中遇到的所有细节都没有深究，比如DataBlockScanner的原理是什么，这些都留到以后再介绍。

参考链接：
<a href="http://caibinbupt.iteye.com/blog/287870">1</a>

## <a id="reference"></a> 参考资料
在研究源码的过程中也找到了一些比较好的参考资料，仅供参考

1. [xhh198781的博客](http://blog.csdn.net/xhh198781)
2. [caibinbupt的博客](http://caibinbupt.iteye.com)
3. [一份pdf文件](http://india.paxcel.net:6060/LargeDataMatters/wp-content/uploads/2010/09/HDFS1.pdf)
