---
title: java类加载器
date: 2017-09-20 16:01:43
categories: "java基础"
tags:
---
#### 1.java 默认提供的三个类加载器
1.BootStrap ClassLoader：称为启动类加载器，是Java类加载层次中最顶层的类加载器，
负责加载JDK中的核心类库，如：rt.jar、resources.jar、charsets.jar等，
可通过如下程序获得该类加载器从哪些地方加载了相关的jar或class文件：

2.Extension ClassLoader：称为扩展类加载器，负责加载Java的扩展类库，默认加载
JAVA_HOME/jre/lib/ext/目下的所有jar。

3.App ClassLoader：称为系统类加载器，负责加载应用程序classpath目录下的所有jar
和class文件。
``` bash
    URL[] urls = sun.misc.Launcher.getBootstrapClassPath().getURLs();
    for(int i = 0; i < urls.length; i++) {
        System.out.println(urls[i].toExternalForm());
    }
    System.out.println(System.getProperty("sun.boot.class.path"));

    ClassLoader loader = ClassLoaderTest.class.getClassLoader();//获得加载ClassLoaderTest.class这个类的类加载器
    while(loader != null) {
        System.out.println(loader);
        loader = loader.getParent();//获得父类加载器的引用
    }
    System.out.println(loader);
```

注意： 除了Java默认提供的三个ClassLoader之外，用户还可以根据需要定义自已的ClassLoader，
而这些自定义的ClassLoader都必须继承自java.lang.ClassLoader类，也包括Java提供的另外二个
ClassLoader（Extension ClassLoader和App ClassLoader）在内，但是Bootstrap 
ClassLoader不继承自ClassLoader，因为它不是一个普通的Java类，底层由C++编写，
已嵌入到了JVM内核当中，当JVM启动后，Bootstrap ClassLoader也随着启动，
负责加载完核心类库后，并构造Extension ClassLoader和App ClassLoader类加载器。

#### 2.ClassLoader加载类的原理

1、原理介绍

ClassLoader使用的是双亲委托模型来搜索类的，每个ClassLoader实例都有一个父类加载器的引用
（不是继承的关系，是一个包含的关系），虚拟机内置的类加载器（Bootstrap ClassLoader）本
身没有父类加载器，但可以用作其它ClassLoader实例的的父类加载器。当一个ClassLoader实例需
要加载某个类时，它会试图亲自搜索某个类之前，先把这个任务委托给它的父类加载器，这个过程是
由上至下依次检查的，首先由最顶层的类加载器Bootstrap ClassLoader试图加载，如果没加载到，
则把任务转交给Extension ClassLoader试图加载，如果也没加载到，则转交给App ClassLoader
 进行加载，如果它也没有加载得到的话，则返回给委托的发起者，由它到指定的文件系统或网络等URL
 中加载该类。如果它们都没有加载到这个类时，则抛出ClassNotFoundException异常。否则将这个
 找到的类生成一个类的定义，并将它加载到内存当中，最后返回这个类在内存中的Class实例对象。

2、为什么要使用双亲委托这种模型呢？

因为这样可以避免重复加载，当父亲已经加载了该类的时候，就没有必要子ClassLoader再加载一次。
考虑到安全因素，我们试想一下，如果不使用这种委托模式，那我们就可以随时使用自定义的String来
动态替代java核心api中定义的类型，这样会存在非常大的安全隐患，而双亲委托的方式，就可以避免
这种情况，因为String已经在启动时就被引导类加载器（Bootstrcp ClassLoader）加载，所以用
户自定义的ClassLoader永远也无法加载一个自己写的String，除非你改变JDK中ClassLoader搜索
类的默认算法。

 3、 但是JVM在搜索类的时候，又是如何判定两个class是相同的呢？

