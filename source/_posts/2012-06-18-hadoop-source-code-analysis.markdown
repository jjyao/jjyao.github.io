---
layout: post
title: "Hadoop Source Code Analysis"
date: 2012-06-18 19:28
comments: true
categories: 
published: false
---

1. [创建FileSystem](#instantiate_FileSystem)
2. [HDFS客户端和服务器之间的rpc](#hdfs_client_server_rpc)

<!-- more -->

## <a id="instantiate_FileSystem"></a>创建FileSystem
FileSystem在Hadoop中只是一个抽象类，它屏蔽了各种底层文件系统的实现，这有点Linux VFS的味道。在Hadoop中各个文件系统的实现都是继承了FileSystem这个抽象类的，因此用户只需要通过FileSystem的接口来操作底层的文件系统就可以了。Hadoop中具体的文件系统实现有很多，如DistributedFileSystem和LocalFileSystem。如何告知Hadoop要使用哪一个具体的文件系统实现是这节要讨论的。

在这里我们可以通过在Hadoop的配置文件里指定fs.default.name来实现，当然也可以在程序中显示指定。假设我们指定fs.default.name的值为hdfs://localhost:9000，其中的*hdfs*就是告诉Hadoop要使用DistributedFileSystem。Hadoop会从fs.default.name中抽出hdfs，然后去配置文件里找fs.*hdfs*.impl，这个可以在core-default.xml中找到，值为org.apache.hadoop.hdfs.DistributedFileSystem，可以看出Hadoop已经成功找到具体要创建的文件系统实现是什么了，接下来Hadoop通过反射就能由类名创建出类实例来，然后返回给用户使用。类似的如果我们指定fs.default.name的值为file:///，那么Hadoop就会去配置文件里找fs.*file*.impl，这个的值就是org.apache.hadoop.fs.LocalFileSystem。当然我们还能给fs.default.name指定很多其他的值来告诉Hadoop要使用的底层文件系统实现是什么。在这里还要说明的是fs.default.name的默认值就是file:///。接下来给出一些关键代码：
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

## <a id="hdfs_client_server_rpc"></a>HDFS客户端和服务器之间的rpc
对于HDFS来说我们是通过DistributedFileSystem来进行文件系统的操作的，在这里DistributedFileSystem就是client，而server就跑在NameNode上面。我们通过DistributedFileSystem发出的所有操作都会通过rpc传递到NameNode的server上，然后再执行最后将结果返回。这节就是讨论client和server之间的rpc。

首先谈论client如何将用户对DistributedFileSystem的方法调用传递到server端。在这里我们以mkdirs()方法为例，假设用户调用了DistributedFileSystem的mkdirs()方法，代码如下：
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
可以看到DFSClient又将方法调用转交给了实例变量namenode，于是我们就要了解namenode和与其有关的rpcNamenode这两个实例变量。对于DFSClient的构造函数，这里只显示最重要的两样东西分别是namenode和rpcNamenode。要了解这两样东西首先需要了解Java的[Dynamic Proxy](http://docs.oracle.com/javase/1.4.2/docs/guide/reflection/proxy.html)。我们可以看到rpcNamenode实现了ClientProtocol接口，这个接口就规定了client能向server调用的所有方法。按照Dynamic Proxy的机制我们知道所有对rpcNamenode的方法调用全部归结到了Invoker的invoke()方法上了，这个方法就是将client的调用信息如调用的方法，参数等通过socket传递给server，并获得返回结果。对于Invoker的invoke()方法的剖析这里就不在描述了，可以通过参考链接了解更多。至此client的方法调用就顺利传递到server了。照这样看DFSClient就只要调用rpcNamenode的mkdirs()方法就好了，为什么还要namenode呢？要了解这个就需要RetryProxy的代码了：
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
可以看到对于namenode的方法调用都归结到RetryInvocationHandler的invoke()上了，而这个方法就通过反射再调用rpcNamenode的对应方法，只不过它会retry，也就是说如果调用失败的话，它还会重新尝试调用，这么一来可靠性就增强了，毕竟网络传输很容易出问题的。如果让DFSClient直接调用rpcNamenode的话，一旦失败就会立刻告诉用户，而其实如果多尝试几次还是可能会成功的，这就是为什么要引入namenode的原因了。至此我们就知道client是如何将用户对DistributedFileSystem的方法调用传递到server端的，最后用一张图来总结。

{% img /images/post/hadoop-source-code-analysis/hdfs_client_server_rpc.png %}

参考链接：
<a href="http://caibinbupt.iteye.com/blog/280790">1</a>
<a href="http://caibinbupt.iteye.com/blog/281281">2</a>
<a href="http://caibinbupt.iteye.com/blog/281476">3</a>

