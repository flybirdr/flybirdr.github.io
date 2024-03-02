
# JDPA
`JPDA（Java Platform Debugger Architecture）`是`Java`平台调试架构的缩写。它属于多层架构，包括`JVMTI`、`JDWP`、`JDI`。

`JPDA`被设计为三层：

`JVMTI`：`Java VM Tool Interface` 定义的是一系列跟调试相关的接口，由 `VM` 实现这些接口。`JVM` 提供的一个类似钩子的机制，通过` JVMTI`，可以指挥` JVM` 执行某些操作，例如停在断点处，也可以在 `JVM` 运行过程中，发生某些事件时，通过钩子通知外部感兴趣的人。
`JDWP`：`Java Debug Wire Protocol`，`Java`调试线协议，运行在`JVMTI Agent`的后端模块
`JDI`：`Java Debug Interface`，`Java`调试接口。`Remote`使用`JDWP`与`JVMTI Agent`通信，而`JDI`则将通信细节封装起来，这样就可以方便地调用`JVMTI`。

为什么 `JPDA`机制需要设计层三层呢？原因有几个：

调试方可能在远程进行调试，而`JDWP` 协议又是一个非常底层的二进制协议，实现起来需要花费大量的成本。所以通过 `JDI` 对 `JDWP` 协议进行实现，并对外提供 `API`。
`JDI` 不仅仅是实现了`JDWP` 协议那么简单，它还实现了队列、缓存、连接初始化等等服务，这些服务都可以简单地通过`JDI` 的 `API` 来使用。
有了 `JVMTI`，调试就可以与具体的 `JVM` 解耦，不同类型的 `JVM` 只要遵循 `JVMTI` 规范即可，`JDWP` 不需要假设它正在与某种类型的 JVM 通信。
# Android系统级调试
Android使用`art`或者`dalvik`作为运行时，当然也实现了`JVMTI`，`VM`启动时会根据启动参数决定是否启动`JDWP`线程，这个线程作为`JVMTI Agent`解析`JDWP`命令并且使用`JVMTI`与`VM`通信。在Android上通常使用`adb`或者`socket`来传输`JDWP`命令。
## 调试系统进程：

```java	frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java
        
	/**
     * Applies debugger system properties to the zygote arguments.
     *
     * If "ro.debuggable" is "1", all apps are debuggable. Otherwise,
     * the debugger state is specified via the "--enable-jdwp" flag
     * in the spawn request.
     *
     * 如果"ro.debuggable"置为1，所有程序都可调试。
     * 否则在启动标志中加入--enable-jdwp
     *
     * @param args non-null; zygote spawner args
     */
    public static void applyDebuggerSystemProperty(Arguments args) {
        if (RoSystemProperties.DEBUGGABLE) {
            args.runtimeFlags |= Zygote.DEBUG_ENABLE_JDWP;
        }
    }
    
	//而真正启动Debugger则是在art运行时中
	void Runtime::InitNonZygoteOrPostFork(...){
		// Start the JDWP thread. If the command-line debugger flags specified "suspend=y",
  		// this will pause the runtime (in the internal debugger implementation), so we probably want
  		// this to come last.
  		GetRuntimeCallbacks()->StartDebugger();
	}
```
1、设置系统属性
```shell
adb shell setprop ro.debuggable 1
```
2、重启要调试的应用
```shell
adb shell pkill -9 [processName]
```
3、在Android Studio中附加需要调试的应用

## 调试system_server
1、使用`Intellij Idea`打开对应源码
2、点击`Edit Configurations`增加一个配置
	①名称随意填
	②`Debugger Mode`选择`Attach`将来主动附加`system_server`
	③`Transport`选择`socket`
	④`Port`填写`1235`（可以是你喜欢的任何端口）
	⑤参数应该是这样的`-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:1235`
1、设置系统属性

```shell
adb shell setprop ro.debuggable 1
```
2、重启要调试的应用

```shell
adb shell pkill -9 system_server
```
3、在Android Studio选择`system_process`

至此，就可以下断点调试`system_server`中的`AMS`，`WMS`，`PKMS`了。
