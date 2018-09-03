---
title: Android Framework层的JNI机制（一）
---
JNI（Java Native Interface）Java本地接口。最初对JNI的了解，仅仅停留在Java通过JNI可以实现对C/C++函数的调用。比如，首先在Java中写好native方法。然后在C或C++中文件中，定义一个对应的函数，在这个函数中，实现自己的代码或者调用其他的标准库。最后加载一下生成的动态库，便可以开始使用这个native方法。<!--more-->像是在照葫芦画瓢，知其然，不知其所以然。最近在看Framework层中JNI相关的代码，加上网上大咖的神贴，综合理解，先以Framework中Log为研究对象，分析JNI在其中的使用。

# 概要
+ **JNI机制基本要点**
+ **Android中JNI的存在方式**
+ **Framework层Log的JNI使用**

# 一、JNI机制基本要点
用过JNI的工程师，都接触过下面这些知识点：
+ JavaVM：表示Java虚拟机。
+ JNIEnv：表示JNI环境的上下文，例如注册、查找类、异常等。
+ jclass：在JNI中表示的Java类。
+ jmethodID：在JNI中表示的Java类中的方法。
+ jfiledID：在JNI中表示IDEJava类中的属性。
+ 线程：JNI中通过AttachCurrentThread和DetachCurrentThread方法，实现和Java线程的结合。

它们都在一个叫jni.h的头文件头文件，这个头文件是JNI机制中很重要的一个头文件。
源代码路径：`/libnativehelper/include/nativehelper/jni.h`。

在libnativehelper目录下的源文件，编译后会生成一个libnativehelper.so的动态库。其实，jni.h是Android根据Java本地调用的标准写成的一个头文件，在它里面包括了基本类型（类型的映射），以及JavaVM，JNIEnv，jclass，jmethodID，jfiledID等数据结构的定义。

JavaVM对应于jni.h中JNIInvokeInterface结构体，表示虚拟机。JNIEnv对应于JNINativeInterface结构体，表示JNI的环境。在JNI的使用过程中，所调用的功能大都来自JNINativeInterface结构体。例如，处理Java属性和方法的查找，Java属性的访问，Java方法的调用等功能。另外，在JNINativeInterface结构体中，涉及到的一个JNINativeMethod结构体，它表示在本地实现的一个方法，即native方法，后面进行JNI注册的时候会用到。

# 二、Android中JNI的存在方式
Android中JNI的存在方式主要分两种： 框架层和应用层的JNI使用。不对应用层的使用情况进行介绍，主要目的还是看看框架层里面的JNI。
在Android框架中，JNI库是一些普通的本地动态库，被放置在目标系统的/system/lib目录中。
Java框架层，最主要的JNI内容源代码路径为：`/frameworks/base/core/jni`。
这里面的代码会生成一个libandroid_runtiem.so的动态库。接下来要分析的Log中JNI的使用，就在这个目录之中。

# 三、Framework层Log相关
1. **Java框架层的Log**
在编程的时候，大家都用过Log，其实这个我们经常使用的Log工具，在Java框架层最终调用的是native方法。
贴上源码的路径：`/frameworks/base/core/java/android/util/Log.java`。
如果感兴趣，可以进去瞧一瞧。咱们以Log.java中的println_native()这个本地方法，进行分析。

2. **Log的JNI实现**
Log的JNI的实现是在一个叫`android_util_Log.cpp`的源文件中。
源码路径：
头文件：`/frameworks/base/core/jni/android_util_Log.h`。
源文件：`/frameworks/base/core/jni/android_util_Log.cpp`。

在android_util_Log.cpp源文件中，我们可以找到println_native的身影。
```python
/*
 * JNI registration.
 */
static const JNINativeMethod gMethods[] = {
    /* name, signature, funcPtr */
    { "isLoggable",      "(Ljava/lang/String;I)Z", (void*) android_util_Log_isLoggable },
    { "println_native",  "(IILjava/lang/String;Ljava/lang/String;)I", (void*) android_util_Log_println_native },
    { "logger_entry_max_payload_native",  "()I", (void*) android_util_Log_logger_entry_max_payload_native },
};
```
JNINativeMethod是前面提到的一个结构体，这个结构体表示一个实现的本地方法。这个结构体在jni.h文件中定义，内容如下：
```python
typedef struct {
    const char* name;
    const char* signature;
    void*       fnPtr;
} JNINativeMethod;
```
它有三个指针变量，第一个是字符型指针，可以表示一个字符串，即native方法的名称；第二个也是字符型指针，同样可以表示一个字符串，代表这个native方法的参数和返回值（有特殊的表示方法）；第三个是一个未指定类型指针，表示一个函数指针，指向这个native方法对应的jni函数。

