### 我是谁以及我要做的事情？

- 4年+Android移动端行业一线开发
- 经历过两款日活百万App开发和日常性能维护工作
- 有Akulaku,中移在线工作方式和思维习惯，在公司喜欢和同事交流分享技术，热爱分享、热爱Google技术、热爱生活



未来想做的事情：

- 把自己在职场中遇到的问题、处理事情的方式、学习Android的方法通过文字的方式分享给大家
- 如果我还能写代码，希望可以做到50岁。




### APM简要介绍

> APM 全称 Application Performance Management & Monitoring (应用性能管理/监控)

性能问题是导致 App 用户流失的罪魁祸首之一，如果用户在使用我们 App 的时候遇到诸如页面卡顿、响应速度慢、发热严重、流量电量消耗大等问题的时候，很可能就会卸载掉我们的 App。这也是我们在目前工作中面临的巨大挑战之一，尤其是低端机型。

商业化的APM平台：著名的 NewRelic，还有国内的听云、OneAPM 、阿里百川-码力APM的SDK、百度的APM收费产品等等。

**APM工作方式：**

1. 首先在客户端（Android、iOS、Web等）采集数据；
2. 接着将采集到的数据整理上报到服务器（多种方式 json、xml，上传策略等等）；
3. 服务器接收到数据后建模、存储、挖掘分析，让后将数据进行可视化展示（spark+flink），供用户使用。