JVM在判定两个class是否相同时，不仅要判断两个类名是否相同，而且要判断是否由同一个类加
载器实例加载的。只有两者同时满足的情况下，JVM才认为这两个class是相同的。就算两个class
是同一份class字节码，如果被两个不同的ClassLoader实例所加载，JVM也会认为它们是两个不同
class。比如网络上的一个Java类org.classloader.simple.NetClassLoaderSimple，javac编
译之后生成字节码文件NetClassLoaderSimple.class，ClassLoaderA和ClassLoaderB这两个类
加载器并读取了NetClassLoaderSimple.class文件，并分别定义出了java.lang.Class实例来表
示这个类，对于JVM来说，它们是两个不同的实例对象，但它们确实是同一份字节码文件，如果试图
将这个Class实例生成具体的对象进行转换时，就会抛运行时异常java.lang.ClassCaseException，
提示这是两个不同的类型。现在通过实例来验证上述所描述的是否正确：

#### 3.定义自己的类加载器

既然JVM已经提供了默认的类加载器，为什么还要定义自已的类加载器呢？

因为Java中提供的默认ClassLoader，只加载指定目录下的jar和class，如果我们想加载其它位置
的类或jar时，比如：我要加载网络上的一个class文件，通过动态加载到内存之后，要调用这个类中
的方法实现我的业务逻辑。在这样的情况下，默认的ClassLoader就不能满足我们的需求了，所以需
要定义自己的ClassLoader。

定义自已的类加载器分为两步：

1、继承java.lang.ClassLoader

2、重写父类的findClass方法

读者可能在这里有疑问，父类有那么多方法，为什么偏偏只重写findClass方法？

因为JDK已经在loadClass方法中帮我们实现了ClassLoader搜索类的算法，当在loadClass方法中搜
索不到类时，loadClass方法就会调用findClass方法来搜索类，所以我们只需重写该方法即可。如没
有特殊的要求，一般不建议重写loadClass搜索类的算法。下图是API中ClassLoader的loadClass
方法：

``` bash
    public class NetworkClassLoader extends ClassLoader {  
        private String rootUrl;  
       
        public NetworkClassLoader(String rootUrl) {  
            this.rootUrl = rootUrl;  
        }  
       
        @Override 
        protected Class<?> findClass(String name) throws ClassNotFoundException {  
            Class clazz = null;//this.findLoadedClass(name); // 父类已加载     
            //if (clazz == null) {  //检查该类是否已被加载过  
                byte[] classData = getClassData(name);  //根据类的二进制名称,获得该class文件的字节码数组  
                if (classData == null) {  
                    throw new ClassNotFoundException();  
                }  
                clazz = defineClass(name, classData, 0, classData.length);  //将class的字节码数组转换成Class类的实例  
            //}   
            return clazz;  
        }  
       
        private byte[] getClassData(String name) {  
            InputStream is = null;  
            try {  
                String path = classNameToPath(name);  
                URL url = new URL(path);  
                byte[] buff = new byte[1024*4];  
                int len = -1;  
                is = url.openStream();  
                ByteArrayOutputStream baos = new ByteArrayOutputStream();  
                while((len = is.read(buff)) != -1) {  
                    baos.write(buff,0,len);  
                }  
                return baos.toByteArray();  
            } catch (Exception e) {  
                e.printStackTrace();  
            } finally {  
                if (is != null) {  
                   try {  
                      is.close();  
                   } catch(IOException e) {  
                      e.printStackTrace();  
                   }  
                }  
            }  
            return null;  
        }  
       
        private String classNameToPath(String name) {  
            return rootUrl + "/" + name.replace(".", "/") + ".class";  
        }  
       
    }
    
       
public class ClassLoaderTest {  
   
    public static void main(String[] args) {  
        try {  
            /*ClassLoader loader = ClassLoaderTest.class.getClassLoader();  //获得ClassLoaderTest这个类的类加载器 
            while(loader != null) { 
                System.out.println(loader); 
                loader = loader.getParent();    //获得父加载器的引用 
            } 
            System.out.println(loader);*/ 
               
   
            String rootUrl = "http://localhost:8080/httpweb/classes";  
            NetworkClassLoader networkClassLoader = new NetworkClassLoader(rootUrl);  
            String classname = "org.classloader.simple.NetClassLoaderTest";  
            Class clazz = networkClassLoader.loadClass(classname);  
            System.out.println(clazz.getClassLoader());  
               
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
       
}
输出为：classloadr.NetworkClassLoader@19eeac
```

