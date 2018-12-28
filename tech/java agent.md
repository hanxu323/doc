## 什么是Java agent？
对于普通开发者来说，Java agent绝对算是一项黑科技。有人把他翻译为Java代理，但他的名字“agent”还有另外一层含义，就是“间谍、特工”。我更喜欢这个形象的定义，**他能够在运行时无声无息地“侵入”JVM上的应用，对代码进行修改，而应用程序却一无所知**。  
所以，Java agent非常强大，又极度危险，他能做很多事情，但如果不加小心，JVM也可能被他搞垮。  

Java agent如此神奇，他是如何进行“入侵”的呢？共有两种方式，其一是在JVM启动时，其二是在JVM运行时（runtime）。  

**先来讲在JVM启动时的“入侵”方式。**  
从外表来看，Java agent是一个衣冠楚楚的jar包，他的内部构造，通常包含若干编译好的class和一个MANIFEST.MF描述文件构成。和普通Java程序的`main`方法相似，agent也有一个入口类，其中必须包含一个入口方法，称之为`premain`。这个`premain`方法就是进入“入侵之路”的大门。

大家都知道JVM是可以添加启动参数的，像是`-Xss`、`-Xms`等都是耳熟能详的参数，当我们向启动参数中添加`-javaagent:your-agent.jar`，JVM启动时即会加载“your-agent.jar”这个agent程序，并且调用agent中的`premain`方法，在此可以对加载类的字节码修改等种种操作，当完成后，JVM才会去调用真正应用程序的`main`入口方法。

**自JDK6起，JVM支持了另一种 “入侵”方式--在运行时（runtime）进行动态修改。**  
`premain`还有一个孪生兄弟，称之为`agentmain`方法，他可以在运行时（runtime）动态修改类字节码，很多框架的热加载就是利用这个特性实现的。当我们实现`agentmain`方法后，需要借助Sun提供的一套扩展Attach API进行Instrument注入或解除，在这里不做过多介绍了，有兴趣的同学，可以自行学习。

## 什么是Instrumentation
JVM通过调用`premain`或`agentmain`方法，来执行agent，他究竟能做些什么呢？这里就要提到Instrumentation这个概念。  
>>>
Java Instrumentation指的是可以用独立于应用程序之外的代理（agent）程序来监测和协助运行在JVM上的应用程序。 这种监测和协助包括但不限于获取JVM运行时状态，替换和修改类定义等。  
>>>

`premain`有两种方法签名，分别为  
1、 `public static void premain(String agentArgs, Instrumentation inst)`  
2、 `public static void premain(String agentArgs)`  
方法1优先级比2高，会优先执行，通常我们为了利用强大的Instrumentation，都会选择实现第一种方法，JVM在启动时，会将Instrumentation实例传入到`premain`方法中。  
同样的，`agentmain`也有两种方法签名，在JVM启动后，适时调用，以完成运行时（runtime）动态字节码修改。  
1、  `public static void agentmain(String agentArgs, Instrumentation inst)`  
2、  `public static void agentmain(String agentArgs)`  

JAVA Instrumentation是非常强大的，我们来看几项最显著的能力。  
* 对类进行重定义（redefine）、重转换（retransform），等同于在类加载、或是运行时，对方法体、常量池、属性进行修改；
* 设置native方法的前缀prefix；
* 在运行时动态增加BootClassPath、SystemClassPath，或是在运行时将jar包加入BootClassPath、SystemClassPath。  

上文介绍了Java agent的基本概念和开发步骤，看起来很简单，还有一个值得注意的地方是agent jar包内部的manifest描述文件，和普通Java程序一样，放在META-INF目录中，命名为MANIFEST.MF，包括若干项对agent包的描述信息，详细说明如下。  
`Premain-Class`：顾名思义，标识含有premain方法的类，以便JVM可以找到agent入口方法；  
`Agent-Class`：如果使用运行时Instrument，在此标识含有agentmain方法的类；  
`Can-Redefine-Classes`：如需redefine class（在JVM加载main方法前，称为redefine），则需要写为true，默认false；  
`Can-Retransform-Classes`：如需retransform class（JVM运行时修改，称之为retransform），则需写为true，默认false；  
`Can-Set-Native-Method-Prefix`：如需修改native方法前缀，则需写为true，默认false；  
`Boot-Class-Path`：添加需要bootstrap类加载器搜索的路径。  

## Show me the code
至此，我们已经了解Java agent的核心概念与强大之处，来看看代码吧。  
编写Java agent是非常简单的，难点在于，如果要修改类，必须使用字节码操作，而JDK自身没有提供字节码操作工具，我们需要利用一些第三方工具库，比如ASM、Javassist等。以下代码使用Javassist进行字节码修改。  

