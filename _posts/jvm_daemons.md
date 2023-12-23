# JVM的几个守护线程

在`Runtime`的`Start()`中会调用到`StartDaemonThreads()`:

```cpp
void Runtime::StartDaemonThreads() {
  ScopedTrace trace(__FUNCTION__);
  VLOG(startup) << "Runtime::StartDaemonThreads entering";

  Thread* self = Thread::Current();

  // Must be in the kNative state for calling native methods.
  CHECK_EQ(self->GetState(), kNative);

  JNIEnv* env = self->GetJniEnv();
  env->CallStaticVoidMethod(WellKnownClasses::java_lang_Daemons,
                            WellKnownClasses::java_lang_Daemons_start);
  if (env->ExceptionCheck()) {
    env->ExceptionDescribe();
    LOG(FATAL) << "Error starting java.lang.Daemons";
  }

  VLOG(startup) << "Runtime::StartDaemonThreads exiting";
}
```
调用这个方法后将使用反射调用到`java.lang.Daemons`的`start()`方法，这个方法实现如下：
```java
    private static final Daemon[] DAEMONS = new Daemon[] {
            HeapTaskDaemon.INSTANCE,
            ReferenceQueueDaemon.INSTANCE,
            FinalizerDaemon.INSTANCE,
            FinalizerWatchdogDaemon.INSTANCE,
    };
    @UnsupportedAppUsage
    public static void start() {
        for (Daemon daemon : DAEMONS) {
            daemon.start();
        }
    }
```
可以看到一共有4个守护线程：
* `HeapTaskDaemon`用来
* `ReferenceQueueDaemon`用来
* `FinalizerDaemon`用来从`FinalizerReference.queue`取出`Finalize`对象并执行`finalize()`
* `FinalizerWatchdogDaemon`用来监测`FinalizerDaemon`是否卡住，如果执行`finalize()`超过`MAX_FINALIZATION_MILLIS`就认为卡住。如果卡住看门狗会退出虚拟机。