#### 4.容器的类加载器

目前常用web服务器中都定义了自己的类加载器，用于加载web应用指定目录下的类库（jar或class），
如：Weblogic、Jboss、tomcat等，
下面我以Tomcat为例，展示该web容器都定义了哪些个类加载器：
``` bash
    public JSONResponse getA() throws CapiInnerException {
        ClassLoader loader = this.getClass().getClassLoader();
        JSONResponse rs = new JSONResponse();
        String name = "";
        while(loader != null) {
            name = name + loader.getClass().getName()+ ";";
            loader = loader.getParent();
        }
        String bb = String.valueOf(loader);
        return rs.buildSuccess( name + bb);
    }
Tomcat
{
org.apache.catalina.loader.WebAppClassLoader
org.apache.catalina.loader.StandardClassLoader
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader;
null;
}
jetty
{
org.eclipse.jetty.webapp.WebAppClassLoader;
com.sankuai.mms.boot.Classpath$MmsBootClassLoader;
sun.misc.Launcher$ExtClassLoader;
null;
}
``` 
#### 5.什么时候需要类加载器

首先介绍自定义类的应用场景：
（1）加密：Java代码可以轻易的被反编译，如果你需要把自己的代码进行加密以防止反编译，可以先
将编译后的代码用某种加密算法加密，类加密后就不能再用Java的ClassLoader去加载类了，这时就
需要自定义ClassLoader在加载类的时候先解密类，然后再加载。
（2）从非标准的来源加载代码：如果你的字节码是放在数据库、甚至是在云端，就可以自定义类加载器，
从指定的来源加载类。
（3）以上两种情况在实际中的综合运用：比如你的应用需要通过网络来传输 Java 类的字节码，
为了安全性，这些字节码经过了加密处理。这个时候你就需要自定义类加载器来从某个网络地址上读取加
密后的字节代码，接着进行解密和验证，最后定义出在Java虚拟机中运行的类。

``` bash
//双亲委派模型的工作过程源码
protected synchronized Class<?> loadClass(String name, boolean resolve)
throws ClassNotFoundException
{
// First, check if the class has already been loaded
Class c = findLoadedClass(name);
if (c == null) {
try {
if (parent != null) {
c = parent.loadClass(name, false);
} else {
c = findBootstrapClassOrNull(name);
}
} catch (ClassNotFoundException e) {
// ClassNotFoundException thrown if class not found
// from the non-null parent class loader
//父类加载器无法完成类加载请求
}
if (c == null) {
// If still not found, then invoke findClass in order to find the class
//子加载器进行类加载 
c = findClass(name);
}
}
if (resolve) {//判断是否需要链接过程，参数传入
resolveClass(c);
}
return c;
}
```

双亲委派模型的工作过程如下：
（1）当前类加载器从自己已经加载的类中查询是否此类已经加载，如果已经加载则直接返回原来已经加载
的类。
（2）如果没有找到，就去委托父类加载器去加载（如代码c = parent.loadClass(name, false)所示）。父类加载器也会采用同样的策略，查看自己已经加载过的类中是否包含这个类，有就返回，没有就委托父类的父类去加载，一直到启动类加载器。因为如果父加载器为空了，就代表使用启动类加载器作为父加载器去加载。
（3）如果启动类加载器加载失败（例如在$JAVA_HOME/jre/lib里未查找到该class），会使用拓展
类加载器来尝试加载，继续失败则会使用AppClassLoader来加载，继续失败则会抛出一个异常
ClassNotFoundException，然后再调用当前加载器的findClass()方法进行加载。
 
双亲委派模型的好处：
（1）主要是为了安全性，避免用户自己编写的类动态替换 Java的一些核心类，比如 String。
（2）同时也避免了类的重复加载，因为 JVM中区分不同类，不仅仅是根据类名，相同的 class文件
被不同的 ClassLoader加载就是不同的两个类。