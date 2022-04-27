---
title: 共享内存实现—Ring-Buffer
date: 2022-04-27 21:35:33
tags:
- FastDDS
- DDS
- 自动驾驶

categories:
- FastDDS

cover: /img/fastdds-04-shm-ringbuffer.png
---

> 上文有提到，Shared Memory Port用来提供读写的通道，在这个通道中传输的缓存被设计为一个Ring-Buffer，提供P0的发现服务数据读写，或者用户自定义的数据读写。
> 

## `MultiProducerConsumerRingBuffer`

这个名字有点长，但是顾名思义，Ring-Buffer被设计为多生产者和多消费者的模式，有如下几个特点：

- 读写操作都是无锁的
- 同一个RingBuffer中的数据单元（cell）数目是固定的，数据类型都是一样的
- 消费者（listener）必须注册获得listener来访问数据
- 当一个数据单元（cell）被推送到buffer，这个数据单元的计数值为当前注册的listener个数
- 当某个数据单元（cell）被所有的listener都取出之后，这个数据单元会被释放掉

### `MultiProducerConsumerRingBuffer::Cell`

```cpp
class Cell
    {
    public:
        const T& data() const
        {
            return data_;
        }
        void data(
                const T& data)
        {
            data_ = data;
        }
        uint32_t ref_counter() const
        {
            return ref_counter_.load(std::memory_order_relaxed);
        }
        friend class MultiProducerConsumerRingBuffer<T>;
        std::atomic<uint32_t> ref_counter_;
        T data_;
    };
```

`MultiProducerConsumerRingBuffer`是一个模板类，Cell中的T就是`MultiProducerConsumerRingBuffer`模板类的类型，可以看到Cell类对一个数据类型进行了简单的封装，提供了保存和获取这个数据的接口，还有一个重要的`ref_count_`成员变量，用来标识这个cell当前被多少个listener持有。

### `MultiProducerConsumerRingBuffer::Listener`

```cpp
class Listener
    {
    public:

        Listener(
                MultiProducerConsumerRingBuffer<T>& buffer,
                uint32_t write_p)
            : buffer_(buffer)
            , read_p_(write_p) {}
        ~Listener()
        {
            buffer_.unregister_listener(*this);
        }
        /**
         * @returns the Cell at the read pointer or nullptr if the buffer is empty
         */
        Cell* head()
        {
            auto pointer = buffer_.node_->pointer_.load(std::memory_order_relaxed);

            // If local read_pointer and write_pointer are equal => buffer is empty for this listener
            if (read_p_ == pointer.ptr.write_p )
            {
                return nullptr;
            }

            auto cell = &buffer_.cells_[get_pointer_value(read_p_)];
            return cell->ref_counter() != 0 ? cell : nullptr;
        }
        /**
         * Decreases the ref_counter of the head cell,
         * if the counter reaches 0 the cell becomes dirty
         * and free_cells are incremented
         * @return true if the cell ref_counter is 0 after pop
         * @throw std::exception if buffer is empty
         */
        bool pop()
        {
            auto cell = head();
            if (!cell)
            {
                throw std::runtime_error("Buffer empty");
            }
            auto counter = cell->ref_counter_.fetch_sub(1);
            assert(counter > 0);
            // If all the listeners have read the cell
            if (counter == 1)
            {
                // Increase the free cells => increase the global read pointer
                auto pointer = buffer_.node_->pointer_.load(std::memory_order_relaxed);
                while (!buffer_.node_->pointer_.compare_exchange_weak(pointer,
                        { { pointer.ptr.write_p, pointer.ptr.free_cells + 1 } },
                        std::memory_order_release,
                        std::memory_order_relaxed))
                {
                }
            }
            // Increase the local read pointer
            read_p_ = buffer_.inc_pointer(read_p_);
            return (counter == 1);
        }

    private:
        MultiProducerConsumerRingBuffer<T>& buffer_;
        uint32_t read_p_;
    };
```

Listener对象是通过`MultiProducerConsumerRingBuffer`的`register_listener()`接口获取到

```cpp
std::unique_ptr<Listener> register_listener()
    {
        // The new listener's read pointer is the current write pointer
        auto listener = std::unique_ptr<Listener>(
            new Listener(
                *this, node_->pointer_.load(std::memory_order_relaxed).ptr.write_p));

        node_->registered_listeners_++;

        return listener;
    }
```

可以看到创建的listener对象的第二个参数是当前RingBuffer的写入的位置，**也就是之前写入的数据时无法被正在注册的listener访问到的。**

同样的，Listener的`head()`接口获取到当前listener读位置的cell，当当前的读位置和当前ringbuffer的写位置相等时，返回`nullptr`，也就是当前listener注册之后没有新数据被写入。

Listener的pop()接口用来取出第一个cell，当cell被取出之后，它的引用计数会被减一。

### `MultiProducerConsumerRingBuffer::Node`

然后看一下`MultiProducerConsumerRingBuffer`的构造函数，需要传入一个cell的指针和放入cell的个数。初始化所有cell的引用参数，然后构造Node对象，Node里面存放了cell的个数和当前listener的个数。Node中的pointer用来标识当前RingBuffer写入的位置，它的三个成员函数可以好好研究一下。

```cpp
union PtrType
    {
        struct Pointer
        {
            uint32_t write_p;
            uint32_t free_cells;
        }
        ptr;
        uint64_t u;
    };

    struct Node
    {
        alignas(8) std::atomic<PtrType> pointer_;
        uint32_t total_cells_;

        uint32_t registered_listeners_;
    };
```

```cpp
static uint32_t get_pointer_value(
            uint32_t pointer)
    {
        // Bit 31 is loop_flag, 0-30 are value
        return pointer & 0x7FFFFFFF;
    }
```

这个函数用来获取当前指针（读/写）的值，虽然是一个RingBuffer，但是这个RingBuffer的读写指针不会重复的回环，会一直往下递增，直到达到最大值。

然后是`inc_pointer`和`pointer_to_head`函数，用来辅助计算获取指针。

MultiProducerConsumerRingBuffer还提供了一个copy()方法，用来把当前RingBuffer中给所有的数据到一个数组中。

```cpp
void copy(
            std::vector<const T*>* enqueued_cells)
    {
        if (node_->registered_listeners_ > 0)
        {
            auto pointer = node_->pointer_.load(std::memory_order_relaxed);

            uint32_t p = pointer_to_head(pointer);

            while (p != pointer.ptr.write_p)
            {
                auto cell = &cells_[get_pointer_value(p)];

                // If the cell has not been read by any listener
                if (cell->ref_counter() > 0)
                {
                    enqueued_cells->push_back(&cell->data());
                }

                p = inc_pointer(p);
            }
        }
    }
```