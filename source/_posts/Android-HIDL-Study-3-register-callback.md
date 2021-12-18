---
title: 'Android HIDL学习（3） ---- 注册回调'
date: 2019-12-15 21:03:17
categories:
- Android
- HIDL
tags:
---

#### **回顾一下**

上一节我们学会了如何创建HIDL的server端和client端，对于那些没玩过Android O或者以上的BSP开发者而言，可以吹上一阵子牛逼了，毕竟比人家多了一个技能，面试的时候也可以装一下了^_^

OK，我们还知道了在Android O或者以上的Android版本上创建一个HAL模块的一般流程是如何的，我们这一节来看一个比较简单的东西，也是每个模块基本必不可少的一个玩意儿，那就是回调函数。

<!--more-->

#### **注册回调**

怎么个回事呢，我们来举一个栗子

![](https://s2.ax1x.com/2020/03/10/8ChDHA.gif)

我们把HAL独立为一个单独的进程，client也是一个单独的进程，那么对于一般的模块而言，都是需要从底层（HAL以及以下）获取数据，比如sensor，需要获取sensor数据，Camera，需要获取camera的raw、yuv等数据流，那么对于软件设计而言，如果是同步的话，很简单，我们通过getXXX()函数来获取即可，但是如果是异步的，比如底层的实现是中端的机制，你不知道他什么时候会出来数据，那么这个时候通常的，我们会通过callback来实现**异步的回调**。

看下面的图就比较清楚了

![](https://s2.ax1x.com/2020/03/10/8C4pU1.jpg)

我们这一节就来实现简单的回调机制。

#### 实战演练

这个例子很简单，写一个简单的HAL模块，就跟之前的差不多，然后我们在**.hal**文件里面加入一个**setCallback**函数，传入一个callback指针，当我们HAL的server端起来的时候会起一个线程，每隔5秒钟时间调用一下传入的这个回调函数，实现回调的机制，OK，废话不多说，上代码。

看一下HIDL 接口IHello.hal

```c++
package vendor.sample.hello@1.0;

import IHelloCallback;

interface IHello {
    init();
    release();
    setCallback(IHelloCallback callback);
};
```

定义了三个接口

*   init：做一些初始化的动作

*   release：做一些释放的动作

*   setCallback：让client端设置一个callback方法到server端

下面来看看这个callback里面都定义了些啥，我们要为这个callback些一个接口IHelloCallback.hal

```c++
package vendor.sample.hello@1.0;
​
interface IHelloCallback {
 oneway onNotify(HalEvent event);
};
```

回调函数里面有一个回调方法，可以让server传一个HalEvent的结构体到client端，这个结构体也是自定义的，在types.hal，可以定义自己喜欢的类型，这里是一个简单的int成员变量

```c++
package vendor.sample.hello@1.0;
​
struct HalEvent {
 int32_t value;
};
```

OK，HIDL的接口定义好之后，我们来使用一条牛逼的指令为我们生产代码框架：
```shell
hidl-gen -o vendor/honeywell/common/sample/hidl-impl/sample/ -Lc++-impl -rvendor.sample:vendor/honeywell/common/sample/interfaces -randroid.hidl:system/libhidl/transport vendor.sample.hello@1.0
```
生成了一坨代码：

├── Android.mk  
├── hidl-impl  
│ ├── Android.mk  
│ └── sample  ***│ 
├── Android.bp***  ***
│ ├── HelloCallback.cpp***  ***
│ ├── HelloCallback.h***  ***
│ ├── Hello.cpp***  ***
│ └── Hello.h***
└── interfaces
  ​ ├── Android.bp
  ​   └── hello
  ​     └── 1.0
  ​       ├── Android.bp
        ​ ├── IHelloCallback.hal
        ​ ├── IHello.hal
  ​       └── types.hal

其中有一个代码是用不到的，HelloCallback.h和HelloCallback.cpp，也不知道为什么指令会为我们创建出来，肯定是个bug。删掉他们。

好了接下来就是写代码了，注意，要把hidl-impl/sample/Android.bp里面的HelloCallback.cpp也要删掉

```shell
cc_library_shared {
 proprietary: true,
 srcs: [
 "Hello.cpp",
 ],
 shared_libs: [
 "libhidlbase",
 "libhidltransport",
 "libutils",
 "vendor.sample.hello@1.0",
 ],
}
```

就变成这个样子了。在vendor分区，要起一个service来handle这个HIDL 接口，这个我们在上一节中有详细讲到，贴一下代码：

```c++
#include <vendor/sample/hello/1.0/IHello.h>
​
#include <hidl/LegacySupport.h>
​
using vendor::sample::hello::V1_0::IHello;
using android::hardware::defaultPassthroughServiceImplementation;
​
int main()
{
 return defaultPassthroughServiceImplementation<IHello>();
}
```
然后是makefile：

```shell
cc_binary {
 name: "vendor.sample.hello@1.0-service",
 relative_install_path: "hw",
 defaults: ["hidl_defaults"],
 vendor: true,
 init_rc: ["vendor.sample.hello@1.0-service.rc"],
 srcs: [
 "service.cpp",
 ],
 shared_libs: [
 "liblog",
 "libutils",
 "libhidlbase",
 "libhidltransport",
 "libutils",
 "vendor.sample.hello@1.0",
 ],
}
```

然后编译一把，应该就能看到生产impl的库和一个可执行程序用来起server的。

看一下下面的代码，是主体实现端的代码。

```c++
#define LOG_TAG     "Sample"
​
#include "Hello.h"
#include <log/log.h>
​
namespace vendor {
namespace sample {
namespace hello {
namespace V1_0 {
namespace implementation {
​
sp<IHelloCallback> Hello::mCallback = nullptr;
​
// Methods from ::vendor::sample::hello::V1_0::IHello follow.
Return<void> Hello::init() {
 mExit = false;
 run("sample");
 return Void();
}
​
Return<void> Hello::release() {
 mExit = true;
 return Void();
}
​
Return<void> Hello::setCallback(const sp<::vendor::sample::hello::V1_0::IHelloCallback>& callback) {
 mCallback = callback;
 if(mCallback != nullptr) {
 ALOGD("setCallback: done");
 }
​
return Void();
​
}
​
bool Hello::threadLoop()
{
 static int32_t count = 0;
 HalEvent event;
 while(!mExit) {
 ::sleep(1);
 event.value = count ++;
 if(mCallback != nullptr) {
 mCallback->onNotify(event);
 }
 }
 ALOGD("threadLoop: exit");
 return false;
}
​
// Methods from ::android::hidl::base::V1_0::IBase follow.
​
IHello* HIDL_FETCH_IHello(const char* /* name */) {
 return new Hello();
}
//
}  // namespace implementation
}  // namespace V1_0
}  // namespace hello
}  // namespace sample
}  // namespace vendor
```

在init函数里面调用run方法去启动线程，线程的主体是threadLoop函数，可以看到在线程里面，是一个死循环，会每隔1秒钟去callback一次方法，还是很简单的。

下面是client的实现，

```c++
#define LOG_TAG     "TestHello"
​
#include <log/log.h>
#include <vendor/sample/hello/1.0/types.h>
#include <vendor/sample/hello/1.0/IHello.h>
#include <vendor/sample/hello/1.0/IHelloCallback.h>
#include <hidl/Status.h>
#include <hidl/HidlSupport.h>
​
using android::sp;
using android::hardware::Return;
using android::hardware::Void;
​
using vendor::sample::hello::V1_0::HalEvent;
using vendor::sample::hello::V1_0::IHello;
using vendor::sample::hello::V1_0::IHelloCallback;
​
class HelloCallback: public IHelloCallback {
public:
 HelloCallback() {

}
​
~HelloCallback() {

}
​
Return<void> onNotify(const HalEvent& event) {
​
 ALOGD("onNotify: value = %d", event.value);
​
 return Void();
}
​
};
​
int main(void)
{
 sp<IHello> service = IHello::getService();
 if(service == nullptr) {
 ALOGE("main: failed to get hello service");
 return -1;
 }
​
sp<HelloCallback> callback = new HelloCallback();
service->setCallback(callback);
service->init();
​
::sleep(10);
service->release();
​
return 0;
​
}
```

OK，在client端就是简单的打印了callback回来的event里面的数据。

下面是makefile的代码:

```shell
cc_binary {
 name: "test_hello",
 srcs: [
 "test_hello.cpp",
 ],
 shared_libs: [
 "liblog",
 "libutils",
 "libhidlbase",
 "libhidltransport",
 "libutils",
 "vendor.sample.hello@1.0",
 ],
}
```

现在可以手动运行测试程序了，还是跟上一节介绍的一样，可以看到logcat有如下输出：

![](https://s2.ax1x.com/2020/03/10/8C48Kg.png)


OK，达到了我们的预期效果了。

这一节我们就简单的介绍了在Android O/P里面使用HIDL的回调机制是如何实现的。
