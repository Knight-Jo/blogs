+++
date = '2026-04-15T13:40:38+08:00'
title = 'Cpp_concurrent'
draft = false
+++

这是一篇面向中级开发者的 C++ 并发技术教程。文章既给出学习路线，也在关键点给出可直接落地的代码与解释，帮助你从“能写多线程”进阶到“能写出高性能、可维护的并发代码”。

---

## C++ 并发实战

我们将这个大板块细化为五个循序渐进的阶段，从 API 使用深入到 CPU 硬件级别的性能优化。

### 1. 锁

熟练使用基础线程和锁只是起点，后端开发更看重在复杂场景下如何选择合适的锁以最大化并发度。

* **核心知识点：**
    * **RAII 锁进阶：** 深入对比 `std::lock_guard`（轻量级、不可移动）与 `std::unique_lock`（灵活、支持延迟锁定和时间锁定、可移动）的语义差异与使用场景。
    * **读写锁的应用：** C++17 引入 `std::shared_mutex` 和 `std::shared_lock`；若使用 C++14，可改用 `std::shared_timed_mutex`。理解在“读多写少”的场景下读写锁优于普通互斥锁的原因。
    * **死锁的系统性防范：** 掌握按固定顺序加锁原则，理解 `std::scoped_lock` 的多锁规避策略。
* **面试高频：** “如何设计一个线程安全的单例模式？”（考察 Double-Checked Locking 与 `std::call_once`）。

#### 1.1 RAII 锁

不要手动调用 `mtx.lock()` 和 `mtx.unlock()`，使用 RAII(Resource Acquisition Is Initialization) 机制管理锁，保证异常安全。

并发优化的第一步通常不是换无锁结构，而是收紧“锁粒度(Critical Section)”，尽可能缩小临界区，让锁占用时间更短、并发度更高。

- `std::lock_guard`：创建时立即加锁，离开作用域时自动解锁，不可拷贝、不可移动。适合“只在一个作用域内保护临界区”。
- `std::unique_lock`：内部维护锁状态，支持延迟加锁 `std::defer_lock`、尝试加锁 `try_lock`、提前解锁 `unlock`，支持移动语义。适合需要手动控制锁粒度或配合 `std::condition_variable` 的场景。

```cpp
void process_data() {
    // 场景 1：简单保护，使用 lock_guard
    {
        std::lock_guard<std::mutex> lock(mtx);
        shared_data++;
        // 离开大括号，自动解锁
    }

    // 场景 2：需要中间解锁执行耗时非共享操作，使用 unique_lock
    {
        std::unique_lock<std::mutex> lock(mtx);
        shared_data++;

        lock.unlock(); // 提前手动解锁，减小锁粒度

        // 模拟耗时的 IO 或计算操作，此时不需要霸占锁
        std::cout << "执行耗时非共享操作..." << std::endl;

        lock.lock(); // 如果后续还需要访问共享资源，可以再次加锁
        shared_data--;
    }
}
```
核心逻辑：场景 1 用 `lock_guard` 自动管理锁生命周期；场景 2 用 `unique_lock` 主动缩小临界区。执行结果是共享变量的修改保持互斥，同时耗时操作不阻塞其他线程。

#### 1.2 读写锁：高并发读多写少

C++17 引入 `std::shared_mutex`，允许多个线程同时读，但写操作是独占的。若是 C++14，可使用 `std::shared_timed_mutex`。

```cpp
class ConfigManager {
private:
    std::map<std::string, std::string> config_;
    mutable std::shared_mutex rw_mtx_; // mutable 允许在 const 成员函数中加锁

public:
    // 读操作：允许多个线程同时进入
    std::string getConfig(const std::string& key) const {
        // 使用 shared_lock 加上共享锁(读锁)
        std::shared_lock<std::shared_mutex> lock(rw_mtx_);
        auto it = config_.find(key);
        return it != config_.end() ? it->second : "";
    }

    // 写操作：独占访问，阻塞其他所有的读和写
    void setConfig(const std::string& key, const std::string& value) {
        // 使用 unique_lock 加上排他锁(写锁)
        std::unique_lock<std::shared_mutex> lock(rw_mtx_);
        config_[key] = value;
    }
};
```
补充说明：`getConfig` 是 `const` 成员函数，但加锁会修改互斥量内部状态，因此互斥量必须声明为 `mutable`，否则无法编译。
核心逻辑：读操作使用 `std::shared_lock` 共享锁，写操作使用 `std::unique_lock` 排他锁。执行效果是“读并发、写独占”，提升读多写少场景的吞吐。

#### 1.3 `std::scoped_lock` 与死锁防范