先写一个测试类SimpleTest，包含入口main方法。  
```java
public class SimpleTest {
    public static void main(String[] args) {
        printHello();
        printWorld();
    }

    public static void printHello() {
        System.out.println("hello ");
    }

    public static void printWorld() {
        System.out.println("world");
    }
}
```
这个类是程序入口，调用两个方法打印出“hello world”。    

**下一步编写Java agent，对SimpleTest类注入一些代码，让他在执行每个方法前都打印出当前的方法名称。**

创建一个agent类，并实现上文所述的premain方法。利用Instrumentation添加了一个class transformer，来进行代码注入。
```java
public class Agent {
    public static void premain(String agentArgs, Instrumentation inst) {
        SimpleClassTransformer transformer = new SimpleClassTransformer();
        inst.addTransformer(transformer);
    }
}
```
这个agent类十分简单，代码注入操作都在SimpleClassTransformer中，我们再来看看具体实现。  
```java
public class SimpleClassTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(
            ClassLoader loader,
            String className,
            Class<?> classBeingRedefined,
            ProtectionDomain protectionDomain,
            byte[] classfileBuffer) throws IllegalClassFormatException {

        if (!className.endsWith("com/hx/agent/SimpleTest")) {
            return new byte[]{};
        }

        try {
            CtClass clazz = ClassPool.getDefault().get(className.replace("/", "."));
            CtMethod[] methods = clazz.getDeclaredMethods();

            for (CtMethod method : methods) {
                method.insertBefore("System.out.println(\"enter method '" + method.getName() + "'\");");
            }

            byte[] byteCode = clazz.toBytecode();
            clazz.detach();
            return byteCode;
        } catch (NotFoundException | CannotCompileException | IOException e) {
            e.printStackTrace();
        }

        return new byte[]{};
    }
}
```
这里有些复杂，使用了Javassist进行了字节码操作，其中`CtClass`、`ClassPool`、`CtMethod`均由Javassist提供。需要关注一点，这里的`className`并不是以“.”分割，而是“/”，从代码中可以看到，我们只过滤对SimpleTest类进行字节码修改，并对类中的每一个方法，利用Javassist在方法开始时插入了一行打印代码。  

最后打包并生成MANIFEST.MF，我使用了maven插件，为了测试方便，main程序和agent打在了一个jar包中，而在实际使用中，agent通常会独立打包。  

生成的MANIFEST.MF如下  
```
Manifest-Version: 1.0
Premain-Class: com.hx.agent.Agent
Archiver-Version: Plexus Archiver
Built-By: hanxu5
Created-By: Apache Maven 3.1.1
Build-Jdk: 1.8.0_73
Main-Class: com.hx.agent.SimpleTest
```

现在一切准备就绪，我们在控制台执行观察结果。  
```
java -jar jagent-1.0-SNAPSHOT.jar
```
控制台输出  
```
hello
world
```

增加`-javaagent`参数（:javaagent可以有多个，JVM会按顺序执行premain）  
```
java -javaagent:jagent-1.0-SNAPSHOT.jar -jar jagent-1.0-SNAPSHOT.jar
```
输出如下，在进入每个方法后都打印了当前方法名称  
```
enter method 'main'
enter method 'printHello'
hello
enter method 'printWorld'
world
```

## Java agent在pinpoint中的应用
在我接触的日常项目中，暂时未使用到agent，所以我选择了大家接触过APM工具Pinpoint，看看他是如何使用agent的。Pinpoint分为Collector、Storage、Agent、Web UI四部分，本文不详细讨论Pinpoint架构设计，来看看他的agent是如何实现的。  

Pinpoint agent作用是做字节码植入，插桩收集性能及调用链相关指标，再通过Collector模块进行收集，用于后续记录和查询呈现。  

以pinpoint-bootstrap-1.6.2.jar为例，在com.navercorp.pinpoint.bootstrap包内找到名为PinpointBootStrap的类，部分源码如下：
```java
public class PinpointBootStrap {
    // ...
    public static void premain(String agentArgs, Instrumentation instrumentation) {
        if(agentArgs == null) {
            agentArgs = "";
        }

        logger.info("pinpoint agentArgs:" + agentArgs);
        boolean success = STATE.start();
        if(!success) {
            logger.warn("pinpoint-bootstrap already started. skipping agent loading.");
        } else {
            Map agentArgsMap = argsToMap(agentArgs);
            AgentDirBaseClassPathResolver classPathResolver = new AgentDirBaseClassPathResolver();
            if(!classPathResolver.verify()) {
                logger.warn("Agent Directory Verify fail. skipping agent loading.");
                logPinpointAgentLoadFail();
            } else {
                BootstrapJarFile bootstrapJarFile = classPathResolver.getBootstrapJarFile();
                appendToBootstrapClassLoader(instrumentation, bootstrapJarFile);
                PinpointStarter bootStrap = new PinpointStarter(agentArgsMap, bootstrapJarFile, classPathResolver, instrumentation);
                if(!bootStrap.start()) {
                    logPinpointAgentLoadFail();
                }

            }
        }
    }
    
    // ......
}
```

