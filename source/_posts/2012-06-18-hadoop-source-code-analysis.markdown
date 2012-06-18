---
layout: post
title: "Hadoop Source Code Analysis"
date: 2012-06-18 19:28
comments: true
categories: 
published: false
---

1. [创建FileSystem](#instantiate_FileSystem)

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