![image-20190914180724597](https://user-gold-cdn.xitu.io/2020/5/19/1722d15ef73124df?w=2570&h=832&f=jpeg&s=216694)

（图片来自 百度APM产品介绍https://cloud.baidu.com/product/apm.html）

那么移动端需要做的事情就是：

- 双端统一原则（技术选型 NDK c++）
- 数据采集 （采集指标、细化等等）
- 数据存储（写文件？mmap、fileIO流）
- 数据上报（上报策略、上报方式）

那我们到底应该怎么做？一定要学会看开源的东西。让我们先看看大厂的开源怎么做的？我们在自己造轮子，完成自己的APM采集框架。

**目前核心开源APM框架产品**

- 腾讯 https://github.com/Tencent/matrix
- 360 https://github.com/Qihoo360/ArgusAPM
- 滴滴 https://github.com/didi/booster

你会发现自定义Gradle插件技术、ASM技术、打包流程Hook、Android打包流程等。那思考一下，为什么大家做的主要的流程都是一样的，不一样的是具体的实现细节，比如如何插桩采集到页面帧率、流量、耗电量、GC log等等。

 [ArgusAPM性能监控平台介绍&SDK开源-卜云涛.pdf](ArgusAPM性能监控平台介绍&SDK开源-卜云涛.pdf) 

我们先简单来看下在matrix中，如何利用Java Hook和 Native Hook完成IO 磁盘性能的监控？

![image-20190914182546349](https://user-gold-cdn.xitu.io/2020/5/19/1722d15f22716eb3?w=2124&h=992&f=jpeg&s=93113)

Java Hook的hook点是系统类`CloseGuard`，hook的方式是使用动态代理。

https://github.com/Tencent/matrix/blob/b83c481938b21c0080540d0c2babb04caa5e72c9/matrix/matrix-android/matrix-io-canary/src/main/java/com/tencent/matrix/iocanary/detect/CloseGuardHooker.java#L74

~~~java
private boolean tryHook() {
        try {
            Class<?> closeGuardCls = Class.forName("dalvik.system.CloseGuard");
            Class<?> closeGuardReporterCls = Class.forName("dalvik.system.CloseGuard$Reporter");
            Method methodGetReporter = closeGuardCls.getDeclaredMethod("getReporter");
            Method methodSetReporter = closeGuardCls.getDeclaredMethod("setReporter", closeGuardReporterCls);
            Method methodSetEnabled = closeGuardCls.getDeclaredMethod("setEnabled", boolean.class);

            sOriginalReporter = methodGetReporter.invoke(null);

            methodSetEnabled.invoke(null, true);

            // open matrix close guard also
            MatrixCloseGuard.setEnabled(true);

            ClassLoader classLoader = closeGuardReporterCls.getClassLoader();
            if (classLoader == null) {
                return false;
            }

            methodSetReporter.invoke(null, Proxy.newProxyInstance(classLoader,
                new Class<?>[]{closeGuardReporterCls},
                new IOCloseLeakDetector(issueListener, sOriginalReporter)));

            return true;
        } catch (Throwable e) {
            MatrixLog.e(TAG, "tryHook exp=%s", e);
        }

        return false;
    }
~~~

这里的CloseGuard有啥用？为什么腾讯的人要hook这个。这个在后续的分线中我们在来详细的说。如果要解决这个疑问，做好的办法就是看源码。（==系统埋点方式，监控系统资源的异常回收==）

**关于native hook：**

Native Hook是采用PLT(GOT) Hook的方式hook了系统so中的IO相关的`open`、`read`、`write`、`close`方法。在代理了这些系统方法后，Matrix做了一些逻辑上的细分，从而检测出不同的IO Issue。

https://github.com/Tencent/matrix/blob/b83c481938b21c0080540d0c2babb04caa5e72c9/matrix/matrix-android/matrix-io-canary/src/main/cpp/io_canary_jni.cc#L290

~~~c++
 JNIEXPORT jboolean JNICALL
        Java_com_tencent_matrix_iocanary_core_IOCanaryJniBridge_doHook(JNIEnv *env, jclass type) {
            __android_log_print(ANDROID_LOG_INFO, kTag, "doHook");

            for (int i = 0; i < TARGET_MODULE_COUNT; ++i) {
                const char* so_name = TARGET_MODULES[i];
                __android_log_print(ANDROID_LOG_INFO, kTag, "try to hook function in %s.", so_name);
								//打开so文件，并在内存中映射成ELF文件格式
                loaded_soinfo* soinfo = elfhook_open(so_name);
                if (!soinfo) {
                    __android_log_print(ANDROID_LOG_WARN, kTag, "Failure to open %s, try next.", so_name);
                    continue;
                }
								//替换open函数
                elfhook_replace(soinfo, "open", (void*)ProxyOpen, (void**)&original_open);
                elfhook_replace(soinfo, "open64", (void*)ProxyOpen64, (void**)&original_open64);

                bool is_libjavacore = (strstr(so_name, "libjavacore.so") != nullptr);
                if (is_libjavacore) {
                    if (!elfhook_replace(soinfo, "read", (void*)ProxyRead, (void**)&original_read)) {
                        __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook read failed, try __read_chk");
                        //http://refspecs.linux-foundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/libc---read-chk-1.html 类似于read()
                        if (!elfhook_replace(soinfo, "__read_chk", (void*)ProxyRead, (void**)&original_read)) {
                            __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook failed: __read_chk");
                            elfhook_close(soinfo);
                            return false;
                        }
                    }

                    if (!elfhook_replace(soinfo, "write", (void*)ProxyWrite, (void**)&original_write)) {
                        __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook write failed, try __write_chk");
                        if (!elfhook_replace(soinfo, "__write_chk", (void*)ProxyWrite, (void**)&original_write)) {
                            __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook failed: __write_chk");
                            elfhook_close(soinfo);
                            return false;
                        }
                    }
                }

                elfhook_replace(soinfo, "close", (void*)ProxyClose, (void**)&original_close);

                elfhook_close(soinfo);
            }

            return true;
        }
~~~

**关于transform api**：

http://google.github.io/android-gradle-dsl/javadoc/2.1/com/android/build/api/transform/Transform.html

我们编译Android项目时，如果我们想拿到编译时产生的Class文件，并在生成Dex之前做一些处理，我们可以通过编写一个`Transform`来接收这些输入(编译产生的Class文件),并向已经产生的输入中添加一些东西。

![image-20190914185405607](https://user-gold-cdn.xitu.io/2020/5/19/1722d1648236ffe7?w=1332&h=876&f=jpeg&s=49256)

如何使用的？

https://github.com/Tencent/matrix/blob/master/matrix/matrix-android/matrix-gradle-plugin/src/main/java/com/tencent/matrix/trace/transform/MatrixTraceTransform.java

- 编写一个自定义的`Transform`
- 注册一个Plugin完成或者在gradle文件中直接注册。

```java
//MyCustomPlgin.groovy
public class MyCustomPlgin implements Plugin<Project> {

    @Override
    public void apply(Project project) {
        project.getExtensions().findByType(BaseExtension.class)
                .registerTransform(new MyCustomTransform());
    }
}
```

```java
project.extensions.findByType(BaseExtension.class).registerTransform(new MyCustomTransform());  //在build.gradle中直接写
```

MatrixTraceTransform 利用编译期字节码插桩技术，优化了移动端的FPS、卡顿、启动的检测手段。在打包过程中，hook生成Dex的Task任务，添加方法插桩的逻辑。我们的hook点是在Proguard之后，Class已经被混淆了，所以需要考虑类混淆的问题。

`MatrixTraceTransform`主要逻辑在`transform`方法中：

~~~java
@Override
    public void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        long start = System.currentTimeMillis()
        //是否增量编译
        final boolean isIncremental = transformInvocation.isIncremental() && this.isIncremental()
        //transform的结果，重定向输出到这个目录
        final File rootOutput = new File(project.matrix.output, "classes/${getName()}/")
        if (!rootOutput.exists()) {
            rootOutput.mkdirs()
        }
        final TraceBuildConfig traceConfig = initConfig()
        Log.i("Matrix." + getName(), "[transform] isIncremental:%s rootOutput:%s", isIncremental, rootOutput.getAbsolutePath())
        //获取Class混淆的mapping信息，存储到mappingCollector中
        final MappingCollector mappingCollector = new MappingCollector()
        File mappingFile = new File(traceConfig.getMappingPath());
        if (mappingFile.exists() && mappingFile.isFile()) {
            MappingReader mappingReader = new MappingReader(mappingFile);
            mappingReader.read(mappingCollector)
        }

        Map<File, File> jarInputMap = new HashMap<>()
        Map<File, File> scrInputMap = new HashMap<>()

        transformInvocation.inputs.each { TransformInput input ->
            input.directoryInputs.each { DirectoryInput dirInput ->
                //收集、重定向目录中的class
                collectAndIdentifyDir(scrInputMap, dirInput, rootOutput, isIncremental)
            }
            input.jarInputs.each { JarInput jarInput ->
                if (jarInput.getStatus() != Status.REMOVED) {
                    //收集、重定向jar包中的class
                    collectAndIdentifyJar(jarInputMap, scrInputMap, jarInput, rootOutput, isIncremental)
                }
            }
        }
        //收集需要插桩的方法信息，每个插桩信息封装成TraceMethod对象
        MethodCollector methodCollector = new MethodCollector(traceConfig, mappingCollector)
        HashMap<String, TraceMethod> collectedMethodMap = methodCollector.collect(scrInputMap.keySet().toList(), jarInputMap.keySet().toList())
       //执行插桩逻辑，在需要插桩方法的入口、出口添加MethodBeat的i/o逻辑
        MethodTracer methodTracer = new MethodTracer(traceConfig, collectedMethodMap, methodCollector.getCollectedClassExtendMap())
        methodTracer.trace(scrInputMap, jarInputMap)
        //执行原transform的逻辑；默认transformClassesWithDexBuilderForDebug这个task会将Class转换成Dex
        origTransform.transform(transformInvocation)
        Log.i("Matrix." + getName(), "[transform] cost time: %dms", System.currentTimeMillis() - start)
    }
~~~

 看到这里了，我们是不是应该总结下APM的核心技术是什么？

大厂面试之一：APM的核心技术是什么？做过自研的APM吗？

APM核心原理一句话总结：$\textcolor{Red}{依据打包原理，在 class 转换为 dex 的过程中，调用 gradle transform api 遍历 class 文件，借助 Javassist、ASM 等框架修改字节码，插入我们自己的代码实现性能数据的统计。这个过程是在编译器见完成} $

你掌握了APM的核心原理，也可以做Android的无痕埋点了，本质是一样的，不一样的是Hook的地方不一样。

**目前大厂面临的问题？**

- 人手不够，每个员工都有招人的KPI（3-6年最吃香，<32岁，能吃苦耐劳的）
- APP需要一套系统，包括阿里的**系统，头条的复制机器，都需要实现快速的复制、快速的中间件、快速的监控体系化，这是未来的大厂工程化技术趋势。那么APM该怎么做到快速的复制呢？
- 目前高手和Android发烧友不好招，大厂缺人才，严重缺。 

你自己的定位？有机会在谈一谈我自己的心得。

###APM监控维度和指标

App基础性能指标集中为8类：网络性能、崩溃、启动加载、内存、图片、页面渲染、IM和VoIP（业务相关性和你的APP相关）、用户行为监控，基础维度包括App、系统平台、App版本和时间维度。

> 网络性能

网络服务成功率，平均耗时和访问量、上下行的速率监控。访问的链接、耗时等等。思考怎么结合okhttp来做？

> 崩溃

崩溃数据的采集和分析，类似于Bugly平台的功能

> 启动加载

App的启动我们做了大的力气进行了优化，多线程等等， Spark的有向无环图（DAG）来处理业务的依赖性。对App的冷启动时长、Android安装后首次启动时长和Android Bundle（atlas框架）启动加载时长进行监控。

> 内存

四大监测目标：内存峰值、内存均值、内存抖动、内存泄露。

> IM和VoIP等业务指标

这两项都属于业务型技术指标的监控，例如对各类IM消息到达率和VoIP通话的成功率、平均耗时和请求量进行监控。这里需要根据自己的APP的业务进行针对性的梳理。

> 用户行为监控

用于App统计用户行为，实际上就是监控所有事件并把事件发送到服务上去。这在以前是埋点做的事情，现在也规整成APM需要做的事情，比如用户的访问路径，类似于PC时代的PV，UV等概念。

> 图片

资源文件的监测，比如Bitmap冗余处理。haha库处理，索引值。

> 页面渲染

界面流畅性监测、FPS的监测、慢函数监测、卡顿监测、文件IO开销监测等导致页面渲染的各种问题。

###微信Matrix框架解析

#### 整体分析

~~~java
  Matrix.Builder builder = new Matrix.Builder(application); // build matrix
  builder.patchListener(new TestPluginListener(this)); // add general pluginListener
  DynamicConfigImplDemo dynamicConfig = new DynamicConfigImplDemo(); // dynamic config
  
  // init plugin 
  IOCanaryPlugin ioCanaryPlugin = new IOCanaryPlugin(new IOConfig.Builder()
                    .dynamicConfig(dynamicConfig)
                    .build());
  //add to matrix               
  builder.plugin(ioCanaryPlugin);
  
  //init matrix
  Matrix.init(builder.build());

  // start plugin 
  ioCanaryPlugin.start();
~~~

整理的结构如下：

![image-20190914200458489](https://user-gold-cdn.xitu.io/2020/5/19/1722d171ff03ef79?w=1625&h=1080&f=jpeg&s=219235)

核心功能：

~~~html
Resource Canary:
   Activity 泄漏
   Bitmap 冗余
Trace Canary
   界面流畅性
   启动耗时
   页面切换耗时
   慢函数
   卡顿
SQLite Lint: 按官方最佳实践自动化检测 SQLite 语句的使用质量
IO Canary: 检测文件 IO 问题
   文件 IO 监控
   Closeable Leak 监控
~~~

整体架构分析：

**matrix-android-lib**

- Plugin 被定义为某种监控能力的抽象
- Issue 发生的某种被监控事件
- Report 一种观察者模式的实现，用于发现Issue时，通知观察者
- Matrix 单例模式的实现，暴漏给外部的接口
- matrix-config.xml 相关监控的配置项
- IDynamicConfig 开放给用户自定义的相关监控的配置项，其实例会被各个Plugin持

**plugin核心接口**

- IPlugin 一种监控能力的抽象
- PluginListener 开放给用户的，感知监控模块初始化、启动、停止、发现问题等生命周期的能力。 用户可自行实现，并注入到Matrix中
- Plugin
  1、对IPlugin的默认实现，以触发相关PluginListener生命周期
  2、实现 IssuePublisher.OnIssueDetectListener接口的onDetectIssue方法
  - 该方法会在 具体监控事件发生时，被 Matrix.with().getPluginByClass(xxx).onDetectIssue(Issue)这样调用。注意这里是第一种通知方式
  - 该方法内部对Issue相关环境变量进行赋值，譬如引发该Issue的Plugin信息
  - 该方法最终触发PluginListener#onReportIssue

~~~java
public interface IPlugin {

    /**
     * 用于标识当前的监控，相当于名称索引(也可用classname直接索引)
     */
    String getTag();

    /**
     * 在Matrix对象构建时被调用
     */
    void init(Application application, PluginListener pluginListener);

    /**
     * 对activity前后台转换的感知能力
     */
    void onForeground(boolean isForeground);

    void start();

    void stop();

    void destroy();
    
}

public interface PluginListener {
    void onInit(Plugin plugin);

    void onStart(Plugin plugin);

    void onStop(Plugin plugin);

    void onDestroy(Plugin plugin);

    void onReportIssue(Issue issue);
}
~~~

**Matrix对外接口**

~~~java
public class Matrix {
    private static final String TAG = "Matrix.Matrix";

	/**********************************  单例实现 **********************/
	private static volatile Matrix sInstance;
    public static Matrix init(Matrix matrix) {
        if (matrix == null) {
            throw new RuntimeException("Matrix init, Matrix should not be null.");
        }
        synchronized (Matrix.class) {
            if (sInstance == null) {
                sInstance = matrix;
            } else {
                MatrixLog.e(TAG, "Matrix instance is already set. this invoking will be ignored");
            }
        }
        return sInstance;
    }
    public static boolean isInstalled() {
        return sInstance != null;
    }
    public static Matrix with() {
        if (sInstance == null) {
            throw new RuntimeException("you must init Matrix sdk first");
        }
        return sInstance;
    }
	 
	
    /****************************  构造函数 **********************/
    private final Application     application;
	private final HashSet<Plugin> plugins;
    private final PluginListener  pluginListener;
	
	private Matrix(Application app, PluginListener listener, HashSet<Plugin> plugins) {
        this.application = app;
        this.pluginListener = listener;
        this.plugins = plugins;
        for (Plugin plugin : plugins) {
            plugin.init(application, pluginListener);
            pluginListener.onInit(plugin);
        }

    }
	
	/****************************  控制能力 **********************/
	
	
    public void startAllPlugins() {
        for (Plugin plugin : plugins) {
            plugin.start();
        }
    }

    public void stopAllPlugins() {
        for (Plugin plugin : plugins) {
            plugin.stop();
        }
    }

    public void destroyAllPlugins() {
        for (Plugin plugin : plugins) {
            plugin.destroy();
        }
    }
	
	/****************************  get | set **********************/
	
    public Plugin getPluginByTag(String tag) {
        for (Plugin plugin : plugins) {
            if (plugin.getTag().equals(tag)) {
                return plugin;
            }
        }
        return null;
    }

    public <T extends Plugin> T getPluginByClass(Class<T> pluginClass) {
        String className = pluginClass.getName();
        for (Plugin plugin : plugins) {
            if (plugin.getClass().getName().equals(className)) {
                return (T) plugin;
            }
        }
        return null;
    }
	/****************************  其他 **********************/
	public static void setLogIml(MatrixLog.MatrixLogImp imp) {
        MatrixLog.setMatrixLogImp(imp);
    }

}
~~~

**Utils 辅助功能**

- MatrixLog 开放给用户的log能力
- MatrixHandlerThread 公共线程，往往用于异步线程执行一些任务
- DeviceUtil 获取相关设备信息
- MatrixUtil 判断主线程等其他utils

**IssuePublisher被监控事件观察者**

- Issue 被监控事件
  type:类型，用于区分同一个tag不同类型的上报
  tag: 该上报对应的tag
  stack:该上报对应的堆栈
  process:该上报对应的进程名
  time:issue 发生的时间

- IssuePublisher 观察者模式

  - 持有一个 发布Listener(其实现往往是上文的 Plugin)

  - 持有一个 已发布信息的Map，在一次运行时长内，避免针对同一事件的重复发布

  - 一般而言，某种监控的监控探测器往往继承该类，并在检测到事件发生时，调用publishIssue(Issue)—>IssuePublisher.OnIssueDetectListener接口的onDetectIssue方法—>最终触发PluginListener#onReportIssue


#### Matrix的模块IO监测核心源码分析

IO Canary:核心的作用是检测文件 IO 问题，包括：文件 IO 监控和 Closeable Leak 监控。要想理解IO的监测和看懂开源的代码，最重要的基础就是掌握Native和Java层面的Hook。

Java层面的hook主要是基于反射技术，大家都比较熟悉了，那我们来聊一聊Native层面的Hook。在JVM层面，Android使用Android **PLT** （Procedure Linkage Table）Hook和**Inline Hook**、**ptrace**三种主流的技术。

Matrix采用PLT的技术来实现SO文件API的Hook。

ELF: https://en.wikipedia.org/wiki/Executable_and_Linkable_Format

https://refspecs.linuxbase.org/elf/elf.pdf

**ELF文件的三种形式：**

- 可重定位的对象文件(Relocatable file)，也就是我们熟悉的那些.o文件，当然.a库也算（因为它是.o的集合）
- 可执行的对象文件(Executable file)，linux下的可执行文件
- 可被共享的对象文件(Shared object file)，动态链接库.so

我们思考下C的程序是需要进行编译和链接再到最后的运行的。那么ELF文件从这个参与程序运行的角度也是分为2种视图的。

![image-20190916143208858](https://user-gold-cdn.xitu.io/2020/5/19/1722d171f4bac8d6?w=862&h=558&f=jpeg&s=56205)

​		ELF header 位于文件的最开始处，描述整个文件的组织结构。Program Header Table 告诉系统如何创建进程镜像，在执行程序时必须存在，在 relocatable files 中则不需要。每个 program header 分别描述一个 segment，包括 segment 在文件和内存中的大小及地址等等。执行视图中的 segment 其实是由很多个 section 组成。在一个进程镜像中通常具有 text segment 和 data segment 等等。

**关于重定位的作用和概念**

重定位就是把符号引用与符号定义链接起来的过程，这也是 android linker 的主要工作之一。当程序中调用一个函数时，相关的 call 指令必须在执行期将控制流转到正确的目标地址。所以，so 文件中必须包含一些重定位相关的信息，linker 据此完成重定位的工作。

https://docs.oracle.com/cd/E19683-01/816-1386/chapter6-54839/index.html

https://android.googlesource.com/platform/bionic/+/master/linker/linker.cpp

![image-20190916144744091](https://user-gold-cdn.xitu.io/2020/5/19/1722d166880c8ca0?w=1661&h=1080&f=jpeg&s=157527)

![image-20190916151707293](https://user-gold-cdn.xitu.io/2020/5/19/1722d171ddf8be9b?w=1294&h=1080&f=jpeg&s=251497)

符号表表项的结构为elf32_sym：

```
typedef struct elf32_sym {
    Elf32_Word  st_name;    /* 名称 – index into string table */
    Elf32_Addr  st_value;   /* 偏移地址 */
    Elf32_Word  st_size;    /* 符号长度（ 例如，函数的长度） */
    unsigned char   st_info;    /* 类型和绑定类型 */
    unsigned char   st_other;   /* 未定义 */
    Elf32_Half  st_shndx;   /* section header的索引号，表示位于哪个section中 */
} Elf32_Sym;
```

重定位核心代码：

http://androidxref.com/8.0.0_r4/xref/bionic/linker/linker.cpp#2513  （具体的重定位类型定义和计算方法可以参考elf说明文档的 4.6.1.2 小节）

![image-20190916152120420](https://user-gold-cdn.xitu.io/2020/5/19/1722d171ed904ede?w=1392&h=1010&f=jpeg&s=289700)

**Android PLT Hook的基本原理**

​		Linux在执行动态链接的ELF的时候，为了优化性能使用了一个叫延时绑定的策略。当在动态链接的ELF程序里调用共享库的函数时，第一次调用时先去查找PLT表中相应的项目，而PLT表中再跳跃到GOT表中希望得到该函数的实际地址，但这时GOT表中指向的是PLT中那条跳跃指令下面的代码，最终会执行`_dl_runtime_resolve()`并执行目标函数。因此，PLT Hook通过直接修改GOT表，使得在调用该共享库的函数时跳转到的是用户自定义的Hook功能代码。

![image-20190916153407043](https://user-gold-cdn.xitu.io/2020/5/19/1722d171f562ba6a?w=1340&h=1080&f=jpeg&s=160156)

IO 监控流程：

![image-20190916154924031](https://user-gold-cdn.xitu.io/2020/5/19/1722d16a67277534?w=1256&h=314&f=jpeg&s=34874)

~~~c++
JNIEXPORT jboolean JNICALL
        Java_com_tencent_matrix_iocanary_core_IOCanaryJniBridge_doHook(JNIEnv *env, jclass type) {
            __android_log_print(ANDROID_LOG_INFO, kTag, "doHook");

            for (int i = 0; i < TARGET_MODULE_COUNT; ++i) {
                const char* so_name = TARGET_MODULES[i];
                __android_log_print(ANDROID_LOG_INFO, kTag, "try to hook function in %s.", so_name);

                loaded_soinfo* soinfo = elfhook_open(so_name);
                if (!soinfo) {
                    __android_log_print(ANDROID_LOG_WARN, kTag, "Failure to open %s, try next.", so_name);
                    continue;
                }
								//hook OS
                elfhook_replace(soinfo, "open", (void*)ProxyOpen, (void**)&original_open);
                elfhook_replace(soinfo, "open64", (void*)ProxyOpen64, (void**)&original_open64);

                bool is_libjavacore = (strstr(so_name, "libjavacore.so") != nullptr);
                if (is_libjavacore) {
                    if (!elfhook_replace(soinfo, "read", (void*)ProxyRead, (void**)&original_read)) {
                        __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook read failed, try __read_chk");
                        if (!elfhook_replace(soinfo, "__read_chk", (void*)ProxyRead, (void**)&original_read)) {
                            __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook failed: __read_chk");
                            elfhook_close(soinfo);
                            return false;
                        }
                    }
	                  //hook OS
                    if (!elfhook_replace(soinfo, "write", (void*)ProxyWrite, (void**)&original_write)) {
                        __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook write failed, try __write_chk");
                        if (!elfhook_replace(soinfo, "__write_chk", (void*)ProxyWrite, (void**)&original_write)) {
                            __android_log_print(ANDROID_LOG_WARN, kTag, "doHook hook failed: __write_chk");
                            elfhook_close(soinfo);
                            return false;
                        }
                    }
                }
	              //hook OS
                elfhook_replace(soinfo, "close", (void*)ProxyClose, (void**)&original_close);

                elfhook_close(soinfo);
            }

            return true;
        }

~~~

hook的替换核心代码：（本质上就是置指针替换）

![image-20190916155810095](https://user-gold-cdn.xitu.io/2020/5/19/1722d16a66bd0ac7?w=1878&h=1056&f=jpeg&s=206690)

看看代理方法：

![image-20190916155946965](https://user-gold-cdn.xitu.io/2020/5/19/1722d16b14867d8a?w=1086&h=538&f=jpeg&s=55140)

很明显腾讯的人没有考虑到自线程的问题。这里可以优化的。具体的其他部分的细节请参见源码。

####Matrix的模块内存泄漏监测核心源码分析

https://github.com/Tencent/matrix/wiki/Matrix-Android-ResourceCanary

设计目的：

- 自动且较为准确地监测Activity泄漏，发现泄漏之后再触发Dump Hprof而不是根据预先设定的内存占用阈值盲目触发
- 自动获取泄漏的Activity和冗余Bitmap对象的引用链
- 能灵活地扩展Hprof的分析逻辑，必要时允许提取Hprof文件人工分析

为了解决线上监测和后台分析，Matrix的ResourceCanary最终决定将监测步骤和分析步骤拆成两个独立的工具，以满足设计目标。

- Hprof文件留在了服务端，为人工分析提供了机会
- 如果跳过触发Dump Hprof，甚至可以把监测步骤在现网环境启用，以发现测试阶段难以触发的Activity泄漏

客户端解决的问题是内存泄漏的监测和Hprof文件的裁剪，具体看下面的流程图：

![image-20190916162339863](https://user-gold-cdn.xitu.io/2020/5/19/1722d16c92d9e973?w=2221&h=1080&f=jpeg&s=351407)

**ResourcePlugin**

`ResourcePlugin`是该模块的入口，负责注册Android生命周期的监听以及配置部分参数和接口回调。

**ActivityRefWatcher**
ActivityRefWatcher负责的任务有弹出Dump内存的Dialog、Dump内存数据、读取内存数据裁剪Hprof文件、生成包含裁剪后的Hprof以及泄漏的Activity的信息（进程号、Activity名、时间等）、通知主线程完成内存信息的备份并关闭Dialog。

我们看下最为核心的内存泄漏监测代码：

~~~java
//ActivityRefWatcher
private final Application.ActivityLifecycleCallbacks mRemovedActivityMonitor = new ActivityLifeCycleCallbacksAdapter() {
        private int mAppStatusCounter = 0;
        private int mUIConfigChangeCounter = 0;

        @Override
        public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
            mCurrentCreatedActivityCount.incrementAndGet();
        }

        @Override
        public void onActivityStarted(Activity activity) {
            if (mAppStatusCounter <= 0) {
                MatrixLog.i(TAG, "we are in foreground, start watcher task.");
                mDetectExecutor.executeInBackground(mScanDestroyedActivitiesTask);
            }
            if (mUIConfigChangeCounter < 0) {
                ++mUIConfigChangeCounter;
            } else {
                ++mAppStatusCounter;
            }
        }

        @Override
        public void onActivityStopped(Activity activity) {
            if (activity.isChangingConfigurations()) {
                --mUIConfigChangeCounter;
            } else {
                --mAppStatusCounter;
                if (mAppStatusCounter <= 0) {
                    MatrixLog.i(TAG, "we are in background, stop watcher task.");
                    mDetectExecutor.clearTasks();
                }
            }
        }

        @Override
        public void onActivityDestroyed(Activity activity) {
          //当activity销毁的时候开始。。。
            pushDestroyedActivityInfo(activity);
            synchronized (mDestroyedActivityInfos) {
                mDestroyedActivityInfos.notifyAll();
            }
        }
    };
~~~

~~~java
 private void pushDestroyedActivityInfo(Activity activity) {
        final String activityName = activity.getClass().getName();
   	    //该Activity确认存在泄漏，且已经上报
        if (isPublished(activityName)) {
            MatrixLog.d(TAG, "activity leak with name %s had published, just ignore", activityName);
            return;
        }
        final UUID uuid = UUID.randomUUID();
        final StringBuilder keyBuilder = new StringBuilder();
        //生成Activity实例的唯一标识
        keyBuilder.append(ACTIVITY_REFKEY_PREFIX).append(activityName)
            .append('_').append(Long.toHexString(uuid.getMostSignificantBits())).append(Long.toHexString(uuid.getLeastSignificantBits()));
        final String key = keyBuilder.toString();
        //构造一个数据结构，表示一个已被destroy的Activity
        final DestroyedActivityInfo destroyedActivityInfo
            = new DestroyedActivityInfo(key, activity, activityName, mCurrentCreatedActivityCount.get());
       //放入ConcurrentLinkedQueue数据结构中，用于后续的检查
        mDestroyedActivityInfos.add(destroyedActivityInfo);
    }
~~~

内存泄漏的核心代码：

~~~java
private final RetryableTask mScanDestroyedActivitiesTask = new RetryableTask() {

        @Override
        public Status execute() {
            // If destroyed activity list is empty, just wait to save power.
            while (mDestroyedActivityInfos.isEmpty()) {
                synchronized (mDestroyedActivityInfos) {
                    try {
                        mDestroyedActivityInfos.wait();
                    } catch (Throwable ignored) {
                        // Ignored.
                    }
                }
            }

            // Fake leaks will be generated when debugger is attached.
           //Debug调试模式，检测可能失效，直接return
            if (Debug.isDebuggerConnected() && !mResourcePlugin.getConfig().getDetectDebugger()) {
                MatrixLog.w(TAG, "debugger is connected, to avoid fake result, detection was delayed.");
                return Status.RETRY;
            }
             //创建一个对象的弱引用
            final WeakReference<Object> sentinelRef = new WeakReference<>(new Object());
            triggerGc();
            //系统未执行GC，直接return
            if (sentinelRef.get() != null) {
                // System ignored our gc request, we will retry later.
                MatrixLog.d(TAG, "system ignore our gc request, wait for next detection.");
                return Status.RETRY;
            }

            final Iterator<DestroyedActivityInfo> infoIt = mDestroyedActivityInfos.iterator();

            while (infoIt.hasNext()) {
                final DestroyedActivityInfo destroyedActivityInfo = infoIt.next();
               //该实例对应的Activity已被标泄漏，跳过该实例
                if (isPublished(destroyedActivityInfo.mActivityName)) {
                    MatrixLog.v(TAG, "activity with key [%s] was already published.", destroyedActivityInfo.mActivityName);
                    infoIt.remove();
                    continue;
                }
               //若不能通过弱引用获取到Activity实例，表示已被回收，跳过该实例
                if (destroyedActivityInfo.mActivityRef.get() == null) {
                    // The activity was recycled by a gc triggered outside.
                    MatrixLog.v(TAG, "activity with key [%s] was already recycled.", destroyedActivityInfo.mKey);
                    infoIt.remove();
                    continue;
                }
								 //该Activity实例 检测到泄漏的次数+1
                ++destroyedActivityInfo.mDetectedCount;
    //当前显示的Activity实例与泄漏的Activity实例相差几个Activity跳转
                long createdActivityCountFromDestroy = mCurrentCreatedActivityCount.get() - destroyedActivityInfo.mLastCreatedActivityCount;
               //若Activity实例 检测到泄漏的次数未达到阈值，或者泄漏的Activity与当前显示的Activity很靠近，可认为是一种容错手段(实际应用中有这种场景)，跳过该实例
                if (destroyedActivityInfo.mDetectedCount < mMaxRedetectTimes
                    || (createdActivityCountFromDestroy < CREATED_ACTIVITY_COUNT_THRESHOLD && !mResourcePlugin.getConfig().getDetectDebugger())) {
                    // Although the sentinel tell us the activity should have been recycled,
                    // system may still ignore it, so try again until we reach max retry times.
                    MatrixLog.i(TAG, "activity with key [%s] should be recycled but actually still \n"
                            + "exists in %s times detection with %s created activities during destroy, wait for next detection to confirm.",
                        destroyedActivityInfo.mKey, destroyedActivityInfo.mDetectedCount, createdActivityCountFromDestroy);
                    continue;
                }

                MatrixLog.i(TAG, "activity with key [%s] was suspected to be a leaked instance.", destroyedActivityInfo.mKey);
                if (mHeapDumper != null) {
                    final File hprofFile = mHeapDumper.dumpHeap();
                    if (hprofFile != null) {
                        markPublished(destroyedActivityInfo.mActivityName);
                        final HeapDump heapDump = new HeapDump(hprofFile, destroyedActivityInfo.mKey, destroyedActivityInfo.mActivityName);
                        mHeapDumpHandler.process(heapDump);
                        infoIt.remove();
                    } else {
                        MatrixLog.i(TAG, "heap dump for further analyzing activity with key [%s] was failed, just ignore.",
                                destroyedActivityInfo.mKey);
                        infoIt.remove();
                    }
                } else {
                    // Lightweight mode, just report leaked activity name.
                    MatrixLog.i(TAG, "lightweight mode, just report leaked activity name.");
                    markPublished(destroyedActivityInfo.mActivityName);
                    if (mResourcePlugin != null) {
                        final JSONObject resultJson = new JSONObject();
                        try {
                            resultJson.put(SharePluginInfo.ISSUE_ACTIVITY_NAME, destroyedActivityInfo.mActivityName);
                        } catch (JSONException e) {
                            MatrixLog.printErrStackTrace(TAG, e, "unexpected exception.");
                        }
                        mResourcePlugin.onDetectIssue(new Issue(resultJson));
                    }
                }
            }

            return Status.RETRY;
        }
    };
~~~

**RetryableTaskExecutor**

`RetryableTaskExecutor`中包含了两个Handler对象，一个`mBackgroundHandler`和`mMainHandler`，分别给主线程和后台的线程提交任务。默认重试次数是3。

**AndroidHeapDumper**

`AndroidHeapDumper`这个其实就是封装了`android.os.Debug`的接口的类。主要是用系统提供的类`android.os.Debug`Dump内存信息到本地，`android.os.Debug`会在本地生成一个Hprof文件，也是Matrix需要分析和裁剪的原始文件。

注意：一般Dump一次要5s～15s之间，线上建议不要使用，有一定的风险。

Dump的时候，`AndroidHeapDumper`会展示一个Dialog提示当前正在Dump中，Dump完毕就会将Dialog关闭。

~~~java
 Debug.dumpHprofData(hprofFile.getAbsolutePath());
~~~

####Matrix的模块Trace模块核心源码分析

Trace Canary: 用于监控界面流畅性、启动耗时、页面切换耗时、慢函数及卡顿等问题。（思考一下，技术上怎么实现）

入口函数探针分析：

~~~java
public class TracePlugin extends Plugin {
    private static final String TAG = "Matrix.TracePlugin";

    private final TraceConfig traceConfig;
    private EvilMethodTracer evilMethodTracer;//慢函数
    private StartupTracer startupTracer; //启动监测
    private FrameTracer frameTracer; //fps
    private AnrTracer anrTracer; //anr

    public TracePlugin(TraceConfig config) {
        this.traceConfig = config;
    }
  ...
~~~

【关键知识点1】: MessageQueue中的IdleHandler接口有什么用？

在Android中，我们可以处理Message，这个Message我们可以立即执行也可以delay 一定时间执行。Handler线程在执行完所有的Message消息，它会wait，进行阻塞，直到有新的Message到达。如果这样子，那么这个线程也太浪费了。MessageQueue提供了另一类消息，IdleHandler。也就是说当我们的MessageQueue中的消息被处理完后，就会触发一次或者多次回调消息。

应用场景：1、比如主线程在开始加载页面完成后，如果线程空闲就提前加载些二级页面的内容。

​                    2、消息触发器 例如在APM中的作用

​                    3、优化Activity的启动时间，在Resume中是不是可以增加idle的监听

~~~java
Looper.myQueue().addIdleHandler(() -> {
            initializeData();
		    return false;
        });
~~~



源码分析：

~~~java
  Message next(){
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }
								//根据IdleHandler中的回掉方法来判断是否移除
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
~~~

**LooperMonitor**类监测卡顿问题：

发生在Android主线程的每16ms重绘操作依赖于Main Looper中消息的发送和获取。如果App一切运行正常，无卡顿无丢帧现象发生，那么开发者的代码在主线程Looper消息队列中发送和接收消息的时间会很短，理想情况是16ms，这是也是Android系统规定的时间。但是，如果一些发生在主线程的代码写的太重，执行任务花费时间太久，就会在主线程延迟Main Looper的消息在16ms尺度范围内的读和写。

**我们如何检测卡顿的问题？**

使用主线程的Looper监测系统发生的卡顿和丢帧。编程技巧是设置一个阈值，看是否可以打印stack信息。

网络上说使用Android的Choreographer监测App发生的UI卡顿丢帧问题，本质上还是利用了Android的主线程的Looper消息机制。Android 系统每隔 16.67 ms 都会发送一个 VSYNC 信号触发 UI 的渲染，正常情况下两个 VSYNC 信号之间是 16.67 ms ，如果超过 16.67 ms 则可以认为渲染发生了卡顿。

~~~java
Choreographer.getInstance()
            .postFrameCallback(new Choreographer.FrameCallback() {
                @Override
                public void doFrame(long l) {
                     if(frameTimeNanos - mLastFrameNanos > 100) {
                        ...
                     }
                     mLastFrameNanos = frameTimeNanos;
                     Choreographer.getInstance().postFrameCallback(this);
                }
        });
~~~

**本质**：判断相邻的两次 `FrameCallback.doFrame(long l)` 间隔是否超过阈值，如果超过阈值则发生了卡顿，则可以在另外一个子线程中 dump 当前主线程的堆栈信息进行分析。

**消息处理**

**UIThreadMonitor**类

init():

~~~java
public void init(TraceConfig config) {
        if (Thread.currentThread() != Looper.getMainLooper().getThread()) {
            throw new AssertionError("must be init in main thread!");
        }
        this.isInit = true;
        this.config = config;
        choreographer = Choreographer.getInstance();
        callbackQueueLock = reflectObject(choreographer, "mLock");              
        callbackQueues = reflectObject(choreographer, "mCallbackQueues");                                                                    // 代码 1

        addInputQueue = reflectChoreographerMethod(callbackQueues[CALLBACK_INPUT], ADD_CALLBACK, long.class, Object.class, Object.class);             // 代码 2
        addAnimationQueue = reflectChoreographerMethod(callbackQueues[CALLBACK_ANIMATION], ADD_CALLBACK, long.class, Object.class, Object.class);     // 代码 3
        addTraversalQueue = reflectChoreographerMethod(callbackQueues[CALLBACK_TRAVERSAL], ADD_CALLBACK, long.class, Object.class, Object.class);     // 代码 4  
        frameIntervalNanos = reflectObject(choreographer, "mFrameIntervalNanos");

        LooperMonitor.register(new LooperMonitor.LooperDispatchListener() {                                                                  // 代码 5  
            @Override
            public boolean isValid() {
                return isAlive;
            }

            @Override
            public void dispatchStart() {
                super.dispatchStart();
                UIThreadMonitor.this.dispatchBegin();                                                                                       // 代码 6
            }

            @Override
            public void dispatchEnd() {
                super.dispatchEnd();
                UIThreadMonitor.this.dispatchEnd();                                                                                      // 代码 7
            }

        });

                ......

    }  
~~~



~~~java
- 代码 1：通过反射拿到了 Choreographer 实例的 mCallbackQueues 属性，mCallbackQueues 是一个回调队列数组 CallbackQueue[] mCallbackQueues，其中包括四个回调队列，
    第一个是输入事件回调队列 CALLBACK_INPUT = 0，
    第二个是动画回调队列 CALLBACK_ANIMATION = 1，
    第三个是遍历绘制回调队列 CALLBACK_TRAVERSAL = 2，
    第四个是提交回调队列 CALLBACK_COMMIT = 3。
这四个阶段在每一帧的 UI 渲染中是依次执行的，每一帧中各个阶段开始时都会回调 mCallbackQueues 中对应的回调队列的回调方法。
- 代码 2：通过反射拿到输入事件回调队列的 addCallbackLocked 方法
- 代码 3：通过反射拿到动画回调队列的 addCallbackLocked 方法
- 代码 4：通过反射拿到遍历绘制回调队列的addCallbackLocked 方法
- 代码 5：通过 LooperMonitor.register(LooperDispatchListener listener) 方法向 LooperMonitor 中设置 LooperDispatchListener listener
- 代码 6：在 Looper.loop() 中的消息处理开始时的回调
- 代码 7：在 Looper.loop() 中的消息处理结束时的回调
~~~

核心：

```java
private void dispatchBegin() {
  //记录2个时间 线程起始时间 和CPU的开始时间
        token = dispatchTimeMs[0] = SystemClock.uptimeMillis();
        dispatchTimeMs[2] = SystemClock.currentThreadTimeMillis();
        AppMethodBeat.i(AppMethodBeat.METHOD_ID_DISPATCH);

        synchronized (observers) {
            for (LooperObserver observer : observers) {
                if (!observer.isDispatchBegin()) {
                    observer.dispatchBegin(dispatchTimeMs[0], dispatchTimeMs[2], token);
                }
            }
        }
    }  
```

~~~java
private void dispatchEnd() {                                                                                            // 代码 3
        if (isBelongFrame) {
            doFrameEnd(token);
        }

        dispatchTimeMs[3] = SystemClock.currentThreadTimeMillis();
        dispatchTimeMs[1] = SystemClock.uptimeMillis();

        AppMethodBeat.o(AppMethodBeat.METHOD_ID_DISPATCH);

        synchronized (observers) {
            for (LooperObserver observer : observers) {
                if (observer.isDispatchBegin()) {
                    observer.dispatchEnd(dispatchTimeMs[0], dispatchTimeMs[2], dispatchTimeMs[1], dispatchTimeMs[3], token, isBelongFrame);
                }
            }
        }

    }  
~~~

核心点总结：

- 在 UIThreadMonitor 中有两个长度是三的数组 `queueStatus` 和 `queueCost` 分别对应着每一帧中输入事件阶段、动画阶段、遍历绘制阶段的状态和耗时，`queueStatus` 有三个值：DO_QUEUE_DEFAULT、DO_QUEUE_BEGIN 和 DO_QUEUE_END。
-  `UIThreadMonitor` 实现 `Runnable` 接口，也是为了将 `UIThreadMonitor` 作为输入事件回调 `CALLBACK_INPUT` 的回调方法，设置到 `Choreographer` 中去的。

 **看到这里应该搞明白了卡顿的检测原理，那么FPS的计算呢？**

每一帧的时间信息通过 `HashSet<LooperObserver> observers` 回调出去，看一下是在哪里向 `observers` 添加 `LooperObserver` 回调的。主要看一下 **`FrameTracer`** 这个类，其中涉及到了帧率 FPS 的计算相关的代码。

`FPSCollector` 是 `FrameTracer` 的一个内部类，实现了 `IDoFrameListener` 接口，主要逻辑是在 `doFrameAsync()` 方法中

- 代码 1：会根据当前 ActivityName 创建一个对应的 FrameCollectItem 对象，存放在 HashMap 中
- 代码 2：调用 `FrameCollectItem#collect()`，计算帧率 FPS 等一些信息
- 代码 3：如果此 Activity 的绘制总时间超过 timeSliceMs（默认是 10s），则调用 `FrameCollectItem#report()` 上报统计数据，并从 HashMap 中移除当前 ActivityName 和对应的 FrameCollectItem 对象

~~~java
private class FPSCollector extends IDoFrameListener {

        private Handler frameHandler = new Handler(MatrixHandlerThread.getDefaultHandlerThread().getLooper());
        private HashMap<String, FrameCollectItem> map = new HashMap<>();

        @Override
        public Handler getHandler() {
            return frameHandler;
        }

        @Override
        public void doFrameAsync(String focusedActivityName, long frameCost, int droppedFrames) {
            super.doFrameAsync(focusedActivityName, frameCost, droppedFrames);
            if (Utils.isEmpty(focusedActivityName)) {
                return;
            }

            FrameCollectItem item = map.get(focusedActivityName);       // 代码 1
            if (null == item) {
                item = new FrameCollectItem(focusedActivityName);
                map.put(focusedActivityName, item);
            }

            item.collect(droppedFrames);                                                                // 代码 2

            if (item.sumFrameCost >= timeSliceMs) { // report                       // 代码 3
                map.remove(focusedActivityName);
                item.report();
            }
        }

    }
~~~

**FrameCollectItem**:计算FPS。

- 根据 `float fps = Math.min(60.f, 1000.f * sumFrame / sumFrameCost)` 计算 fps 值
- sumDroppedFrames 统计总的掉帧个数

- sumFrameCost 代表总耗时

计算帧率参考资料：

https://github.com/friendlyrobotnyc/TinyDancer 熟读代码 并应用到自己的项目中。




关于hook的部分：

[ActivityThreadHacker.java](https://github.com/Tencent/matrix/blob/master/matrix/matrix-android/matrix-trace-canary/src/main/java/com/tencent/matrix/trace/hacker/ActivityThreadHacker.java)

利用了反射机制进行了hook，代码比较清晰，目的很明确就是利用自己写的HackCallback来替换ActivityThread类里的mCallback，达到偷梁换柱的效果，这样做的意义就是可以拦截mCallback的原有的消息，然后选择自己要处理的消息。搞清楚Activity的启动流程 AMS和ActivityThread进程之间的交互。

![image-20190921190329689](https://user-gold-cdn.xitu.io/2020/5/19/1722d16d3cc92700?w=1420&h=920&f=jpeg&s=100733)

获取到Activity的启动时机。

**AppMethodBeat** 通过hook记录每个方法执行的时间。

~~~java
/**
     * hook method when it's called in.
     *
     * @param methodId
     */
    public static void i(int methodId) {

        if (status <= STATUS_STOPPED) {
            return;
        }
        if (methodId >= METHOD_ID_MAX) {
            return;
        }

        if (status == STATUS_DEFAULT) {
            synchronized (statusLock) {
                if (status == STATUS_DEFAULT) {
                    realExecute();
                    status = STATUS_READY;
                }
            }
        }

        if (Thread.currentThread().getId() == sMainThread.getId()) {
            if (assertIn) {
                android.util.Log.e(TAG, "ERROR!!! AppMethodBeat.i Recursive calls!!!");
                return;
            }
            assertIn = true;
            if (sIndex < Constants.BUFFER_SIZE) {
                mergeData(methodId, sIndex, true);
            } else {
                sIndex = -1;
            }
            ++sIndex;
            assertIn = false;
        }
    }

    /**
     * hook method when it's called out.
     *
     * @param methodId
     */
    public static void o(int methodId) {

        if (status <= STATUS_STOPPED) {
            return;
        }
        if (methodId >= METHOD_ID_MAX) {
            return;
        }
        if (Thread.currentThread().getId() == sMainThread.getId()) {
            if (sIndex < Constants.BUFFER_SIZE) {
                mergeData(methodId, sIndex, false);
            } else {
                sIndex = -1;
            }
            ++sIndex;
        }
    }

 /**
     * merge trace info as a long data
     *
     * @param methodId
     * @param index
     * @param isIn
     */
    private static void mergeData(int methodId, int index, boolean isIn) {
        if (methodId == AppMethodBeat.METHOD_ID_DISPATCH) {
          //记录上面2个方法的时间差
            sCurrentDiffTime = SystemClock.uptimeMillis() - sDiffTime;
        }
        long trueId = 0L;
        if (isIn) {
            trueId |= 1L << 63;
        }
        trueId |= (long) methodId << 43;
        trueId |= sCurrentDiffTime & 0x7FFFFFFFFFFL;
        sBuffer[index] = trueId;
        checkPileup(index);
        sLastIndex = index;
    }

~~~

源码读到这里是不是可以想一想，我们应该怎么找方法的调用？

https://github.com/Tencent/matrix/blob/b54b09ae06cc225c1cc9aedc8be39f3db4a2a340/matrix/matrix-android/matrix-gradle-plugin/src/main/java/com/tencent/matrix/trace/MethodTracer.java

利用ASM技术完成在方法前执行“com/tencent/matrix/trace/core/AppMethodBeat”这个class里的`i()`方法，在每个方法最后执行`o()`方法。

###ASM

应用场景：

- 无痕埋点
- hook
- apm监控
- 编译期间动态修改代码需求

**ASM是什么**

ASM 可以直接产生二进制的class 文件，也可以在增强既有类的功能。Java class 被存储在严格格式定义的 .class文件里，这些类文件拥有足够的元数据来解析类中的所有元素：类名称、方法、属性以及 Java 字节码（指令）。

http://blog.jamesdbloom.com/JavaCodeToByteCode_PartOne.html

https://blog.csdn.net/weelyy/article/details/78969412

https://asm.ow2.io/

![image-20190925192210187](https://user-gold-cdn.xitu.io/2020/5/19/1722d1728322f5b2?w=1272&h=326&f=jpeg&s=32562)

![image-20190925192348722](https://user-gold-cdn.xitu.io/2020/5/19/1722d17081179c7f?w=1144&h=1080&f=jpeg&s=109132)

class 文件结构：

- **Magic**：该项存放了一个 Java 类文件的魔数（magic number）和版本信息。一个 Java 类文件的前 4 个字节被称为它的魔数。每个正确的 Java 类文件都是以 0xCAFEBABE 开头的，这样保证了 Java 虚拟机能很轻松的分辨出 Java 文件和非 Java 文件。
- **Version：**该项存放了 Java 类文件的版本信息，它对于一个 Java 文件具有重要的意义。因为 Java 技术一直在发展，所以类文件的格式也处在不断变化之中。类文件的版本信息让虚拟机知道如何去读取并处理该类文件。
- **Constant Pool：**该项存放了类中各种文字字符串、类名、方法名和接口名称、final 变量以及对外部类的引用信息等常量。虚拟机必须为每一个被装载的类维护一个常量池，常量池中存储了相应类型所用到的所有类型、字段和方法的符号引用，因此它在 Java 的动态链接中起到了核心的作用。常量池的大小平均占到了整个类大小的 60% 左右。
- **Access_flag：**该项指明了该文件中定义的是类还是接口（一个 class 文件中只能有一个类或接口），同时还指名了类或接口的访问标志，如 public，private, abstract 等信息。ACC_***
- **This Class：**指向表示该类全限定名称的字符串常量的指针。
- **Super Class：**指向表示父类全限定名称的字符串常量的指针。
- **Interfaces：**一个指针数组，存放了该类或父类实现的所有接口名称的字符串常量的指针。以上三项所指向的常量，特别是前两项，在我们用 ASM 从已有类派生新类时一般需要修改：将类名称改为子类名称；将父类改为派生前的类名称；如果有必要，增加新的实现接口。
- **Fields：**该项对类或接口中声明的字段进行了细致的描述。需要注意的是，fields 列表中仅列出了本类或接口中的字段，并不包括从超类和父接口继承而来的字段。
- **Methods：**该项对类或接口中声明的方法进行了细致的描述。例如方法的名称、参数和返回值类型等。需要注意的是，methods 列表里仅存放了本类或本接口中的方法，并不包括从超类和父接口继承而来的方法。使用 ASM 进行 AOP 编程，通常是通过调整 Method 中的指令来实现的。
- **Class attributes：**该项存放了在该文件中类或接口所定义的属性的基本信息。

**核心类**

- ClassReader：该类用来解析编译过的class字节码文件。（事件生产者）
- ClassWriter：该类用来重新构建编译后的类，比如说修改类名、属性以及方法，甚至可以生成新的类的字节码文件。（事件消费者，是ClassVisitor的子类）
- ClassVisitor：主要负责 “拜访” 类成员信息。其中包括标记在类上的注解，类的构造方法，类的字段，类的方法，静态代码块。
- AdviceAdapter：实现了MethodVisitor接口，主要负责 “拜访” 方法的信息，用来进行具体的方法字节码操作。



#### 函数插桩

什么是函数插桩？

**插桩**：目标程序代码中某些位置**插入或修改**成一些代码，从而在目标程序运行过程中获取某些程序状态并加以分析。简单来说就是**在代码中插入代码**。 那么**函数插桩**，便是在函数中插入或修改代码。**在Android编译过程中，往字节码里插入自定义的字节码**，所以也可以称为**字节码插桩**。

**作用**

函数插桩可以帮助我们实现很多手术刀式的代码设计，如**无埋点统计上报、轻量级AOP**等。 应用到在**Android**中，可以用来做用行为统计、方法耗时统计等功能。



**About Gradle Transform API **

 http://google.github.io/android-gradle-dsl/javadoc/2.1/com/android/build/api/transform/Transform.html

Transform的使用场景:

一般我们使用`Transform`会有下面两种场景

1. 我们需要对编译class文件做自定义的处理。
2. 我们需要读取编译产生的class文件，做一些其他事情，但是不需要修改它。

~~~groovy
public abstract class Transform {
    public abstract String getName(); //自定义的Transform的名字

    public abstract Set<ContentType> getInputTypes(); //Transform处理的输入类型 (CLASSES、RESOURCES)

    public abstract Set<? super Scope> getScopes();

    public abstract boolean isIncremental(); // 是否支持增量编译
}
~~~

Scope代码：

~~~groovy
enum Scope implements ScopeType {
        /** Only the project content */
        PROJECT(0x01), //只是当前工程的代码
        /** Only the project's local dependencies (local jars) */
        PROJECT_LOCAL_DEPS(0x02), // 工程的本地jar
        /** Only the sub-projects. */
        SUB_PROJECTS(0x04),  // 只包含子工工程
        /** Only the sub-projects's local dependencies (local jars). */
        SUB_PROJECTS_LOCAL_DEPS(0x08),
        /** Only the external libraries */
        EXTERNAL_LIBRARIES(0x10),
        /** Code that is being tested by the current variant, including dependencies */
        TESTED_CODE(0x20),
        /** Local or remote dependencies that are provided-only */
        PROVIDED_ONLY(0x40);
    }
~~~

测试代码：

~~~groovy
public void transform(TransformInvocation invocation) {
        for (TransformInput input : invocation.getInputs()) {
            input.getJarInputs().parallelStream().forEach(jarInput -> {
            File src = jarInput.getFile();
            JarFile jarFile = new JarFile(file);
            Enumeration<JarEntry> entries = jarFile.entries();
            while (entries.hasMoreElements()) {
                JarEntry entry = entries.nextElement();
                //处理
            }
        }
    }
~~~

