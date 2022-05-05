# JVM 参数

JVM 中的参数可以分为三大类：

* 标准参数（-）：所有的 JVM 实现都必须实现这些参数的功能，而且向后兼容
* 非标准参数（-X）：默认 JVM 实现这些参数的功能，但是并不保证所有 JVM 实现都满足，且不保证向后兼容
* 非 Stable 参数（-XX）：此类参数各个 JVM 实现会有所不同，将来可能会随时取消，需要慎重使用
  

## 标准参数

* `-client`：设置 JVM 使用客户端模式，这是一般在 PC 机器上使用的模式，启动很快，但性能和内存管理效率并不高

* `-server`：使用服务器模式，启动速度虽然慢（比客户端模式慢 10% 左右），但是性能和内存管理效率很高，适用于服务器，用于生产环境、开发环境或测试环境的服务端

  如果未显式指定以服务器模式或以客户端模式启动的话，则 JVM 自动检测。

  默认情况下，不同的启动模式，执行 GC 的方式有所区别：

  | 启动模式 | 新生代 GC 方式 | 老年代 GC 的方式 |
  | -------- | -------------- | ---------------- |
  | client   | 串行           | 串行             |
  | server   | 并行           | 并发             |

* `-jar`：JVM 从指定 jar 包中的主类开始运行。

* `-classpath / -cp`：JVM 加载和搜索文件的目录路径，多个路径用 `;` 分隔。注意，如果使用了此选项，JVM 就不会再搜索环境变量中定义的 CLASSPATH 路径。

* `-verbose`：这是查询 GC 问题最常用的命令之一，具体参数如：

  * -verbose:class：输出 JVM 载入类的相关信息，当 JVM 报告说找不到类或者类冲突时可此进行诊断。
  * -verbose:gc：输出每次 GC 的相关情况。
  * -verbose:jni：输出本地方法调用的相关情况，一般用于诊断 JNI 调用错误信息。

* `-version`：获取当前 JVM 版本信息。

## 非标准参数

* `-Xmn`：新生代内存大小，包括 Eden 区和两个 Survivor 区的总和，使用方法如：-Xmn65535，-Xmn1024k，-Xmn512m，-Xmn1g (-Xms,-Xmx也是种写法)。

* `-Xms`：初始堆的大小，也是堆大小的最小值，默认值是总共的物理内存 / 64。当堆中可用内存小于一定比例时，堆内存会开始增加，一直增加到 -Xmx 的大小。

* `-Xmx`：堆的最大值，默认值是总共的物理内存 / 64。当堆中可用内存大于一定比例时，堆内存会开始减少，一直减小到 -Xms 的大小。

  生产环境下，建议把 `-Xms` 和 `-Xmx` 设置为相同的大小，防止抖动。

* `-Xss`：设置每个线程的栈内存，默认1M
* `-Xprof`：跟踪正运行的程序，并将跟踪数据在标准输出，适合于开发环境调试。
* `-Xnoclassgc`：关闭针对 Class 的 GC 功能；因为其阻止内存回收，所以可能会导致 OOM 错误。
* `-Xincgc`：开启增量 GC（默认为关闭）。这有助于减少长时间 GC 时应用程序出现的停顿，但由于可能和应用程序并发执行，所以会降低 CPU 对应用的处理能力。

## 非 Stable 参数

以 -XX 表示的非 Stable 参数， JVM（以 HotSpot 为例）中主要的参数可以大致分为 3 类：

* 性能参数（Performance Options）：用于 JVM 的性能调优和内存分配控制，如初始化内存大小的设置
* 行为参数（Behavioral Options）：用于改变 JVM 的基础行为，如 GC 的方式和算法的选择
* 调试参数（Debugging Options）：用于监控、打印、输出等，用于显示 JVM 更加详细的信息

对于非Stable参数，使用方法有4种：

* `-XX:+<option>`：启用选项

* `-XX:-<option>`：不启用选项
* `-XX:<option>=<number>`：给选项设置一个数字类型值，可跟单位，例如 32k、1024m、2g 等
* `-XX:<option>=<string>`：给选项设置一个字符串值，例如 -XX:HeapDumpPath=./dump.core 等

### 性能参数

性能参数往往用来定义内存分配的大小和比例，相比于行为参数和调试参数，一个比较明显的区别是性能参数后面往往跟的有数值，常用如下：

