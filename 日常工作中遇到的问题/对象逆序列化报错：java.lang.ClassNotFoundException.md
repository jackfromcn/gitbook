# 对象逆序列化报错：java.lang.ClassNotFoundException

简单的想从保存的对象中又一次解析出对象。用了逆序列化，但是报错：

```bash
java.lang.ClassNotFoundException: xxxxxxxxxxxx
at java.net.URLClassLoader1.run(URLClassLoader.java:366)
at java.net.URLClassLoader1.run(URLClassLoader.java:355)
at java.security.AccessController.doPrivileged(Native Method)
at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
at java.lang.ClassLoader.loadClass(ClassLoader.java:423)
at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:308)
at java.lang.ClassLoader.loadClass(ClassLoader.java:356)
at java.lang.Class.forName0(Native Method)
at java.lang.Class.forName(Class.java:264)
at java.io.ObjectInputStream.resolveClass(ObjectInputStream.java:622)
at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:1593)
at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1514)
at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:1750)
at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1347)
at java.io.ObjectInputStream.readObject(ObjectInputStream.java:369)
at xxxxxxxxxxxxxxxxx(TestMetadata.java:103)
```

提示类找不到。但实际上类文件是确确实实存在的。那就上网搜，果然找到答案。

能够參考文章： <http://www.javapractices.com/topic/TopicAction.do?Id=45>

最主要的两点：

- 1) 须要相同的包名

- 2) 相同的序列化ID



总结：对象在序列化时，已经将对应的**包名**，**序列化ID**序列化到字节码文件中了，所以反序列化的时候，也需要之前序列化好的**包名**和**序列化ID**。



这里出现这个问题的原因是：<br/>在初始化项目框架的时候，写了一个包路径，项目中使用了 `shiro` 来控制权限，用户登录后，将对象序列化成字节码数组，缓存到了 `redis` 中。但是，后来，同事在原来的包路径的基础上增加了一层，所以，重新登录的用户态是没有问题的。但是，之前已经登录过的用户态，反序列化时，就找不到对应的 `class` 了。