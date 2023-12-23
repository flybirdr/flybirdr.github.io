# ART启动过程

我们知道，Zygote通过调用`AppRuntime.Start()`来启动Android运行时。

在`AppRuntime.Start()`中又调用`startVm()`函数来启动虚拟机，在这两个函数中先是配置了许多启动参数，然后调用`libart.so`的导出函数`jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args)`来创建虚拟机并启动虚拟机。

在`Runtime.Start()`中，调用`InitNativeMethods()`来初始化`libart.so`
```mermaid
sequenceDiagram
participant app as AppRuntime[AndroidRuntime]
participant art as libart.so
participant rt as Runtime
participant jvm as JavaVm

app-->>app:start("com.android.internal.os.ZygoteInit", args, zygote)
app-->>app:startVm()
app->>art:JNI_CreateJavaVM()
art->>rt:Current()
art->>rt:Start()
rt->>rt:InitNativeMethods()
rt->>rt:RegisterRuntimeNativeMethods()
rt->>rt:WellKnownClasses::Init(env);
rt->>jvm:LoadLibrary("libicu_jni.so")
rt->>jvm:LoadLibrary("libjavacore.so")
rt->>jvm:LoadLibrary("libopenjdk.so")
rt->>rt:WellKnownClasses::LateInit(env)
rt->>rt:InitializeIntrinsics()
rt->>rt:InitializeCorePlatformApiPrivateFields()
rt->>rt:CreateJitCodeCache()
rt->>rt:CreateJit()
rt->>rt:CreateSystemClassLoader()
rt->>rt:StartDaemonThreads()

```