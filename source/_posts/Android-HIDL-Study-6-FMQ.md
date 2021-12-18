---
title: 'Android HIDL学习（6）---Fast Message Queue'
date: 2019-12-15 21:15:57
categories:
- Android
- HIDL
tags:
---
想聊聊FMQ的，无意中看到下面这篇文章，写的很好，所以就直接拿来用了，笑纳笑纳~

<!--more-->

[https://www.jianshu.com/p/5c6e35c7c346](https://www.jianshu.com/p/5c6e35c7c346)

## 快速消息队列 (FMQ)

HIDL 的远程过程调用 (RPC) 基础架构使用 Binder 机制，这意味着调用涉及开销、需要内核操作，并且可以触发调度程序操作。

不过，对于必须在开销较小且无内核参与的进程之间传输数据的情况，则使用快速消息队列 (FMQ) 系统。

FMQ 会创建具有所需属性的消息队列。MQDescriptorSync 或 MQDescriptorUnsync 对象可通过 HIDL RPC 调用发送，并可供接收进程用于访问消息队列。

## MessageQueue 类型

Android 支持两种队列类型（称为“风格”）：

*   **未同步队列**：
    可以溢出，并且可以有多个读取器；每个读取器都必须及时读取数据，否则数据将会丢失。
*   **已同步队列**：
    不能溢出，并且只能有一个读取器。

这两种队列都不能下溢（从空队列进行读取将会失败），并且只能有一个写入器。

#### 未同步

未同步队列只有一个写入器，但可以有任意多个读取器。此类队列有*一个写入位置*；不过，每个读取器都会跟踪各自的独立读取位置。

对此类队列执行写入操作一定会成功（不会检查是否出现溢出情况），但前提是写入的内容不超出配置的队列容量（如果写入的内容超出队列容量，则操作会立即失败）。
由于各个读取器的读取位置可能不同，因此每当新的写入操作需要空间时，系统都允许数据离开队列，而无需等待每个读取器读取每条数据。

读取操作负责在数据离开队列末尾之前对其进行检索。如果读取操作尝试读取的数据超出可用数据量，则该操作要么立即失败（如果非阻塞），要么等到有足够多的可用数据时（如果阻塞）。如果读取操作尝试读取的数据超出队列容量，则读取一定会立即失败。

如果某个读取器的读取速度无法跟上写入器的写入速度，则写入的数据量和该读取器尚未读取的数据量加在一起会超出队列容量，这会导致下一次读取不会返回数据；相反，该读取操作会将读取器的读取位置重置为等于最新的写入位置，然后返回失败。如果在发生溢出后但在下一次读取之前，系统查看可供读取的数据，则会显示可供读取的数据超出了队列容量，这表示发生了溢出。（如果队列溢出发生在系统查看可用数据和尝试读取这些数据之间，则溢出的唯一表征就是读取操作失败。）

#### 已同步

已同步队列有一个写入器和一个读取器，其中写入器有一个写入位置，读取器有一个读取位置。写入的数据量不可能超出队列可提供的空间；读取的数据量不可能超出队列当前存在的数据量。如果尝试写入的数据量超出可用空间或尝试读取的数据量超出现有数据量，则会立即返回失败，或会阻塞到可以完成所需操作为止，具体取决于调用的是阻塞还是非阻塞写入或读取函数。如果尝试读取或尝试写入的数据量超出队列容量，则读取或写入操作一定会立即失败。

## 设置 FMQ

一个消息队列需要多个 MessageQueue 对象：一个对象用作数据写入目标位置，以及一个或多个对象用作数据读取来源。没有关于哪些对象用于写入数据或读取数据的显式配置；用户需负责确保没有对象既用于读取数据又用于写入数据，也就是说最多只有一个写入器，并且对于已同步队列，最多只有一个读取器。

#### 创建第一个 MessageQueue 对象

通过单个调用创建并配置消息队列：

```
#include <fmq/MessageQueue.h>
using android::hardware::kSynchronizedReadWrite;
using android::hardware::kUnsynchronizedWrite;
using android::hardware::MQDescriptorSync;
using android::hardware::MQDescriptorUnsync;
using android::hardware::MessageQueue;
....
// For a synchronized non-blocking FMQ
mFmqSynchronized =
  new (std::nothrow) MessageQueue<uint16_t, kSynchronizedReadWrite>
      (kNumElementsInQueue);
// For an unsynchronized FMQ that supports blocking
mFmqUnsynchronizedBlocking =
  new (std::nothrow) MessageQueue<uint16_t, kUnsynchronizedWrite>
      (kNumElementsInQueue, true /* enable blocking operations */);

```

*   `MessageQueue<T, flavor>(numElements)` 初始化程序负责创建并初始化支持消息队列功能的对象。
*   `MessageQueue<T, flavor>(numElements, configureEventFlagWord)` 初始化程序负责创建并初始化支持消息队列功能和阻塞的对象。
*   `flavor` 可以是 `kSynchronizedReadWrite`（对于已同步队列）或 `kUnsynchronizedWrite`（对于未同步队列）。
*   `uint16_t`（在本示例中）可以是任意不涉及嵌套式缓冲区（无 `string` 或 `vec` 类型）、句柄或接口的 [HIDL 定义的类型](https://source.android.com/devices/architecture/hidl-cpp/types.html)。
*   `kNumElementsInQueue` 表示队列的大小（以条目数表示）；它用于确定将为队列分配的共享内存缓冲区的大小。

#### 创建第二个 MessageQueue 对象

使用从消息队列的第一侧获取的 `MQDescriptor` 对象创建消息队列的第二侧。通过 HIDL RPC 调用将 `MQDescriptor` 对象发送到将容纳消息队列末端的进程。`MQDescriptor` 包含该队列的相关信息，其中包括：

*   用于映射缓冲区和写入指针的信息。
*   用于映射读取指针的信息（如果队列已同步）。
*   用于映射事件标记字词的信息（如果队列是阻塞队列）。
*   对象类型 (`<T, flavor>`)，其中包含 [HIDL 定义的队列元素类型](https://source.android.com/devices/architecture/hidl-cpp/types.html)和队列风格（已同步或未同步）。

`MQDescriptor` 对象可用于构建 `MessageQueue` 对象：

```
MessageQueue<T, flavor>::MessageQueue(const MQDescriptor<T, flavor>& Desc, bool resetPointers)

```

`resetPointers` 参数表示是否在创建此 `MessageQueue` 对象时将读取和写入位置重置为 0。在未同步队列中，读取位置（在未同步队列中，是每个 `MessageQueue` 对象的本地位置）在此对象创建过程中始终设为 0。通常，`MQDescriptor` 是在创建第一个消息队列对象过程中初始化的。要对共享内存进行额外的控制，您可以手动设置 `MQDescriptor`（`MQDescriptor` 是在 [`system/libhidl/base/include/hidl/MQDescriptor.h`](https://android.googlesource.com/platform/system/libhidl/+/master/base/include/hidl/MQDescriptor.h)中定义的），然后按照本部分所述内容创建每个 `MessageQueue` 对象。

#### 阻塞队列和事件标记

默认情况下，队列不支持阻塞读取/写入。有两种类型的阻塞读取/写入调用：

*   短格式：有三个参数（数据指针、项数、超时）。支持阻塞针对单个队列的各个读取/写入操作。在使用这种格式时，队列将在内部处理事件标记和位掩码，并且第一个消息队列对象必须初始化为第二个参数为 `true`。例如：

```
// For an unsynchronized FMQ that supports blocking
mFmqUnsynchronizedBlocking =
  new (std::nothrow) MessageQueue<uint16_t, kUnsynchronizedWrite>
      (kNumElementsInQueue, true /* enable blocking operations */);

```

*   长格式：有六个参数（包括事件标记和位掩码）。支持在多个队列之间使用共享 EventFlag 对象，并允许指定要使用的通知位掩码。在这种情况下，必须为每个读取和写入调用提供事件标记和位掩码。

对于长格式，可在每个 readBlocking() 和 writeBlocking() 调用中显式提供 EventFlag。

可以将其中一个队列初始化为包含一个内部事件标记，如果是这样，则必须使用 getEventFlagWord() 从相应队列的 MessageQueue 对象中提取该标记，以用于在每个进程中创建与其他 FMQ 一起使用的 EventFlag 对象。或者，可以将 EventFlag 对象初始化为具有任何合适的共享内存。

一般来说，每个队列都应只使用以下三项之一：非阻塞、短格式阻塞，或长格式阻塞。混合使用也不算是错误；但要获得理想结果，则需要谨慎地进行编程。

## 使用 MessageQueue

MessageQueue 对象的公共 API 是：

```
size_t availableToWrite()  // Space available (number of elements).
size_t availableToRead()  // Number of elements available.
size_t getQuantumSize()  // Size of type T in bytes.
size_t getQuantumCount() // Number of items of type T that fit in the FMQ.
bool isValid() // Whether the FMQ is configured correctly.
const MQDescriptor<T, flavor>* getDesc()  // Return info to send to other process.

bool write(const T* data)  // Write one T to FMQ; true if successful.
bool write(const T* data, size_t count) // Write count T's; no partial writes.

bool read(T* data);  // read one T from FMQ; true if successful.
bool read(T* data, size_t count);  // Read count T's; no partial reads.

bool writeBlocking(const T* data, size_t count, int64_t timeOutNanos = 0);
bool readBlocking(T* data, size_t count, int64_t timeOutNanos = 0);

// Allows multiple queues to share a single event flag word
std::atomic<uint32_t>* getEventFlagWord();

bool writeBlocking(const T* data, size_t count, uint32_t readNotification,
uint32_t writeNotification, int64_t timeOutNanos = 0,
android::hardware::EventFlag* evFlag = nullptr); // Blocking write operation for count Ts.

bool readBlocking(T* data, size_t count, uint32_t readNotification,
uint32_t writeNotification, int64_t timeOutNanos = 0,
android::hardware::EventFlag* evFlag = nullptr) // Blocking read operation for count Ts;

//APIs to allow zero copy read/write operations
bool beginWrite(size_t nMessages, MemTransaction* memTx) const;
bool commitWrite(size_t nMessages);
bool beginRead(size_t nMessages, MemTransaction* memTx) const;
bool commitRead(size_t nMessages);

```

**1.** availableToWrite() 和 availableToRead() 可用于确定在一次操作中可传输的数据量。
在未同步队列中：

*   availableToWrite() 始终返回队列容量。
*   每个读取器都有自己的读取位置，并会针对 availableToRead() 进行自己的计算。
*   如果是读取速度缓慢的读取器，队列可以溢出，这可能会导致 availableToRead() 返回的值大于队列的大小。发生溢出后进行的第一次读取操作将会失败，并且会导致相应读取器的读取位置被设为等于当前写入指针，无论是否通过 availableToRead() 报告了溢出都是如此。

**2.** 如果所有请求的数据都可以（并已）传输到队列/从队列传出，则 read() 和 write() 方法会返回 true。这些方法不会阻塞；它们要么成功（并返回 true），要么立即返回失败 (false)。

**3.** readBlocking() 和 writeBlocking() 方法会等到可以完成请求的操作，或等到超时（timeOutNanos 值为 0 表示永不超时）。
阻塞操作使用事件标记字词来实现。
默认情况下，每个队列都会创建并使用自己的标记字词来支持短格式的 readBlocking() 和 writeBlocking()。
多个队列可以共用一个字词，这样一来，进程就可以等待对任何队列执行写入或读取操作。可
以通过调用 getEventFlagWord() 获得指向队列事件标记字词的指针，此类指针（或任何指向合适的共享内存位置的指针）可用于创建 EventFlag 对象，以传递到其他队列的长格式 readBlocking() 和 writeBlocking()。readNotification 和 writeNotification 参数用于指示事件标记中的哪些位应该用于针对相应队列发出读取和写入信号。readNotification 和 writeNotification 是 32 位的位掩码。
readBlocking() 会等待 writeNotification 位；如果该参数为 0，则调用一定会失败。
如果 readNotification 值为 0，则调用不会失败，但成功的读取操作将不会设置任何通知位。
在已同步队列中，这意味着相应的 writeBlocking() 调用一定不会唤醒，除非已在其他位置对相应的位进行设置
。在未同步队列中，writeBlocking() 将不会等待（它应仍用于设置写入通知位），而且对于读取操作来说，不适合设置任何通知位。
同样，如果 readNotification 为 0，writeblocking() 将会失败，并且成功的写入操作会设置指定的 writeNotification 位。
要一次等待多个队列，请使用 EventFlag 对象的 wait() 方法来等待通知的位掩码。wait() 方法会返回一个状态字词以及导致系统设置唤醒的位。然后，该信息可用于验证相应的队列是否有足够的控件或数据来完成所需的写入/读取操作，并执行非阻塞 write()/read()。要获取操作后通知，请再次调用 EventFlag 的 wake() 方法。

#### 零复制操作

read/write/readBlocking/writeBlocking() API 会将指向输入/输出缓冲区的指针作为参数，并在内部使用 memcpy() 调用，以便在相应缓冲区和 FMQ 环形缓冲区之间复制数据。为了提高性能，Android 8.0 及更高版本包含一组 API，这些 API 可提供对环形缓冲区的直接指针访问，这样便无需使用 memcpy 调用。

```
bool beginWrite(size_t nMessages, MemTransaction* memTx) const;
bool commitWrite(size_t nMessages);

bool beginRead(size_t nMessages, MemTransaction* memTx) const;
bool commitRead(size_t nMessages);

```

*   beginWrite 方法负责提供用于访问 FMQ 环形缓冲区的基址指针。在数据写入之后，使用 commitWrite() 提交数据。beginRead/commitRead 方法的运作方式与之相同。
*   beginRead/Write 方法会将要读取/写入的消息条数视为输入，并会返回一个布尔值来指示是否可以执行读取/写入操作。如果可以执行读取或写入操作，则 memTx 结构体中会填入基址指针，这些指针可用于对环形缓冲区共享内存进行直接指针访问。
*   MemRegion 结构体包含有关内存块的详细信息，其中包括基础指针（内存块的基址）和以 T 表示的长度（以 HIDL 定义的消息队列类型表示的内存块长度）。
*   MemTransaction 结构体包含两个 MemRegion 结构体（first 和 second），因为对环形缓冲区执行读取或写入操作时可能需要绕回到队列开头。这意味着，要对 FMQ 环形缓冲区执行数据读取/写入操作，需要两个基址指针。

从 MemRegion 结构体获取基址和长度：

```
T* getAddress(); // gets the base address
size_t getLength(); // gets the length of the memory region in terms of T
size_t getLengthInBytes(); // gets the length of the memory region in bytes

```

获取对 MemTransaction 对象内的第一个和第二个 MemRegion 的引用：

```
const MemRegion& getFirstRegion(); // get a reference to the first MemRegion
const MemRegion& getSecondRegion(); // get a reference to the second MemRegion

```

使用零复制 API 写入 FMQ 的示例：

```
MessageQueueSync::MemTransaction tx;
if (mQueue->beginRead(dataLen, &tx)) {
    auto first = tx.getFirstRegion();
    auto second = tx.getSecondRegion();

    foo(first.getAddress(), first.getLength()); // method that performs the data write
    foo(second.getAddress(), second.getLength()); // method that performs the data write

    if(commitWrite(dataLen) == false) {
       // report error
    }
} else {
   // report error
}

```

以下辅助方法也是 MemTransaction 的一部分：

*   **T * getSlot(size_t idx);**
    返回一个指针，该指针指向属于此 MemTransaction 对象一部分的 MemRegions 内的槽位 idx。如果 MemTransaction 对象表示要读取/写入 N 个类型为 T 的项目的内存区域，则 idx 的有效范围在 0 到 N-1 之间。

*   **bool copyTo(const T * data, size_t startIdx, size_t nMessages = 1);**
    将 nMessages 个类型为 T 的项目写入到该对象描述的内存区域，从索引 startIdx 开始。此方法使用 memcpy()，但并非旨在用于零复制操作。如果 MemTransaction 对象表示要读取/写入 N 个类型为 T 的项目的内存区域，则 idx 的有效范围在 0 到 N-1 之间。

*   **bool copyFrom(T * data, size_t startIdx, size_t nMessages = 1);**
    一种辅助方法，用于从该对象描述的内存区域读取 nMessages 个类型为 T 的项目，从索引 startIdx 开始。此方法使用 memcpy()，但并非旨在用于零复制操作。

## HIDL消息队列的使用方法总结

##### 在创建侧执行的操作：

**1.** 创建消息队列对象，如上所述。
**2.** 使用 isValid() 验证对象是否有效。
**3.** 如果您要通过将 EventFlag 传递到长格式的readBlocking()/writeBlocking() 来等待多个队列，则可以从经过初始化的 MessageQueue 对象提取事件标记指针（使用 getEventFlagWord()）以创建标记，然后使用该标记创建必需的 EventFlag 对象。
**4.** 使用 MessageQueue getDesc() 方法获取描述符对象。
**5.** 在 .hal 文件中，为某个方法提供一个类型为 fmq_sync 或 fmq_unsync 的参数，其中 T 是 HIDL 定义的一种合适类型。使用此方法将 getDesc() 返回的对象发送到接收进程。

#### 在接收侧执行的操作：

**1.** 使用描述符对象创建 MessageQueue 对象。务必使用相同的队列风格和数据类型，否则将无法编译模板。
**2.** 如果您已提取事件标记，则在接收进程中从相应的 MessageQueue 对象提取该标记。
**3.** 使用 MessageQueue 对象传输数据。

## 使用 Binder IPC

从 Android O 开始，Android 框架和 HAL 现在使用 Binder 互相通信。由于这种通信方式极大地增加了 Binder 流量，因此 Android O 包含了几项改进，旨在确保 Binder IPC 的速度。集成最新版 Binder 驱动程序的 SoC 供应商和原始设备制造商 (OEM) 应该查看这些改进的列表、用于 3.18、4.4 和 4.9 版内核的相关 SHA，以及所需的用户空间更改。

#### 多个 Binder 域（上下文）

为了明确地拆分框架（与设备无关）和供应商（与具体设备相关）代码之间的 Binder 流量，Android O 引入了“Binder 上下文”这一概念。每个 Binder 上下文都有自己的设备节点和上下文（服务）管理器。您只能通过上下文管理器所属的设备节点对其进行访问，并且在通过特定上下文传递 Binder 节点时，只能由另一个进程从相同的上下文访问上下文管理器，从而确保这些域完全互相隔离。如需使用方法的详细信息，请参阅 [vndbinder](https://source.android.com/devices/architecture/hidl/binder-ipc#vndbinder) 和 [vndservicemanager](https://source.android.com/devices/architecture/hidl/binder-ipc#vndservicemanager)。

#### 分散-集中

在之前的 Android 版本中，Binder 调用中的每条数据都会被复制 3 次：

*   一次是在调用进程中将数据序列化为 Parce
*   一次是在内核驱动程序中将 Parcel 复制到目标进程
*   一次是在目标进程中对 Parcel 进行反序列化

Android O 使用`分散-集中`优化机制将复制次数从 3 次减少到了 1 次。数据保留其原始结构和内存布局，且 Binder 驱动程序会立即将数据复制到目标进程中，而不是先在 `Parcel` 中对数据进行序列化。在目标进程中，这些数据的结构和内存布局保持不变，并且，在无需再次复制的情况下即可读取这些数据。

## vndbinder

Android O 支持供应商服务使用新的 Binder 域，这可通过使用 /dev/vndbinder（而非 /dev/binder）进行访问。添加 /dev/vndbinder 后，Android 现在拥有以下 3 个 IPC 域：

| IPC 域 | 说明 |
| --- | --- |
| /dev/binder | 框架/应用进程之间的 IPC，使用 AIDL 接口 |
| dev/hwbinder | 框架/供应商进程之间的 IPC，使用 HIDL 接口供应商进程之间的 IPC，使用 HIDL 接口 |
| /dev/vndbinder | 供应商/供应商进程之间的 IPC，使用 AIDL 接口 |

## HIDL Memory Block

HIDL 内存块是一个建立在HIDL @1.0::IAllocator, 和 HIDL @1.0::IMapper的抽象层。
它是为具有多个内存块共享单个内存堆的HIDL Severis而设计的。

### 结构

HIDL内存块体系结构包括多个内存块共享一个内存堆的HIDL services：

![image](http://q2k8089lz.bkt.clouddn.com/FMQ-1.webp)

### 使用实例

##### 声明HAL

IFoo HAL:

```
import android.hidl.memory.block@1.0::MemoryBlock;

interface IFoo {
    getSome() generates(MemoryBlock block);
    giveBack(MemoryBlock block);
};

```

Android.bp:

```
hidl_interface {
    ...
    srcs: [
        "IFoo.hal",
    ],
    interfaces: [
        "android.hidl.memory.block@1.0",
        ...
};

```

### 实施HAL

1.获取 hidl_memory

```
#include <android/hidl/allocator/1.0/IAllocator.h>

using ::android::hidl::allocator::V1_0::IAllocator;
using ::android::hardware::hidl_memory;
...
  sp<IAllocator> allocator = IAllocator::getService("ashmem");
  allocator->allocate(2048, [&](bool success, const hidl_memory& mem)
  {
        if (!success) { /* error */ }
        // you can now use the hidl_memory object 'mem' or pass it
  }));

```

2.创建一个HidlMemoryDealer来获取hidl_memory：

```
#include <hidlmemory/HidlMemoryDealer.h>

using ::android::hardware::HidlMemoryDealer
/* The mem argument is acquired in the Step1, returned by the ashmemAllocator->allocate */
sp<HidlMemoryDealer> memory_dealer = HidlMemoryDealer::getInstance(mem);

```

3.使用MemoryBlock申请内存

```
struct MemoryBlock {
IMemoryToken token;
uint64_t size;
uint64_t offset;
};

```

```
#include <android/hidl/memory/block/1.0/types.h>

using ::android::hidl::memory::block::V1_0::MemoryBlock;

Return<void> Foo::getSome(getSome_cb _hidl_cb) {
    MemoryBlock block = memory_dealer->allocate(1024);
    if(HidlMemoryDealer::isOk(block)){
        _hidl_cb(block);
    ...

```

4.  解除分配：

```
Return<void> Foo::giveBack(const MemoryBlock& block) {
    memory_dealer->deallocate(block.offset);
...

```

5.使用数据

```
#include <hidlmemory/mapping.h>
#include <android/hidl/memory/1.0/IMemory.h>

using ::android::hidl::memory::V1_0::IMemory;

sp<IMemory> memory = mapMemory(block);
uint8_t* data =

static_cast<uint8_t*>(static_cast<void*>(memory->getPointer()));

```

6.配置 Android.bp

```
shared_libs: [
        "android.hidl.memory@1.0",

        "android.hidl.memory.block@1.0"

        "android.hidl.memory.token@1.0",
        "libhidlbase",
        "libhidlmemory",

```

## 线程模型

注意：
标记为 oneway 的方法不会阻塞。对于未标记为 oneway 的方法，在服务器完成执行任务或调用同步回调（以先发生者为准）之前，客户端的方法调用将一直处于阻塞状态。
服务器方法实现最多可以调用一个同步回调；多出的回调调用会被舍弃并记录为错误。如果方法应通过回调返回值，但未调用其回调，系统会将这种情况记录为错误，并作为传输错误报告给客户端。

#### 直通模式下的线程

在直通模式下，大多数调用都是同步的。不过，为确保 oneway 调用不会阻塞客户端这一预期行为，系统会分别为每个进程创建线程。

#### 绑定式 HAL 中的线程

为了处理传入的 RPC 调用（包括从 HAL 到 HAL 用户的异步回调）和终止通知，系统会为使用 HIDL 的每个进程关联一个线程池。
如果单个进程实现了多个 HIDL 接口和/或终止通知处理程序，则所有这些接口和/或处理程序会共享其线程池。当进程接收从客户端传入的方法调用时，它会从线程池中选择一个空闲线程，并在该线程上执行调用。如果没有空闲的线程，它将会阻塞，直到有可用线程为止。

如果服务器只有一个线程，则传入服务器的调用将按顺序完成。具有多个线程的服务器可以不按顺序完成调用，即使客户端只有一个线程也是如此

不过，对于特定的接口对象，`oneway` 调用会保证按顺序进行（请参阅服务器线程模型。对于托管了多个界面的多线程服务器，对不同界面的多项 `oneway` 调用可能会并行处理，也可能会与其他阻塞调用并行处理。

#### 服务器线程模型

（直通模式除外）HIDL 接口的服务器实现位于不同于客户端的进程中，并且需要一个或多个线程等待传入的方法调用。

这些线程构成服务器的线程池；服务器可以决定它希望在其线程池中运行多少线程，并且可以利用一个线程大小的线程池来按顺序处理其接口上的所有调用。如果服务器的线程池中有多个线程，则服务器可以在其任何接口上接收同时传入的调用（在 C++ 中，这意味着必须小心锁定共享数据）。

传入同一接口的单向调用会按顺序进行处理。如果多线程客户端在接口 IFoo 上调用 method1 和 method2，并在接口 IBar 上调用 method3，则 method1 和 method2 将始终按顺序运行，但 method3 可以与 method1 和 method2 并行运行。

单一客户端执行线程可能会通过以下两种方式在具有多个线程的服务器上引发并行运行：

*   oneway 调用不会阻塞。如果执行 oneway 调用，然后调用非 oneway，则服务器可以同时执行 oneway 调用和非 oneway 调用。
*   当系统从服务器调用回调时，通过同步回调传回数据的服务器方法可以立即解除对客户端的阻塞。

#### 客户端线程模型

非阻塞调用（带有 oneway 关键字标记的函数）与阻塞调用（未指定 oneway 关键字的函数）的客户端线程模型有所不同。

###### 阻塞调用

对于阻塞调用来说，除非发生以下情况之一，否则客户端将一直处于阻塞状态：

*   出现传输错误；Return 对象包含可通过 Return::isOk() 检索的错误状态。
*   服务器实现调用回调（如果有）。
*   服务器实现返回值（如果没有回调参数）。

如果成功的话，客户端以参数形式传递的回调函数始终会在函数本身返回之前被服务器调用。回调是在进行函数调用的同一线程上执行，所以在函数调用期间，实现人员必须谨慎地持有锁（并尽可能彻底避免持有锁）。不含 generates 语句或 oneway 关键字的函数仍处于阻塞状态；在服务器返回 Return<void> 对象之前，客户端将一直处于阻塞状态。

###### 单向调用

如果某个函数标记有 oneway，则客户端会立即返回，而不会等待服务器完成其函数调用。

## 数据类型

本节只列举C++的相关数据类型。

| HIDL 类型 | C++ 类型 | 头文件/库 |
| --- | --- | --- |
| enum | enum class |
| uint8_t..uint64_t | uint8_t..uint64_t | <stdint.h> |
| int8_t..int64_t | int8_t..int64_t | <stdint.h> |
| float | float |
| double | double |
| vec<T> | hidl_vec<T> | libhidlbase |
| T[S1][S2]...[SN] | T[S1][S2]...[SN] |
| string | hidl_string | libhidlbase |
| handle | hidl_handle | libhidlbase |
| opaque | uint64_t | <stdint.h> |
| struct | struct |
| union | union |
| fmq_sync | MQDescriptorSync | libhidlbase |
| fmq_unsync | MQDescriptorUnsync | libhidlbase |

### 枚举

HIDL 形式的枚举会变成 C++ 形式的枚举。例如：

```
enum Mode : uint8_t { WRITE = 1 << 0, READ = 1 << 1 };

```

变为：

```
enum class Mode : uint8_t { WRITE = 1, READ = 2 };

```

### bitfield<T>

bitfield<T>（其中 T 是用户定义的枚举）会变为 C++ 形式的该枚举的底层类型。在上述示例中，bitfield<Mode> 会变为 uint8_t。

### vec<T>

hidl_vec<T> 类模板是 libhidlbase 的一部分，可用于传递具备任意大小的任何 HIDL 类型的矢量。与之相当的具有固定大小的容器是 hidl_array。此外，您也可以使用 hidl_vec::setToExternal() 函数将 hidl_vec<T> 初始化为指向 T 类型的外部数据缓冲区。

除了在生成的 C++ 头文件中适当地发出/插入结构之外，您还可以使用 vec<T> 生成一些便利函数，用于转换到 std::vector 和 T 裸指针或从它们进行转换。如果您将 vec<T> 用作参数，则使用它的函数将过载（将生成两个原型），以接受并传递该参数的 HIDL 结构和 std::vector<T> 类型。

### 数组

hidl 中的常量数组由 libhidlbase 中的 hidl_array 类表示。hidl_array<T, S1, S2, …, SN> 表示具有固定大小的 N 维数组 T[S1][S2]…[SN]。

### 字符串

hidl_string 类（libhidlbase 的一部分）可用于通过 HIDL 接口传递字符串，并在 /system/libhidl/base/include/hidl/HidlSupport.h 下进行定义。该类中的第一个存储位置是指向其字符缓冲区的指针。

### struct

HIDL 形式的 struct 只能包含固定大小的数据类型，不能包含任何函数。HIDL 结构定义会直接映射到 C++ 形式的标准布局 struct，从而确保 struct 具有一致的内存布局。一个struct可以包括多种指向单独的可变长度缓冲区的 HIDL 类型（包括 handle、string 和 vec<T>）。

### handle

handle 类型由 C++ 形式的 hidl_handle 结构表示，该结构是一个简单的封装容器，用于封装指向 const native_handle_t 对象的指针（该对象已经在 Android 中存在了很长时间）。

```
typedef struct native_handle
{
    int version;        /* sizeof(native_handle_t) */
    int numFds;         /* number of file descriptors at &data[0] */
    int numInts;        /* number of ints at &data[numFds] */
    int data[0];        /* numFds + numInts ints */
} native_handle_t;

```

### memory

HIDL memory 类型会映射到 libhidlbase 中的 hidl_memory 类，该类表示未映射的共享内存。这是要在 HIDL 中共享内存而必须在进程之间传递的对象。要使用共享内存，需满足以下条件：
1.获取 IAllocator 的实例（当前只有“ashmem”实例可用）并使用该实例分配共享内存。
2.IAllocator::allocate() 返回 hidl_memory 对象，该对象可通过 HIDL RPC 传递，并能使用 libhidlmemory 的 mapMemory 函数映射到某个进程。
3.mapMemory 返回对可用于访问内存的 sp<IMemory> 对象的引用（IMemory 和 IAllocator 在 android.hidl.memory@1.0 中定义）。

IAllocator 的实例可用于分配内存：

```
#include <android/hidl/allocator/1.0/IAllocator.h>
#include <android/hidl/memory/1.0/IMemory.h>
#include <hidlmemory/mapping.h>
using ::android::hidl::allocator::V1_0::IAllocator;
using ::android::hidl::memory::V1_0::IMemory;
using ::android::hardware::hidl_memory;
....
  sp<IAllocator> ashmemAllocator = IAllocator::getService("ashmem");
  ashmemAllocator->allocate(2048, [&](bool success, const hidl_memory& mem) {
        if (!success) { /* error */ }
        // now you can use the hidl_memory object 'mem' or pass it around
  }));

```

对内存的实际更改必须通过 IMemory 对象完成（在创建 mem 的一端或在通过 HIDL RPC 接收更改的一端完成）:

```
// Same includes as above

sp<IMemory> memory = mapMemory(mem);
void* data = memory->getPointer();
memory->update();
// update memory however you wish after calling update and before calling commit
data[0] = 42;
memory->commit();
// …
memory->update(); // the same memory can be updated multiple times
// …
memory->commit();

```

### 接口

接口可作为对象传递。“接口”一词可用作 android.hidl.base@1.0::IBase 类型的语法糖；此外，当前的接口以及任何导入的接口都将定义为一个类型。

存储接口的变量应该是强指针：sp<IName>。接受接口参数的 HIDL 函数会将原始指针转换为强指针，从而导致不可预料的行为（可能会意外清除指针）。为避免出现问题，请务必将 HIDL 接口存储为 sp<>。

## 创建 HAL 客户端

首先将 HAL 库添加到 makefile 中：

```
Make：LOCAL_SHARED_LIBRARIES += android.hardware.nfc@1.0
Soong：shared_libs: [ …, android.hardware.nfc@1.0 ]

```

接下来，添加 HAL 头文件：

```
#include <android/hardware/nfc/1.0/IFoo.h>
…
// in code:
sp<IFoo> client = IFoo::getService();
client->doThing();

```

## 创建 HAL 服务器

要创建 HAL 实现，您必须具有表示 HAL 的 .hal 文件并已在 hidl-gen 上使用 -Lmakefile 或 -Landroidbp 为 HAL 生成 makefile（./hardware/interfaces/update-makefiles.sh 会为内部 HAL 文件执行这项操作，这是一个很好的参考）。从 libhardware 通过 HAL 传输时，您可以使用 c2hal 轻松完成许多此类工作。

要创建必要的文件来实现您的 HAL，请使用以下代码：

```
PACKAGE=android.hardware.nfc@1.0
LOC=hardware/interfaces/nfc/1.0/default/
m -j hidl-gen
hidl-gen -o $LOC -Lc++-impl -randroid.hardware:hardware/interfaces \
    -randroid.hidl:system/libhidl/transport $PACKAGE
hidl-gen -o $LOC -Landroidbp-impl -randroid.hardware:hardware/interfaces \
    -randroid.hidl:system/libhidl/transport $PACKAGE

```

接下来，使用相应功能填写存根并设置守护进程。守护进程代码（支持直通）示例：

```
#include <hidl/LegacySupport.h>

int main(int /* argc */, char* /* argv */ []) {
    return defaultPassthroughServiceImplementation<INfc>("nfc");
}

```

defaultPassthroughServiceImplementation 将对提供的 -impl 库执行 dlopen() 操作，并将其作为绑定式服务提供。守护进程代码（对于纯绑定式服务）示例：

```
int main(int /* argc */, char* /* argv */ []) {
    // This function must be called before you join to ensure the proper
    // number of threads are created. The threadpool will never exceed
    // size one because of this call.
    ::android::hardware::configureRpcThreadpool(1 /*threads*/, true /*willJoin*/);

    sp nfc = new Nfc();
    const status_t status = nfc->registerAsService();
    if (status != ::android::OK) {
        return 1; // or handle error
    }

    // Adds this thread to the threadpool, resulting in one total
    // thread in the threadpool. We could also do other things, but
    // would have to specify 'false' to willJoin in configureRpcThreadpool.
    ::android::hardware::joinRpcThreadpool();
    return 1; // joinRpcThreadpool should never return
}

```

此守护进程通常存在于 `$PACKAGE + "-service-suffix"`（例如 `android.hardware.nfc@1.0-service`）中，但也可以位于任何位置。HAL 的特定类的 sepolicy 是属性 `hal_<module>`（例如 `hal_nfc)`）。您必须将此属性应用到运行特定 HAL 的守护进程（如果同一进程提供多个 HAL，则可以将多个属性应用到该进程）。

## 软件包

HIDL 接口软件包位于 hardware/interfaces 或 vendor/ 目录下（少数例外情况除外）。hardware/interfaces 顶层会直接映射到 android.hardware 软件包命名空间；版本是软件包（而不是接口）命名空间下的子目录。

hidl-gen 编译器会将 .hal 文件编译成一组 .h 和 .cpp 文件。这些自动生成的文件可用来编译客户端/服务器实现链接到的共享库。用于编译此共享库的 Android.bp 文件由 hardware/interfaces/update-makefiles.sh 脚本自动生成。每次将新软件包添加到 hardware/interfaces 或在现有软件包中添加/移除 .hal 文件时，您都必须重新运行该脚本，以确保生成的共享库是最新的。

例如，IFoo.hal 示例文件应该位于 hardware/interfaces/samples/1.0 下。IFoo.hal 示例文件可以在 samples 软件包中创建一个 IFoo 接口：

```
package android.hardware.samples@1.0;
interface IFoo {
    struct Foo {
       int64_t someValue;
       handle  myHandle;
    };

    someMethod() generates (vec<uint32_t>);
    anotherMethod(Foo foo) generates (int32_t ret);
};

```

#### 生成的文件

HIDL 软件包中自动生成的文件会链接到与软件包同名的单个共享库（例如 android.hardware.samples@1.0）。该共享库还会导出单个标头 IFoo.h，用于包含在客户端和服务器中。绑定式模式使用 hidl-gen 编译器并以 IFoo.hal 接口文件作为输入，它具有以下自动生成的文件：

![image](http://q2k8089lz.bkt.clouddn.com/FM1-2.webp)

## 链接到共享库

使用软件包中的任何接口的客户端或服务器必须在下面的其中一 (1) 个位置包含该软件包的共享库：
在 Android.mk 中：

```
LOCAL_SHARED_LIBRARIES += android.hardware.samples@1.0

```

在 Android.bp 中：

```
shared_libs: [
    /* ... */
    "android.hardware.samples@1.0",
],

```

![image](http://q2k8089lz.bkt.clouddn.com/FMQ-3.webp)

## 命名空间

HIDL 函数和类型（如 Return<T> 和 Void()）已在命名空间 ::android::hardware 中进行声明。软件包的 C++ 命名空间由软件包的名称和版本号确定。例如，hardware/interfaces 下版本为 1.2 的软件包 mypackage 具有以下特质：

*   C++ 命名空间是 ::android::hardware::mypackage::V1_2
*   该软件包中 IMyInterface 的完全限定名称是
    ::android::hardware::mypackage::V1_2::IMyInterface（IMyInterface 是一个标识符，而不是命名空间的一部分）。
*   在软件包的 types.hal 文件中定义的类型标识为 ::android::hardware::mypackage::V1_2::MyPackageType

学习算是告一段落，东西太多了，消化消化，接下来开始实战。