C++17 引入 `std::scoped_lock`，可以一次性锁定多个互斥量，并采用死锁规避策略。
它的底层通常使用死锁避免算法(如统一内存地址排序加锁)，因此不仅是语法糖，更提供了工程级安全保障。

```cpp
class Account {
public:
    explicit Account(int b) : balance(b) {}
    std::mutex m;
    int balance;
};

void transfer(Account& from, Account& to, int amount) {
    // C++17 的 std::scoped_lock，可以接收任意数量的 mutex
    std::scoped_lock lock(from.m, to.m);

    if (from.balance >= amount) {
        from.balance -= amount;
        to.balance += amount;
        std::cout << "转账成功\n";
    }
}
```
核心逻辑：一次性获取多个互斥量，避免“先锁 A 再锁 B vs 先锁 B 再锁 A”的循环等待。执行结果是多账户转账安全且无死锁风险。

### 2. 异步编程

在现代 C++ 中，很多时候我们不需要直接操作 `std::thread` 和条件变量，基于任务的异步编程是更优雅的选择。

* **核心知识点：**
    * **条件变量避坑指南：** 彻底弄懂虚假唤醒(Spurious Wakeup)和丢失唤醒(Lost Wakeup)，熟练编写带 Predicate(谓词) 的 `cv.wait()`。
    * **Future/Promise 模型：** 掌握 `std::promise`、`std::future` 以及 `std::shared_future`，构建线程间数据通道。
    * **异步任务执行器：** 掌握 `std::async`，理解启动策略(`std::launch::async` vs `std::launch::deferred`)对资源与性能的影响。

#### 2.1 条件变量

`std::condition_variable` 用于线程间同步。

- 丢失唤醒(Lost Wakeup)：A 线程调用 `notify` 时，B 线程尚未进入 `wait`，唤醒信号被错过。
- 虚假唤醒(Spurious Wakeup)：即使没有线程调用 `notify`，等待线程也可能被唤醒，因此必须在循环中检查条件。

```cpp
std::mutex mtx;
std::condition_variable cv;
bool data_ready = false; // 共享状态标志

void worker_thread() {
    std::unique_lock<std::mutex> lock(mtx); // 条件变量必须配合 unique_lock

    std::cout << "工作线程：等待数据准备...\n";

    // wait 的第二个参数传入 Lambda 表达式(谓词 Predicate)
    // 它的底层等价于：while (!data_ready) { cv.wait(lock); }
    cv.wait(lock, [] { return data_ready; });

    std::cout << "工作线程：数据已就绪，开始处理！\n";
}

void main_thread() {
    std::this_thread::sleep_for(std::chrono::seconds(1)); // 模拟耗时准备

    {
        std::lock_guard<std::mutex> lock(mtx);
        data_ready = true; // 修改共享状态
    } // 尽早释放锁

    std::cout << "主线程：数据准备完毕，唤醒工作线程。\n";
    cv.notify_one(); // 唤醒一个等待的线程
}
```
核心逻辑：用 `cv.wait(lock, predicate)` 将“等待 + 条件检查”合并，既避免丢失唤醒，也能抵御虚假唤醒。执行结果是工作线程只在数据就绪后继续执行。

#### 2.2 Future/Promise 模型

`std::promise` 和 `std::future` 是一次性的跨线程数据通道。

```cpp
// 后厨线程
void calculate_result(std::promise<int> prom) {
    std::this_thread::sleep_for(std::chrono::seconds(2)); // 模拟复杂计算
    std::cout << "计算线程：计算完成，填入结果 42\n";
    prom.set_value(42); // 将数据放进 promise
}

void consumer_thread() {
    std::promise<int> prom;
    // 从 promise 获取对应的 future 取餐器
    std::future<int> fut = prom.get_future();

    // 启动线程，把 promise 移动进去(因为 promise 不能拷贝)
    std::thread t(calculate_result, std::move(prom));

    std::cout << "主线程：正在做其他事情，等待结果...\n";

    // 调用 future.get() 会阻塞当前线程，直到 promise 里有数据为止
    // 注意：get() 只能调用一次
    int result = fut.get();
    std::cout << "主线程：拿到结果 -> " << result << "\n";

    t.join();
}
```
核心逻辑：生产者线程用 `promise.set_value()` 传递结果，消费者线程通过 `future.get()` 阻塞等待。执行结果是主线程可以在后台计算时继续处理其他任务。

#### 2.3 `std::async` 异步任务执行器

`std::async` 返回 `std::future`，同时隐藏 `std::thread` 的创建和销毁。

启动策略：
- `std::launch::async`：强制异步，系统必须开辟一个新线程去立即执行这个任务。
- `std::launch::deferred`：延迟执行，不开新线程，直到调用 `future.get()` 时，任务才会在当前线程中同步执行。