可以看到java agent标准入口方法```premain(String agentArgs, Instrumentation instrumentation)```

PS：
在继续向下深入源码之前，先明确两个说明。  
1. 以下的各种类名包含Agent，均是Pinpoint定义的，与java agent无关，请大家不要混淆；  
2. Pinpoint定义了各种接口例如AgentOption、Agent、ApplicationContext，均有默认实现，分别为DefaultAgentOption、DefaultAgent、DefaultApplicationContext，为了下文简便，均以接口说明。  

premain其中的核心即是```PinpointStarter.start()```，继续深入可以看到如下代码片段。

```java
AgentOption option = this.createAgentOption(agentId, applicationName, e, this.instrumentation, pluginJars, this.bootstrapJarFile, serviceTypeRegistryService, annotationKeyRegistryService);
Agent pinpointAgent = agentClassLoader.boot(option);
pinpointAgent.start();
```

此处AgentOption是作为参数封装，其中包含了重要的instrument实例。  
继续深入至Agent的实例中，可以看到构造方法中通过agentOption创建了ApplicationContext实例。  
```java
public DefaultAgent(AgentOption agentOption, InterceptorRegistryBinder interceptorRegistryBinder) {
    // ...
    this.applicationContext = this.newApplicationContext(agentOption, interceptorRegistryBinder);
    // ...
}
```
最后，在ApplicationContext构造方法中，可以找到如下代码，利用instrument机制进行了字节码植入。  
```java
public DefaultApplicationContext(AgentOption agentOption, InterceptorRegistryBinder interceptorRegistryBinder) {
    // ...
    ClassFileTransformer classFileTransformer = this.wrap(this.classFileDispatcher);
    this.instrumentation.addTransformer(classFileTransformer, true);
    // ...
}
```
这里Pinpoint使用了装饰模式，加入了多个transformer，还封装了自己的AOP拦截器，有数个拦截实现类，用于完成各类指标收集及调用跟踪  
挑其一BasicMethodInterceptor一览究竟，有大家熟识的```before```和```after```  
```java
public void before(Object target, Object[] args) {
    if(this.isDebug) {
        this.logger.beforeInterceptor(target, args);
    }

    Trace trace = this.traceContext.currentTraceObject();
    if(trace != null) {
        SpanEventRecorder recorder = trace.traceBlockBegin();
        recorder.recordServiceType(this.serviceType);
    }
}

public void after(Object target, Object[] args, Object result, Throwable throwable) {
    if(this.isDebug) {
        this.logger.afterInterceptor(target, args);
    }

    Trace trace = this.traceContext.currentTraceObject();
    if(trace != null) {
        try {
            SpanEventRecorder recorder = trace.currentSpanEventRecorder();
            recorder.recordApi(this.descriptor);
            recorder.recordException(throwable);
        } finally {
            trace.traceBlockEnd();
        }

    }
}
```
至此，我们可以看到Pinpoint是如何使用java agent进行代码植入的，关于pinpoint的其他agent源码，是比较复杂的，也不在此文赘述了，有兴趣的同学可以找来源码继续深入研究。  

**One more thing...**  看看pinpoint agent的描述文件吧
```
Manifest-Version: 1.0
Premain-Class: com.navercorp.pinpoint.bootstrap.PinpointBootStrap
Archiver-Version: Plexus Archiver
Built-By: hubo9
Can-Redefine-Classes: true
Pinpoint-Version: 1.6.2
Can-Retransform-Classes: true
Created-By: Apache Maven 3.3.9
Build-Jdk: 1.8.0_77
```
我选用了胡博搭建的jtrace平台agent包，还可以看到他的构建痕迹 :smile:

## 结语
运行Java agent并使用Instrumentation改写代码非常容易，但由于是字节码操作，大家要非常小心，如果注入了不合适的代码，严重时可能会导致JVM终止。  

本文示例的agent，与AOP作用相似，但非常轻量，无需任何框架支持。但这并不意味Java agent只能做AOP，他更像是JVM提供的一个钩子，利用Instrumentation API使用JVM的一些高级特性，还可以在JVM启动时或运行时挂载，动态地去改变已经被JVM加载的类。  

在实际使用中，Instrumentation也通常被用来进行类的热加载、热部署，APM监控等，例如Pinpoint、Btrace、Greys等框架，也是利用Instrumentation结合Attach API进行埋点插桩来完成性能指标收集的。  

本文介绍Java agent仅是抛砖引玉、冰山一角，感兴趣的小伙伴，继续深入学习尝试吧。

## 参考资料
* [oracle official document](https://docs.oracle.com/javase/7/docs/api/java/lang/instrument/package-summary.html)  
* [Instrumentation新功能](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html)