有了对JNINativeMethod了解，就可以理解`println_native`在`android_util_Log.cpp`源文件中的含义了。其对应jni实现函数是`android_util_Log_println_native()`。在jni实现函数中，又调用了`__android_log_buf_write()`这个方法，`__android_log_buf_write`是本地框架层（非Java框架层）基础的C库之上，Android最底层的本地Log库。
Log库的源码路径为：
头文件：`system/core/include/cutils/log.h`。
源文件：`system/core/liblog`。
编译后会生成liblog.so动态库和liblog.a静态库。

3. **JNI的注册**
```python
int register_android_util_Log(JNIEnv* env)
{
    jclass clazz = FindClassOrDie(env, "android/util/Log");

    levels.verbose = env->GetStaticIntField(clazz, GetStaticFieldIDOrDie(env, clazz, "VERBOSE", "I"));
    levels.debug = env->GetStaticIntField(clazz, GetStaticFieldIDOrDie(env, clazz, "DEBUG", "I"));
    levels.info = env->GetStaticIntField(clazz, GetStaticFieldIDOrDie(env, clazz, "INFO", "I"));
    levels.warn = env->GetStaticIntField(clazz, GetStaticFieldIDOrDie(env, clazz, "WARN", "I"));
    levels.error = env->GetStaticIntField(clazz, GetStaticFieldIDOrDie(env, clazz, "ERROR", "I"));
    levels.assert = env->GetStaticIntField(clazz, GetStaticFieldIDOrDie(env, clazz, "ASSERT", "I"));

    return RegisterMethodsOrDie(env, "android/util/Log", gMethods, NELEM(gMethods));
}
```
这是`android_util_Log.cpp`源文件中对jni的注册，可以看到RegisterMethodsOrDie()这个方法的调用，传入了我们前面看到的gMethods数组，进行JNI注册。到这里结束了吗？确实第一次看到这里的时候，以为就此结束了。然而，**是谁调用了register_android_util_Log()这个方法？在RegisterMethodsOrDie()这个函数里面又做了什么呢？**

4. **register_android_util_Log()函数的调用**
`register_android_util_Log()`这个方法只在`android_util_Log.cpp`源文件中进行定义，需要找到谁对它进行了调用，才好进一步理解Log的JNI的注册过程。Android源码环境有一个非常不错的方法，可以通过字符串，找到出现过的文件。
指令：`cgrep 'register_android_util_Log'`
类似于：`find . -type f -name "*.cpp" | xargs grep "register_android_util_Log"`
通过结果可以发现一个AndoridRuntime.cpp，这是个什么鬼？只能说它很强，是系统运行时的工具类，为Android的运行提供支持。JNI的部分封装也在这个类中。
源码路径：
`/frameworks/base/include/android_runtime/AndroidRuntime.h`。
`/frameworsk/base/core/jni/AndroidRuntime.cpp`。
可以发现，它也是在/frameworsk/base/core/jni目录下，说明也是在libandroid_runtime.so动态库中。

在AndroidRuntime.cpp源文件gRegJNI数组中，发现了`register_android_util_Log`方法。
```python
static const RegJNIRec gRegJNI[] = {
    REG_JNI(register_com_android_internal_os_RuntimeInit),
    REG_JNI(register_android_os_SystemClock),
    REG_JNI(register_android_util_EventLog),
    REG_JNI(register_android_util_Log),
    REG_JNI(register_android_util_MemoryIntArray),
    //省略
}
```
然后，又在AndroidRuntime.cpp源文件的**AndroidRuntime::startReg()**这个方法中，使用了gRegJNI这个数组。
```python
/*
 * Register android native functions with the VM.
 */
/*static*/ int AndroidRuntime::startReg(JNIEnv* env)
{
    ATRACE_NAME("RegisterAndroidNatives");
    /*
     * This hook causes all future threads created in this process to be
     * attached to the JavaVM.  (This needs to go away in favor of JNI
     * Attach calls.)
     */
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    ALOGV("--- registering native functions ---\n");

    /*
     * Every "register" function calls one or more things that return
     * a local reference (e.g. FindClass).  Because we haven't really
     * started the VM yet, they're all getting stored in the base frame
     * and never released.  Use Push/Pop to manage the storage.
     */
    env->PushLocalFrame(200);

    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);

    //createJavaThread("fubar", quickTest, (void*) "hello");

    return 0;
}
```
按源代码注释的意思是，这里在向虚拟机注册本地方法。同样在AndroidRuntime.cpp源文件中，AndroidRuntime::start()中又调用了AndroidRuntime::startReg()，到这里，需要继续往下看，就得找找**是谁调用了AndroidRuntime::start()**方法。然而，要想知道谁调用了它，已经涉及到zygote这一块的知识。

5. **zygote**
zygote是通过init进程读取init.rc启动的一个守护进程的名称。如果从最下面开始说，得再介绍一下Android启动流程的本地阶段，这一块，属于扩充了解。