```cpp
int complex_math() {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return 100;
}

void test_async() {
    // 启动异步任务，明确要求新开线程(std::launch::async)
    std::future<int> result_future = std::async(std::launch::async, complex_math);

    std::cout << "主线程：任务已提交，不用自己管理线程对象\n";

    // 阻塞等待并获取结果
    int value = result_future.get();
    std::cout << "主线程：异步任务结果是 " << value << "\n";
}
```
核心逻辑：指定 `std::launch::async` 保证新线程执行，`future.get()` 阻塞等待结果。执行结果是调用方不再显式管理线程生命周期。

下面补一个常见“坑”的反面例子：临时 `std::future` 的析构会阻塞等待任务完成，导致异步退化为串行。

```cpp
void pitfall_async() {
    std::cout << "主线程：提交异步任务\n";
    // 错误示范：没有用变量接收 future，返回的临时 future 会立即析构
    // 析构函数会阻塞等待 complex_math 执行完毕
    std::async(std::launch::async, complex_math);

    std::cout << "主线程：你以为我立刻执行了？其实我被阻塞了！\n";
}
```
核心逻辑与执行结果：临时生成的 `std::future` 在语句结束时立即析构，而 `async` 产生的 future 析构会阻塞直到异步线程结束。执行结果是主线程必须等待 `complex_math` 跑完才能打印下一句话，异步失效。

### 3. 无锁编程

当锁成为系统的性能瓶颈时，我们需要下探到硬件指令级别的无锁(Lock-Free) 编程。

* **核心知识点：**
    * **原子类型：** `std::atomic` 的工作原理，理解它不仅保证操作的不可分割性，还影响内存可见性。
    * **CAS 核心机制：** 熟练掌握 Compare-And-Swap 原语，深入对比 `compare_exchange_weak`（容易假失败，通常放循环里）与 `compare_exchange_strong` 的应用场景。
    * **ABA 问题：** 理解无锁数据结构中经典的 ABA 缺陷及其解决方案（如 Hazard Pointers 或带有版本号的指针）。
* **代码示例解析(无锁栈的压栈操作)：**

先用一个生活化类比理解 ABA：你有一个水杯(指针)，里面装着可乐(A)。你转身离开时，室友把可乐喝了换成雪碧(B)，后来觉得不好意思，又换成了一杯新的可乐(新的 A)。你转头回来看到杯子里还是可乐(CAS 只检查指针地址没变)，就以为没人动过，一口喝下去。结果“看起来没变，实际已被替换”，这就是 ABA 的本质。

```cpp
#include <atomic>

template<typename T>
class LockFreeStack {
private:
    struct Node {
        T data;
        Node* next;
        Node(const T& data) : data(data), next(nullptr) {}
    };
    std::atomic<Node*> head;

public:
    void push(const T& data) {
        Node* new_node = new Node(data);
        // 核心 CAS 逻辑：
        // 1. 读取当前的 head 给 new_node->next
        new_node->next = head.load();
        // 2. 如果当前的 head 还是刚才读到的 new_node->next，就把 head 更新为 new_node
        // 3. 如果这期间有其他线程修改了 head，CAS 失败，自动将最新的 head 赋值给 new_node->next，然后重试循环
        while (!head.compare_exchange_weak(new_node->next, new_node)) {
            // 循环直到替换成功
            // compare_exchange_weak 允许假失败，通常在循环里用；在弱内存序架构上更高效
        }
    }
};
```
核心逻辑：使用 CAS(Compare-And-Swap) 在无锁条件下更新 `head` 指针，失败则重试。执行结果是在高并发压栈时减少锁竞争，但需要额外关注 ABA 等问题。

### 4. 性能优化：内存模型与缓存行

这是极具挑战性的一章。你需要跳出语言本身，从 CPU 缓存和指令重排的角度思考并发。

* **核心知识点：**
    * **指令重排序：** 编译器优化重排与 CPU 执行重排，理解为什么单线程正确的代码在多线程下会崩溃。
    * **Memory Order(内存序)：** 精准掌握 `memory_order_relaxed`、`acquire`、`release`、`acq_rel` 以及默认的 `seq_cst`(顺序一致性)。学会使用 Acquire-Release 语义来建立跨线程的 happens-before 关系。
    * **伪共享(False Sharing)：** 理解 CPU 缓存行(Cache Line) 的工作原理。掌握如何使用 C++11 的 `alignas` 关键字对齐数据，避免不同线程频繁修改同一缓存行造成的性能雪崩。

#### 4.1 指令重排序

- 编译器优化：为了更好利用寄存器，调整语句执行顺序。
- CPU 乱序执行：把不相互依赖的机器指令放到不同流水线并行执行。

#### 4.2 C++ 内存模型与 `std::memory_order`

