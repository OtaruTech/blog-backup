---
layout: blog
title: 'Android Input子系统 -- EventHub'
date: 2019-12-18 10:49:07
categories:
- Android
- Input子系统
tags:
cover: /img/android-input.jpg
---
#### 前言

前面其实也有提到EventHub的构造函数，里面就是创建epoll实例，然后把一些事件触发的文件描述符加入到epoll里面统一管理。

- 监控`/dev/input/`目录的iNotify文件mINotifyFd
- 接收Kernel驱动事件(`/dev/input/eventX`)的文件描述符
- 用来唤醒`InputReader`线程的管道读文件
<!--more-->

![](https://s2.ax1x.com/2020/03/10/8CoHZ4.png)



`EventHub`是服务于`InputReader`线程的，前面在InputRead的构造函数里面有创建EventHub的实例。

##### InputReader线程

`InputReader`线程主要就是查看`threadLoop`接口中代码的实现，`threadLoop`返回值是true代表的是会不断的循环，返回false的话就不会继续在调用`threadLoop`函数。

```c++
bool InputReaderThread::threadLoop() {
    mReader->loopOnce();
    return true;
}
```

不停的调用`InputReader`的`loopOnce`函数

##### loopOnce函数

InputReader.cpp

```c++
void InputReader::loopOnce() {
    ...
    {
        AutoMutex _l(mLock);
        uint32_t changes = mConfigurationChangesToRefresh;
        if (changes) {
            timeoutMillis = 0;
            ...
        } else if (mNextTimeout != LLONG_MAX) {
            nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
            timeoutMillis = toMillisecondTimeoutDelay(now, mNextTimeout);
        }
    }

    //从EventHub读取事件，其中EVENT_BUFFER_SIZE = 256
    size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);

    { // acquire lock
        AutoMutex _l(mLock);
         mReaderIsAliveCondition.broadcast();
        if (count) { //处理事件
            processEventsLocked(mEventBuffer, count);
        }
        if (oldGeneration != mGeneration) {
            inputDevicesChanged = true;
            getInputDevicesLocked(inputDevices);
        }
        ...
    } // release lock


    if (inputDevicesChanged) { //输入设备发生改变
        mPolicy->notifyInputDevicesChanged(inputDevices);
    }
    //发送事件到nputDispatcher
    mQueuedListener->flush();
}
```

两个关键的地方，从注释可以很明确的看出来，首先是获取到底层的input事件，处理事件，然后发送到`InputDispatcher`线程

- `mEventHub->getEvents`
- `processEventsLocked(mEventBuffer, count)`
- `mQueuedListener->flush`

##### EventHub->getEvents

EventHub.cpp

```c++
size_t EventHub::getEvents(int timeoutMillis, RawEvent* buffer, size_t bufferSize) {
    AutoMutex _l(mLock); //加锁

    struct input_event readBuffer[bufferSize];
    RawEvent* event = buffer; //原始事件
    size_t capacity = bufferSize; //容量大小为256
    bool awoken = false;
    for (;;) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        ...

        if (mNeedToScanDevices) {
            mNeedToScanDevices = false;
            scanDevicesLocked(); //扫描设备
            mNeedToSendFinishedDeviceScan = true;
        }

        while (mOpeningDevices != NULL) {
            Device* device = mOpeningDevices;
            mOpeningDevices = device->next;
            event->when = now;
            event->deviceId = device->id == mBuiltInKeyboardId ? 0 : device->id;
            event->type = DEVICE_ADDED; //添加设备的事件
            event += 1;
            mNeedToSendFinishedDeviceScan = true;
            if (--capacity == 0) {
                break;
            }
        }
        ...

        bool deviceChanged = false;
        while (mPendingEventIndex < mPendingEventCount) {
            //从mPendingEventItems读取事件项
            const struct epoll_event& eventItem = mPendingEventItems[mPendingEventIndex++];
            ...
            //获取设备ID所对应的device
            ssize_t deviceIndex = mDevices.indexOfKey(eventItem.data.u32);
            Device* device = mDevices.valueAt(deviceIndex);
            if (eventItem.events & EPOLLIN) {
                //从设备不断读取事件，放入到readBuffer
                int32_t readSize = read(device->fd, readBuffer,
                        sizeof(struct input_event) * capacity);

                if (readSize == 0 || (readSize < 0 && errno == ENODEV)) {
                    deviceChanged = true;
                    closeDeviceLocked(device);//设备已被移除则执行关闭操作
                } else if (readSize < 0) {
                    ...
                } else if ((readSize % sizeof(struct input_event)) != 0) {
                    ...
                } else {
                    int32_t deviceId = device->id == mBuiltInKeyboardId ? 0 : device->id;
                    size_t count = size_t(readSize) / sizeof(struct input_event);

                    for (size_t i = 0; i < count; i++) {
                        //获取readBuffer的数据
                        struct input_event& iev = readBuffer[i];
                        //将input_event信息, 封装成RawEvent
                        event->when = nsecs_t(iev.time.tv_sec) * 1000000000LL
                                + nsecs_t(iev.time.tv_usec) * 1000LL;
                        event->deviceId = deviceId;
                        event->type = iev.type;
                        event->code = iev.code;
                        event->value = iev.value;
                        event += 1;
                        capacity -= 1;
                    }
                    if (capacity == 0) {
                        mPendingEventIndex -= 1;
                        break;
                    }
                }
            }
            ...
        }
        ...
        mLock.unlock(); //poll之前先释放锁
        //等待input事件的到来
        int pollResult = epoll_wait(mEpollFd, mPendingEventItems, EPOLL_MAX_EVENTS, timeoutMillis);
        ...
        mLock.lock(); //poll之后再次请求锁

        if (pollResult < 0) { //出现错误
            mPendingEventCount = 0;
            if (errno != EINTR) {
                usleep(100000); //系统发生错误则休眠1s
            }
        } else {
            mPendingEventCount = size_t(pollResult);
        }
    }

    return event - buffer; //返回所读取的事件个数
}
```

虽然精简了代码还是比较长，主要是以下几点：

- EventHub是采用了INotify和epoll机制监听目录`/dev/input`下设备节点，当有IO事件，会通过epoll_wait返回事件的详细信息
- 当epoll_wait被返回有IO数据的时候，通过read函数读取input事件的数据
- 把input_event结构体转化为RawEvent事件

InputEventReader.h

```c++
struct input_event {
 struct timeval time; //事件发生的时间点
 __u16 type;
 __u16 code;
 __s32 value;
};

struct RawEvent {
    nsecs_t when; //事件发生的时间店
    int32_t deviceId; //产生事件的设备Id
    int32_t type; // 事件类型
    int32_t code;
    int32_t value;
};
```

事件类型：

1. DEVICE_ADD
2. DEVICE_REMOVED
3. FINISHED_DEVICE_SCAN
4. type<FIRST_SYNTHETIC_EVENT>

##### InputReader->processEventsLocked

InputReader.cpp

```c++
void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
    for (const RawEvent* rawEvent = rawEvents; count;) {
        int32_t type = rawEvent->type;
        size_t batchSize = 1;
        if (type < EventHubInterface::FIRST_SYNTHETIC_EVENT) {
            int32_t deviceId = rawEvent->deviceId;
            while (batchSize < count) {
                if (rawEvent[batchSize].type >= EventHubInterface::FIRST_SYNTHETIC_EVENT
                        || rawEvent[batchSize].deviceId != deviceId) {
                    break;
                }
                batchSize += 1; //同一设备的事件打包处理
            }
            //数据事件的处理
            processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
        } else {
            switch (rawEvent->type) {
            case EventHubInterface::DEVICE_ADDED:
                //设备添加
                addDeviceLocked(rawEvent->when, rawEvent->deviceId);
                break;
            case EventHubInterface::DEVICE_REMOVED:
                //设备移除
                removeDeviceLocked(rawEvent->when, rawEvent->deviceId);
                break;
            case EventHubInterface::FINISHED_DEVICE_SCAN:
                //设备扫描完成
                handleConfigurationChangedLocked(rawEvent->when);
                break;
            default:
                ALOG_ASSERT(false);//不会发生
                break;
            }
        }
        count -= batchSize;
        rawEvent += batchSize;
    }
}
```

主要是根据不同的事件类型做不同的处理，我们比较关注对数据的处理，主要是看这个函数`processEventsForDeviceLocked`

##### 事件处理 processEventsForDeviceLocked

```c++
void InputReader::processEventsForDeviceLocked(int32_t deviceId,
        const RawEvent* rawEvents, size_t count) {
    ssize_t deviceIndex = mDevices.indexOfKey(deviceId);
    ...

    InputDevice* device = mDevices.valueAt(deviceIndex);
    if (device->isIgnored()) {
        return; //可忽略则直接返回
    }
    //调用对应设备的process函数
    device->process(rawEvents, count);
}
```

通过`deviceId`来获取合适的`InputDevice`，`InputDevice`是当检测到`/dev/input/`下有设备节点添加的时候创建的

调用对应的`device`的process函数

```c++
void InputDevice::process(const RawEvent* rawEvents, size_t count) {
    size_t numMappers = mMappers.size();
    for (const RawEvent* rawEvent = rawEvents; count--; rawEvent++) {
        if (mDropUntilNextSync) {
            if (rawEvent->type == EV_SYN && rawEvent->code == SYN_REPORT) {
                mDropUntilNextSync = false;
            }
        } else if (rawEvent->type == EV_SYN && rawEvent->code == SYN_DROPPED) {
            mDropUntilNextSync = true;
            reset(rawEvent->when);
        } else {
            for (size_t i = 0; i < numMappers; i++) {
                InputMapper* mapper = mMappers[i];
                //调用具体mapper来处理
                mapper->process(rawEvent);
            }
        }
    }
}
```

这里处理输入事件主要是把Linux下发送过来的RawEvent转化为Android 的Key Event和Key Code，在Android中不同的输入设备对应不同的Key Layout文件，这里就是选择对应的kl文件来处理。

下面专门针对按键的事件处理跟一下流程。

##### 按键事件处理流程

InputReader.cpp

KeyBoardInputMapper->process

```c++
void KeyboardInputMapper::process(const RawEvent* rawEvent) {
    switch (rawEvent->type) {
    case EV_KEY: {
        int32_t scanCode = rawEvent->code;
        int32_t usageCode = mCurrentHidUsage;
        mCurrentHidUsage = 0;

        if (isKeyboardOrGamepadKey(scanCode)) {
            int32_t keyCode;
            //获取所对应的KeyCode
            if (getEventHub()->mapKey(getDeviceId(), scanCode, usageCode, &keyCode, &flags)) {
                keyCode = AKEYCODE_UNKNOWN;
                flags = 0;
            }
            //
            processKey(rawEvent->when, rawEvent->value != 0, keyCode, scanCode, flags);
        }
        break;
    }
    case EV_MSC: ...
    case EV_SYN: ...
    }
}
```

通过mapKey函数获取对应scanCode的Android Key Code

EventHub.cpp

```c++
status_t EventHub::mapKey(int32_t deviceId,
        int32_t scanCode, int32_t usageCode, int32_t metaState,
        int32_t* outKeycode, int32_t* outMetaState, uint32_t* outFlags) const {
    AutoMutex _l(mLock);
    Device* device = getDeviceLocked(deviceId); //获取设备对象
    status_t status = NAME_NOT_FOUND;

    if (device) {
        sp<KeyCharacterMap> kcm = device->getKeyCharacterMap();
        if (kcm != NULL) {
            //根据scanCode找到keyCode
            if (!kcm->mapKey(scanCode, usageCode, outKeycode)) {
                *outFlags = 0;
                status = NO_ERROR;
            }
        }
    }
    ...
    return status;
}
```

真正干活做转换的是`KeyCharacterMap::mapKey`函数。

InputReader.cpp

InputMapper->processKey

```c++
void KeyboardInputMapper::processKey(nsecs_t when, bool down, int32_t keyCode,
        int32_t scanCode, uint32_t policyFlags) {
    if (down) {
        if (mParameters.orientationAware && mParameters.hasAssociatedDisplay) {
            keyCode = rotateKeyCode(keyCode, mOrientation);
        }

        ssize_t keyDownIndex = findKeyDown(scanCode);
        if (keyDownIndex >= 0) {
            //mKeyDowns记录着所有按下的键
            keyCode = mKeyDowns.itemAt(keyDownIndex).keyCode;
        } else {
            ...
            mKeyDowns.push(); //压入栈顶
            KeyDown& keyDown = mKeyDowns.editTop();
            keyDown.keyCode = keyCode;
            keyDown.scanCode = scanCode;
        }
        mDownTime = when; //记录按下时间点
    } else {
        ssize_t keyDownIndex = findKeyDown(scanCode);
        if (keyDownIndex >= 0) {
            //键抬起操作，则移除按下事件
            keyCode = mKeyDowns.itemAt(keyDownIndex).keyCode;
            mKeyDowns.removeAt(size_t(keyDownIndex));
        } else {
            return;  //键盘没有按下操作，则直接忽略抬起操作
        }
    }
    nsecs_t downTime = mDownTime;
    ...

    //创建NotifyKeyArgs对象, when记录eventTime, downTime记录按下时间；
    NotifyKeyArgs args(when, getDeviceId(), mSource, policyFlags,
            down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP,
            AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, newMetaState, downTime);
    //通知key事件
    getListener()->notifyKey(&args);
}
```

- `mKeyDowns`记录着所有按下的键
- `mDownTime`记录按下时间点
- `KeyboardInputMapper`的`mContext`指向`InputReader`，`getListener()`获取`mQueuedListener`，然后调用`notifyKey`接口

InputListener.cpp

```c++
void QueuedInputListener::notifyKey(const NotifyKeyArgs* args) {
    mArgsQueue.push(new NotifyKeyArgs(*args));
}
```

把事件放入向量`mArgsQueue`，然后就是把事件发送给InputDispatcher线程了

##### 发送事件 QueuedInputListener->flush

InputListener.cpp

```c++
void QueuedInputListener::flush() {
    size_t count = mArgsQueue.size();
    for (size_t i = 0; i < count; i++) {
        NotifyArgs* args = mArgsQueue[i];
        //
        args->notify(mInnerListener);
        delete args;
    }
    mArgsQueue.clear();
}
```

遍历刚才的向量，调用对应的notify接口，`NotifyArgs`的实现子类包含：

- NotifyConfigurationChangedArgs
- NotifyKeyArgs
- NotifyMotionArgs
- NotifySwitchArgs
- NotifyDeviceResetArgs

```c++
void NotifyKeyArgs::notify(const sp<InputListenerInterface>& listener) const {
    listener->notifyKey(this); // this是指NotifyKeyArgs
}
```

InputDispatcher.cpp

```c++
void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {
    if (!validateKeyEvent(args->action)) {
        return;
    }
    ...
    int32_t keyCode = args->keyCode;

    if (keyCode == AKEYCODE_HOME) {
        if (args->action == AKEY_EVENT_ACTION_DOWN) {
            property_set("sys.domekey.down", "1");
        } else if (args->action == AKEY_EVENT_ACTION_UP) {
            property_set("sys.domekey.down", "0");
        }
    }

    if (metaState & AMETA_META_ON && args->action == AKEY_EVENT_ACTION_DOWN) {
        ...
    } else if (args->action == AKEY_EVENT_ACTION_UP) {
        ...
    }

    KeyEvent event; //初始化KeyEvent对象
    event.initialize(args->deviceId, args->source, args->action,
            flags, keyCode, args->scanCode, metaState, 0,
            args->downTime, args->eventTime);
    //mPolicy是指NativeInputManager对象。
    mPolicy->interceptKeyBeforeQueueing(&event, /*byref*/ policyFlags);

    bool needWake;
    {
        mLock.lock();
        if (shouldSendKeyToInputFilterLocked(args)) {
            mLock.unlock();
            policyFlags |= POLICY_FLAG_FILTERED;
            //当inputEventObj不为空, 则事件被filter所拦截
            if (!mPolicy->filterInputEvent(&event, policyFlags)) {
                return;
            }
            mLock.lock();
        }

        int32_t repeatCount = 0;
        //创建KeyEntry对象
        KeyEntry* newEntry = new KeyEntry(args->eventTime,
                args->deviceId, args->source, policyFlags,
                args->action, flags, keyCode, args->scanCode,
                metaState, repeatCount, args->downTime);
        //将KeyEntry放入队列
        needWake = enqueueInboundEventLocked(newEntry);
        mLock.unlock();
    }

    if (needWake) {
        //唤醒InputDispatcher线程
        mLooper->wake();
    }
}
```

主要完成以下几个事情：

- 调用`NativeInputManager.interceptKeyBeforeQueueing`，加入队列前执行拦截动作，但并不改变流程
  - ​	`IMS.interceptKeyBeforeQueueing`
  - ​	`InputMonitor.interceptKeyBeforeQueueing`（继承`IMS.WindowManagerCallbacks`）
  - ​	`PhoneWindowManager.interceptKeyBeforeQueueing`（继承`WindowManagerPolicy`）
- 当`mInputFilterEnabled=true`（该值默认为`false`，可通过`setInputFilterEnabled`设置），则调用`NativeInputManager.filterInputEvent`过滤输入事件
  - ​	当返回值为`false`则过滤事件，不往下发
- 生成`KeyEvent`，并调用`enqueueInboundEventLocked`，把事件加入到`InputDispatchered`的成员变量`mInboundQueue`。
- 唤醒`InputDispatcher`线程

```c++
void Looper::wake() {
    uint64_t inc = 1;

    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}
```

`InputDispatcher`线程主要是用来跟JAVA层的IMS和WMS通信的，给JAVA层IMS上报输入事件，找到对应的Window。