Android启动流程的本地阶段：
+ BootLoader运行，Linux通用内容。
+ Linux内核运行，Linux通用内容，通常是二进制形式代码形式存在。
+ 内核加载根文件系统，Linux通用内容。
+ init进程运行，用户空间的第一个进程。
+ 运行init.rc脚本。
+ 加载system和date文件系统。
+ 运行各种服务，主要为各种守护进程。

本地部分启动完成，形成一系列守护进程，其中名称为zygote的守护进程，将继续完成Java部分的初始化。

Java部分的启动流程：
+ 从本地可执行程序运行名为zygote的守护进程。
+ zygote运行ZygoteInit（进入Java程序）。
+ ZygoteInit运行SystemServer（Java类），并分裂出新的进程。
+ SystemServer首先运行libandroid_servers.so库当中的初始化（进入本地程序）。
+ 执行libanroid_servers.so当中的系统初始化。
+ SystemServer中的Java初始化再次被调用（再入Java程序）。
+ 建立ServerThread线程。
+ ServerThread线程建立各个服务，然后进入循环。
+ ActivityManagerService在启动结束发送相关信息。
+ 各个Java应用程序运行。

这些是引用资料书中的知识，想详细了解，可以看看《Android核心原理及系统级应用高效开发》或《深入理解Android系统》。回到我们的zygote进程，init.rc中包含了一个init.${ro.zygote}.rc。
init.rc和init.${ro.zygote}.rc源码路径：`/system/core/rootdir`。

在init.zygote32.rc中，相关内容如下：
```python
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    writepid /dev/cpuset/foreground/tasks
```
这个是Android系统中的特殊语法，它启动了一个名称为zygote的进程，也就是`/system/bin/app_process`这个可执行程序。
源码路径为：`/frameworks/base/cmds/app_process`。
在这个目录下，有一个`app_main.cpp`的源文件，其中相关的代码如下：
```python
// 省略
if (zygote) {
    runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
} else if (className) {
    runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
} else {
    fprintf(stderr, "Error: no class name or --zygote supplied.\n");
    app_usage();
    LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    return 10;
}
```
在`app_main.cpp`源文件中，有一个AppRuntime的类，它继承了AndroidRuntime。runtime是AppRuntime类的一个实例，runtime.start()相当于调用了AndroidRuntime::start()这个方法，至此，前后就连接起来了。概括的说，系统启动zygote进程时，会调用AndroidRuntime::start()方法，接着调用AndroidRuntime::startReg()，然后调用到了`register_android_util_Log()`这个方法。剩下最后一个问题，`register_android_util_Log()`被调用后，在它方法体中的**RegisterMethodsOrDie()函数做了什么？**

6. **RegisterMethodsOrDie()函数**
RegisterMethodsOrDie()这个方法是`/frameworks/base/core/jni/core_jni_helpers.h`中声明的一个方法。
```python
static inline int RegisterMethodsOrDie(JNIEnv* env, const char* className,
                                       const JNINativeMethod* gMethods, int numMethods) {
    int res = AndroidRuntime::registerNativeMethods(env, className, gMethods, numMethods);
    LOG_ALWAYS_FATAL_IF(res < 0, "Unable to register native methods.");
    return res;
}
```
在其中我们可以看到，其实又回到了AndroidRuntime这个类，调用了它的registerNativeMethods()方法，并最终调用了jniRegisterNativeMethods()进行本地方法的注册。而jniRegisterNativeMethods()是/libnativehelper/JNIHelp.cpp源文件中的方法，它里面内容为：
```python
extern "C" int jniRegisterNativeMethods(C_JNIEnv* env, const char* className,
    const JNINativeMethod* gMethods, int numMethods)
{
    JNIEnv* e = reinterpret_cast<JNIEnv*>(env);

    ALOGV("Registering %s's %d native methods...", className, numMethods);

    scoped_local_ref<jclass> c(env, findClass(env, className));
    if (c.get() == NULL) {
        char* tmp;
        const char* msg;
        if (asprintf(&tmp,
                     "Native registration unable to find class '%s'; aborting...",
                     className) == -1) {
            // Allocation failed, print default warning.
            msg = "Native registration unable to find class; aborting...";
        } else {
            msg = tmp;
        }
        e->FatalError(msg);
    }

    if ((*env)->RegisterNatives(e, c.get(), gMethods, numMethods) < 0) {
        char* tmp;
        const char* msg;
        if (asprintf(&tmp, "RegisterNatives failed for '%s'; aborting...", className) == -1) {
            // Allocation failed, print default warning.
            msg = "RegisterNatives failed; aborting...";
        } else {
            msg = tmp;
        }
        e->FatalError(msg);
    }

    return 0;
}
```

这段代码可以看出，通过调用(*env)的RegisterNatives指针函数，进行了JNI注册。所以最后的动作是交给了JNINativeInterface结构体所表示的JNI环境执行。当在Java层调用native方法时，不需要依据native方法包和名称寻找对应的JNI函数。而是可以通过已经注册的映射关系，快速找到对应的JNI函数的指针，从而开始函数调用，大大提高执行效率。