- `std::memory_order_seq_cst`：`std::atomic` 的默认内存序，不仅保证原子性，还提供全序一致性。代价是更强的内存屏障与可能的性能损失。
- Acquire-Release 语义：`std::memory_order_release` 用于写，保证写之前的内存操作不被重排到写之后；`std::memory_order_acquire` 用于读，保证读之后的内存操作不被重排到读之前。
- `std::memory_order_relaxed`：只保证原子性，不为其他内存操作建立顺序关系。

#### 4.3 伪共享与缓存行

CPU 读取内存不是按字节，而是按缓存行(Cache Line，常见 64 字节)成块读取。伪共享就像两个同桌共用一张桌子写作业：A 写数学、B 写语文(不同变量)，但只要 A 猛烈摇晃桌子(修改数据导致缓存失效)，B 也不得不停笔重整。使用 `alignas` 就是给他们中间劈开，一人一张桌子。

### 5. 线程池设计与实现

理论必须落地于工程。你需要亲手撸一个健壮的现代 C++ 线程池，它要具备以下特性：

* 使用 `std::vector<std::thread>` 管理工作线程。
* 使用 `std::queue` 配合互斥锁和条件变量构建任务队列。
* 利用变参模板(Variadic Templates)、`std::bind` / Lambda、`std::future` 实现任意参数和返回值的任务提交机制。

```cpp
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <future>
#include <functional>
#include <memory>

class ThreadPool {
public:
    explicit ThreadPool(size_t threads) : stop(false) {
        for (size_t i = 0; i < threads; ++i) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(this->queue_mutex);
                        // 阶段二知识：cv.wait 配合 Lambda 谓词处理虚假唤醒
                        this->condition.wait(lock, [this] {
                            return this->stop || !this->tasks.empty();
                        });
                        if (this->stop && this->tasks.empty()) return;
                        task = std::move(this->tasks.front());
                        this->tasks.pop();
                    }
                    task(); // 执行任务
                }
            });
        }
    }

    // 重点：泛型任务提交(支持任意参数和返回值)
    template<class F, class... Args>
    auto enqueue(F&& f, Args&&... args)
        -> std::future<typename std::invoke_result<F, Args...>::type>
    {
        using return_type = typename std::invoke_result<F, Args...>::type;

        // 阶段二知识：使用 packaged_task 包装任务，以便获取 future
        auto task = std::make_shared<std::packaged_task<return_type()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );

        std::future<return_type> res = task->get_future();
        {
            std::unique_lock<std::mutex> lock(queue_mutex);
            if (stop) throw std::runtime_error("enqueue on stopped ThreadPool");
            // 将任务包装成 void() 类型存入队列
            tasks.emplace([task]() { (*task)(); });
        }
        condition.notify_one();
        return res;
    }

    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(queue_mutex);
            stop = true;
        }
        condition.notify_all();
        for (std::thread& worker : workers) worker.join();
    }

private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queue_mutex;
    std::condition_variable condition;
    bool stop;
};
```
为了让 `enqueue` 更好理解，可以拆成三步：
1) 类型推导：用 `std::invoke_result` 在编译期推导 `F(Args...)` 的返回值类型。
2) 任务打包：通过 `std::bind` 和 `std::forward` 绑定参数，塞进 `std::packaged_task`，再用一层 Lambda 适配成 `void()`。
3) 提取期票：先 `get_future()` 返回给调用者，再把任务推进队列并唤醒工作线程。
核心逻辑：工作线程在条件变量上等待任务，`enqueue` 用 `packaged_task` 包装可调用对象并返回 `future`。执行结果是调用方只需要提交任务并获取结果，不关心线程管理细节。

---

## 常见问题(FAQ)

1. `std::async` 什么时候会退化成同步执行？当使用 `std::launch::deferred` 或未指定策略时，标准允许延迟到 `get()` 时在当前线程执行。
2. 为什么 `memory_order_relaxed` 不是“随便重排”？它只保证原子变量自身的读写原子性，但不会为其他内存操作建立顺序关系。
3. `std::scoped_lock` 会不会锁同一个互斥量两次？这属于未定义行为，应确保传入互斥量互不相同。

## 可落地练习

1. 用 `std::shared_mutex` 实现一个“读多写少”的配置中心，并用 4 读 1 写线程压测读性能。
2. 将线程池改成支持“优先级任务”，对比高优先级任务的平均延迟。
3. 用 `std::atomic` 实现一个无锁计数器，对比 `std::mutex` 版本的吞吐。

## 扩展阅读

1. C++ 内存模型与 happens-before 规则
2. `std::shared_mutex` vs `std::shared_timed_mutex` 的性能差异
3. 线程池中的 work-stealing 策略
