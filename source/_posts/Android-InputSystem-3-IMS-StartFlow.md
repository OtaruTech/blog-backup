---
layout: blog
title: 'Android Input子系统 -- InputManagerService启动'
date: 2019-12-17 21:25:26
categories:
- Android
- Input子系统
tags:
cover: /img/android-input.jpg
---
InputManagerService是Android framework中核心service之一，Android framework层涉及的代码也是非常多，

```c++
frameworks/native/services/inputflinger/
  - InputDispatcher.cpp
  - InputReader.cpp
  - InputManager.cpp
  - EventHub.cpp
  - InputListener.cpp

frameworks/native/libs/input/
  - InputTransport.cpp
  - Input.cpp
  - InputDevice.cpp
  - Keyboard.cpp
  - KeyCharacterMap.cpp
  - IInputFlinger.cpp

frameworks/base/services/core/
  - java/com/android/server/input/InputManagerService.java
  - jni/com_android_server_input_InputManagerService.cpp
```
<!--more-->

![](https://s2.ax1x.com/2020/03/10/8Col26.png)

从简单的分类可以看做是两层：

- Java层，`InputManagerService`负责对外提供服务，给`WindowManagerService`提供输入信息的回调
- Native层，监控Linux上报的输入事件，把事件处理成Android的KeyCode，给想要处理的Window发送输入事件

前面介绍了Linux的输入子系统，可以看到应用层的输入子系统框架都是基于驱动抽象出来的文件系统设备节点的读写来处理的，虽然说起来比较简单，但是在整个复杂的Android操作系统中，把输入事件发送到UI层处理还是非常复杂的一个过程。

当用户按下按键或者触摸屏幕时，输入系统会取出驱动上报的时间，经过层层封装转换成Android层能识别的`KeyEvent`或者`MotionEvent`，最后交付给对应的目标窗口来消费输入事件。

输入模块的组成：

- Native层的`InputReader`负责从EventHub取出事件并处理，再交付给`InputDispatcher`线程
- Native的`InputDispatcher`线程接收到来自`InputReader`的输入事件，并记录WMS的窗口信息，用来派发到合适的窗口
- Java层的`InputManagerService`跟WMS交互，WMS记录窗口信息，同步更新到IMS，为`InputDispatcher`线程正确派发事件到`ViewRootImpl`提供保障

#### InputManagerService启动过程

InputManagerService作为system_server中的重要服务，继承与`IInputManager.Stub`，作为binder的服务端，client位于InputManager的内部通过`IInputManagerStub.asInterface()`获取binder的代理端，C/S两端通信协议是由`IInputManager.aidl`来定义。

![](https://s2.ax1x.com/2020/03/10/8Co2Is.png)

IMS涉及到的重要的类：

- **`InputManagerService`** - 位于Java层的InputManagerService.java文件
  - 其成员变量`mPtr`指向Native层的`NativeInputManager`对象
- **`NativeInputManager`** - 位于Native层的com_android_server_input_InputManagerService.cpp文件
  - 其成员`mServiceObj`指向Java层的IMS对象
  - 其成员`mLooper`是指"android.display"线程的Looper
- **`InputManager`** - 位于libinputflinger中的InputManager.cpp文件
  - `InputDispatcher`和`InputReader`的成员变量`mPolicy`都是指向`NativeInputManager`对象
  - `InputReader`的成员`mQueuedListener`，数据类型为`QueuedInputListener`；通过其内部成员变量`mInnerListener`指向`InputDispatcher`对象；这就是`InputReader`跟`InputDispatcher`交互的中间枢纽。
- **`InputDispatcherPolicyInterface`**
- **`InputDispatcher`**
- **`InputDispatcherInterface`**
- **`InputReaderPolicyInterface`**
- **`InputReader`**
- **`InputReaderInterface`**
- **`EventHubInterface`**
- **`EventHub`**

##### IMS的启动

IMS是伴随着system_server进程的启动而启动的

```java
InputManagerService
	nativeInit
		NativeInputManager
			EventHub
			InputManager
				InputDispatcher
					Looper
				InputReader
					QueueInputListener
				InputReaderThread
				InputDispatcherThread
IMS.start
	nativeStart
		InputManager.start
			InputReaderThread->run
			InputDispatcherThread->run
```

SystemServer.java

```java
private void startOtherServices() {
    //初始化IMS对象
    inputManager = new InputManagerService(context);
    ServiceManager.addService(Context.INPUT_SERVICE, inputManager);
    ...
    //将InputMonitor对象保持到IMS对象
    inputManager.setWindowManagerCallbacks(wm.getInputMonitor());
    inputManager.start();
}
```

##### IMS初始化

InputManagerService.java

```java
public InputManagerService(Context context) {
   this.mContext = context;
   // 运行在线程"android.display"
   this.mHandler = new InputManagerHandler(DisplayThread.get().getLooper());
   ...
   //初始化native对象
   mPtr = nativeInit(this, mContext, mHandler.getLooper().getQueue());
   LocalServices.addService(InputManagerInternal.class, new LocalService());
}
```

##### nativeInit

调用到`InputManagerService`对应的JNI代码com_android_server_input_InputManager.cpp

```c++
static jlong nativeInit(JNIEnv* env, jclass /* clazz */, jobject serviceObj, jobject contextObj, jobject messageQueueObj) {
    //获取native消息队列
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    ...
    //创建Native的InputManager
    NativeInputManager* im = new NativeInputManager(contextObj, serviceObj,
            messageQueue->getLooper());
    im->incStrong(0);
    return reinterpret_cast<jlong>(im); //返回Native对象的指针
}
```

同样的套路，在`nativeInit`函数中创建了一个`NativeInputManager`对象，然后转化为指针保存在一个`long`类型的变量中传递给java层，这样子java层就可以利用这个指针来调用`NativeInputManager`对象中的成员方法和变量。

##### NativeInputManager

com_android_server_input_InputManagerService.cpp

```c++
NativeInputManager::NativeInputManager(jobject contextObj,
        jobject serviceObj, const sp<Looper>& looper) :
        mLooper(looper), mInteractive(true) {
    JNIEnv* env = jniEnv();
    mContextObj = env->NewGlobalRef(contextObj); //上层IMS的context
    mServiceObj = env->NewGlobalRef(serviceObj); //上层IMS对象
    ...
    sp<EventHub> eventHub = new EventHub(); // 创建EventHub对象
    mInputManager = new InputManager(eventHub, this, this); // 创建InputManager对象
}
```

`mLooper`是从IMS的"android.display"线程的Looper对象传递下来的。

##### EventHub

EventHub.cpp

```c++
EventHub::EventHub(void) :
        mBuiltInKeyboardId(NO_BUILT_IN_KEYBOARD), mNextDeviceId(1), mControllerNumbers(),
        mOpeningDevices(0), mClosingDevices(0),
        mNeedToSendFinishedDeviceScan(false),
        mNeedToReopenDevices(false), mNeedToScanDevices(true),
        mPendingEventCount(0), mPendingEventIndex(0), mPendingINotify(false) {
    acquire_wake_lock(PARTIAL_WAKE_LOCK, WAKE_LOCK_ID);
    //创建epoll
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);

    mINotifyFd = inotify_init();
    //此处DEVICE_PATH为"/dev/input"，监听该设备路径
    int result = inotify_add_watch(mINotifyFd, DEVICE_PATH, IN_DELETE | IN_CREATE);

    struct epoll_event eventItem;
    memset(&eventItem, 0, sizeof(eventItem));
    eventItem.events = EPOLLIN;
    eventItem.data.u32 = EPOLL_ID_INOTIFY;
    //添加INotify到epoll实例
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mINotifyFd, &eventItem);

    int wakeFds[2];
    result = pipe(wakeFds); //创建管道

    mWakeReadPipeFd = wakeFds[0];
    mWakeWritePipeFd = wakeFds[1];

    //将pipe的读和写都设置为非阻塞方式
    result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);
    result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);

    eventItem.data.u32 = EPOLL_ID_WAKE;
    //添加管道的读端到epoll实例
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, &eventItem);
    ...
}
```

- 创建epoll实例来处理文件IO的多路复用
- 初始化INotify，监听/dev/input/目录下的文件，并且添加到epoll中
- 创建非阻塞管道文件描述符，添加到epoll中

##### InputManager

InputManager.cpp

```c++
InputManager::InputManager(
        const sp<EventHubInterface>& eventHub,
        const sp<InputReaderPolicyInterface>& readerPolicy,
        const sp<InputDispatcherPolicyInterface>& dispatcherPolicy) {
    //创建InputDispatcher对象
    mDispatcher = new InputDispatcher(dispatcherPolicy);
    //创建InputReader对象
    mReader = new InputReader(eventHub, readerPolicy, mDispatcher);
    initialize();
}
```

创建了`InputDispatcher`和`InputReader`对象，这里的`Reader`和`Dispatcher`的构造参数都是从`NativeInputManager`传递过来的

##### InputManager初始化initialize

InputManager.cpp

```c++
void InputManager::initialize() {
    //创建线程“InputReader”
    mReaderThread = new InputReaderThread(mReader);
    //创建线程”InputDispatcher“
    mDispatcherThread = new InputDispatcherThread(mDispatcher);
}

InputReaderThread::InputReaderThread(const sp<InputReaderInterface>& reader) :
        Thread(/*canCallJava*/ true), mReader(reader) {
}

InputDispatcherThread::InputDispatcherThread(const sp<InputDispatcherInterface>& dispatcher) :
        Thread(/*canCallJava*/ true), mDispatcher(dispatcher) {
}
```

创建两个可以访问`InputManagerService`成员变量的native线程

- `InputReader`线程
- `InputDispatcher`线程

##### IMS.start

InputManagerService.java

```c++
public void start() {
    // 启动native对象
    nativeStart(mPtr);
    Watchdog.getInstance().addMonitor(this);
    //注册触摸点速度和是否显示功能的观察者
    registerPointerSpeedSettingObserver();
    registerShowTouchesSettingObserver();

    mContext.registerReceiver(new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            updatePointerSpeedFromSettings();
            updateShowTouchesFromSettings();
        }
    }, new IntentFilter(Intent.ACTION_USER_SWITCHED), null, mHandler);

    updatePointerSpeedFromSettings(); //更新触摸点的速度
    updateShowTouchesFromSettings(); //是否在屏幕上显示触摸点
}
```

##### nativeStart

com_android_server_input_InputManagerService.cpp

```c++
static void nativeStart(JNIEnv* env, jclass /* clazz */, jlong ptr) {
    //此处ptr记录的便是NativeInputManager
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);
    //
    status_t result = im->getInputManager()->start();
    ...
}
```

##### InputManager.start

InputManager.cpp

```c++
status_t InputManager::start() {
    result = mDispatcherThread->run("InputDispatcher", PRIORITY_URGENT_DISPLAY);
    result = mReaderThread->run("InputReader", PRIORITY_URGENT_DISPLAY);
    ...
    return OK;
}
```

启动两个线程：

- InputReader
- InputDispatcher

```c++
/* Reads raw events from the event hub and processes them, endlessly. */
class InputReaderThread : public Thread {
public:
    explicit InputReaderThread(const sp<InputReaderInterface>& reader);
    virtual ~InputReaderThread();

private:
    sp<InputReaderInterface> mReader;

    virtual bool threadLoop();
};
```

`InputReaderThread`和`InputDispatcherThread`都是集成了Android::Thread，实现了`threadLoop()`接口实现线程的实体。

下面来详细分析一下Android的EventHub。