| 参数及其默认值                  | 描述                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| -XX:NewSize=2.125m              | 新生代大小                                                   |
| -XX:MaxNewSize=size             | 新生代最大值                                                 |
| -XX:PermSize=64m                | 永久代分配的初始内存（JDK 1.8 之前）                         |
| -XX:MaxPermSize=64m             | 永久代所能占用的最大内存（JDK 1.8 之前）                     |
| -XX:MetaspaceSize=N             | 元空间分配的初始内存（JDK 1.8 之后）                         |
| -XX:MaxMetaspaceSize=N          | 元空间所能占用的最大内存（JDK 1.8 之前）                     |
| -XX:MaxTenuringThreshold=15     | 对象在新生代存活区切换的次数（坚持过 MinorGC 的次数，每坚持过一次，该值就增加 1），大于该值会进入老年代，所以也叫做年龄阈值 |
| -XX:MaxHeapFreeRatio=70         | GC 后堆中空闲量占的最大比例，大于该值，则堆内存会减少        |
| -XX:MinHeapFreeRatio=40         | GC 后堆中空闲量占的最小比例，小于该值，则堆内存会增加        |
| -XX:NewRatio=2                  | 新生代内存容量与老年代内存容量的比例，默认是 2，表示新生代占 1，老年代占 2，新生代比例是 1/3 |
| -XX:ReservedCodeCacheSize= 32m  | 保留代码占用的内存容量                                       |
| -XX:ThreadStackSize=512         | 设置线程栈大小，若为 0 则使用系统默认值                      |
| -XX:LargePageSizeInBytes=4m     | 设置用于堆的大页面尺寸                                       |
| -XX:PretenureSizeThreshold=size | 大于该值的对象直接晋升入老年代                               |
| -XX:SurvivorRatio=8             | Eden 区域和 Survivor 区的容量比值，默认值为 8，代表Eden：Survivor From：Survivor To = 8 : 1 : 1 |

### 行为参数

行为参数主要用来选择使用什么样的垃圾收集器组合，以及控制运行过程中的 GC 策略等。

| 参数及其默认值             | 描述                                                         |
| -------------------------- | ------------------------------------------------------------ |
| -XX:+UseSerialGC           | 启用串行 GC，即启用 Serial + Serial Old 收集器组合           |
| -XX:+UseParallelGC         | 启用并行 GC，即采用 Parallel Scavenge + Serial Old 收集器组合（服务器模式下的默认组合） |
| -XX:GCTimeRatio=99         | 设置用户执行时间占总时间的比例（默认值 99，即仅有 1% 的时间用于 GC） |
| -XX:MaxGCPauseMillis=time  | 设置 GC 的最大停顿时间（这个参数只对 Parallel Scavenge 有效） |
| -XX:+UseParNewGC           | 使用 ParNew+Serial Old 收集器组合                            |
| -XX:ParallelGCThreads      | 设置执行内存回收的线程数，在 +UseParNewGC 的情况下使用       |
| -XX:+UseParallelOldGC      | 使用 Parallel Scavenge +Parallel Old 收集器组合              |
| -XX:+UseConcMarkSweepGC    | 使用 ParNew + CMS + Serial Old 收集器组合来并发收集，优先使用 ParNew + CMS，当用户线程内存不足时，采用备用方案 Serial Old 收集 |
| -XX:+DisableExplicitGC     | 禁止调用 `System.gc()`，但 JVM 的 GC 仍然有效                |
| -XX:+ScavengeBeforeFullGC  | Minor GC 优先于 Full GC 执行                                 |
| -XX:HandlePromotionFailure | 设置是否允许担保失败                                         |
| -XX:+UseBiasedLocking      | 启用偏向锁                                                   |

### 调试参数

| 参数及其默认值                                   | 描述                                   |
| ------------------------------------------------ | -------------------------------------- |
| -XX:+CITime                                      | 打印消耗在 JIT 编译的时间              |
| -XX:ErrorFile=./hs_err_pid\<pid\>.log            | 保存错误日志或者数据到文件中           |
| -XX:HeapDumpPath=./java_pid\<pid\>.hprof         | 指定导出堆信息时的路径或文件名         |
| -XX:+HeapDumpOnOutOfMemoryError                  | 当首次遭遇 OOM 时导出此时堆中相关信息  |
| -XX:OnError="\<cmd args>;\<cmd args>"            | 出现致命 ERROR 之后运行自定义命令      |
| -XX:OnOutOfMemoryError="\<cmd args>;\<cmd args>" | 当首次遭遇 OOM 时执行自定义命令        |
| -XX:+PrintClassHistogram                         | 遇到 Ctrl-Break 后打印类实例的柱状信息 |
| -XX:+PrintConcurrentLocks                        | 遇到 Ctrl-Break 后打印并发锁的相关信息 |
| -XX:+PrintCommandLineFlags                       | 打印在命令行中出现过的标记             |
| -XX:+PrintCompilation                            | 当一个方法被编译时打印相关信息         |
| -XX:+PrintGC                                     | 每次 GC 时打印相关信息                 |
| -XX:+PrintGCDetails                              | 每次 GC 时打印详细信息                 |
| -XX:+PrintGCTimeStamps                           | 打印每次 GC 的时间戳                   |
| -XX:+TraceClassLoading                           | 跟踪类的加载信息                       |
| -XX:+TraceClassLoadingPreorder                   | 跟踪被引用到的所有类的加载信息         |
| -XX:+TraceClassResolution                        | 跟踪常量池                             |
| -XX:+TraceClassUnloading                         | 跟踪类的卸载信息                       |
| -XX:+TraceLoaderConstraints                      | 跟踪类加载器约束的相关信息             |