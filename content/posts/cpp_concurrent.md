+++
date = '2026-04-15T13:40:38+08:00'
title = 'Cpp_concurrent'
draft = false
+++

没问题！将火力集中在“并发编程基础”与“进阶并发控制与性能调优”上，是非常聪明的求职策略。在 C++ 后端面试中，这部分不仅是考察频率极高的“重灾区”，更是区分“普通 C++ 程序员”和“具备核心竞争力的后端开发”的分水岭。

我们直接切入正题，为你制定一份深度聚焦多线程与并发调优的学习计划。

---

## 深度聚焦：C++ 多线程与并发调优专属学习计划

我们将这个大板块细化为五个循序渐进的阶段，从 API 使用深入到 CPU 硬件级别的性能优化。

### 第一阶段：并发基石与高级互斥（重点掌握锁的粒度与选择）

熟练使用基础线程和锁只是起点，后端开发更看重在复杂场景下如何选择合适的锁以最大化并发度。

* **核心知识点：**
    * **RAII 锁进阶：** 深入对比 `std::lock_guard`（轻量级、不可移动）与 `std::unique_lock`（灵活、支持延迟锁定和时间锁定、可移动）的底层实现差异。
    * **读写锁的应用：** 掌握 C++14 引入的 `std::shared_mutex` 和 `std::shared_lock`，理解在“读多写少”的高并发场景下，它为何比普通的互斥锁性能更好。
    * **死锁的系统性防范：** 掌握按固定顺序加锁的原则，以及如何使用 C++17 的 `std::scoped_lock` 一次性安全获取多个锁，避免哲学家就餐问题。
* **面试高频：** “如何设计一个线程安全的单例模式？”（考察 Double-Checked Locking 与 `std::call_once`）。

#### 1.1 RAII锁
不要手动调用 `mtx.lock()` 和 `mtx.unlock()`, 使用RAII机制来管理锁，保证异常安全。
- std::lock_guard : 创建时立即加锁，离开作用域时自动解锁，不能被赋值，也不能被移动。只需要再一个作用域内简单保护临界区。
- std::unique_lock : 内部维护一个状态标志。支持延迟加锁 `std::defer_lock`, 尝试加锁 `try_lock` 和提前手动解锁 `unlock`，并且支持移动语义。可以与条件变量`std::condition_variable` 配合使用（条件变量需要等待时释放锁）， 或需要在作用域中提前解锁以减小锁粒度。

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
        
        lock.unlock(); // 提前手动解锁，非常关键！减小锁粒度！
        
        // 模拟耗时的 IO 或计算操作，此时不需要霸占锁
        std::cout << "执行耗时非共享操作..." << std::endl; 
        
        lock.lock(); // 如果后续还需要访问共享资源，可以再次加锁
        shared_data--;
    }
}
```

#### 1.2 读写锁：高并发读多写少
C++14 引入 `std::shared_mutex`，允许多个线程同时读，但写操作是独占的。
```cpp
class ConfigManager {
private:
    std::map<std::string, std::string> config_;
    mutable std::shared_mutex rw_mtx_; // mutable 允许在 const 成员函数中加锁

public:
    // 读操作：允许多个线程同时进入
    std::string getConfig(const std::string& key) const {
        // 使用 shared_lock 加上“共享锁（读锁）”
        std::shared_lock<std::shared_mutex> lock(rw_mtx_);
        auto it = config_.find(key);
        return it != config_.end() ? it->second : "";
    }

    // 写操作：独占访问，阻塞其他所有的读和写
    void setConfig(const std::string& key, const std::string& value) {
        // 使用 unique_lock 加上“排他锁（写锁）”
        std::unique_lock<std::shared_mutex> lock(rw_mtx_);
        config_[key] = value;
    }
};
```

#### 1.3 std::scoped_lock与死锁防范
C++17 引入 `std::scoped_lock` 可以一次性锁定多个互斥量，避免死锁。

```cpp
class Account {
public:
    explicit Account(int b) : balance(b) {}
    std::mutex m;
    int balance;
};

void transfer(Account& from, Account& to, int amount) {
    // C++17 的 std::scoped_lock，可以接收任意数量的 mutex
    // 它保证了原子性地获取所有锁，绝不会发生死锁！
    std::scoped_lock lock(from.m, to.m); 

    if (from.balance >= amount) {
        from.balance -= amount;
        to.balance += amount;
        std::cout << "转账成功\n";
    }
}
```

### 第二阶段：高级同步机制与异步编程（摆脱纯手动的阻塞等待）

在现代 C++ 中，很多时候我们不需要直接操作 `std::thread` 和条件变量，基于任务的异步编程是更优雅的选择。

* **核心知识点：**
    * **条件变量避坑指南：** 彻底弄懂虚假唤醒（Spurious Wakeup）和丢失唤醒（Lost Wakeup），熟练编写带有 Predicate（谓词）的 `cv.wait()`。
    * **Future/Promise 模型：** 掌握 `std::promise`、`std::future` 以及 `std::shared_future`，实现线程间的数据通道。
    * **异步任务执行器：** 掌握 `std::async`，理解其启动策略（`std::launch::async` vs `std::launch::deferred`）对系统资源的影响。

#### 2.1 条件变量
`std::condition_varialbe` 线程之间同步。
- 虚假唤醒 ： A线程发出唤醒信号`notify`,但此时线程B还没进入睡眠等待`wait`，这个信号会凭空消失。
- 虚假唤醒 ： 操作系统底层的异常机制如时间片轮转可能导致即使没有其它线程调用`notify`，处于睡眠状态的线程也可能被意外唤醒，如果醒来之后不检查条件直接向下执行，程序会崩溃。 循环搭配 `wait()`

```cpp
std::mutex mtx;
std::condition_variable cv;
bool data_ready = false; // 共享状态标志

void worker_thread() {
    std::unique_lock<std::mutex> lock(mtx); // 条件变量必须配合 unique_lock
    
    std::cout << "工作线程：等待数据准备...\n";
    
    // 【核心重点】wait 的第二个参数传入一个 Lambda 表达式（谓词 Predicate）
    // 它的底层等价于： while (!data_ready) { cv.wait(lock); }
    // 这完美解决了“虚假唤醒”（醒来后如果 data_ready 为 false 会继续睡）
    // 也解决了“丢失唤醒”（如果 data_ready 已经是 true，就不会去睡了）
    cv.wait(lock, []{ return data_ready; });
    
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


#### 2.2 Future/Promise模型
`std::promise`和`std::future`是一种一次性的跨线程数据通道。
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
    
    // 启动线程，把 promise 移动进去（因为 promise 不能拷贝）
    std::thread t(calculate_result, std::move(prom));
    
    std::cout << "主线程：正在做其他事情，等待结果...\n";
    
    // 调用 future.get() 会阻塞当前线程，直到 promise 里有数据为止
    // 注意：get() 只能调用一次！
    int result = fut.get(); 
    std::cout << "主线程：拿到结果 -> " << result << "\n";
    
    t.join();
}
```

#### 2.3 std::async 异步任务执行器
`std::async`返回一个`std::future`，同时隐藏一个线程`std::thread`的创建和销毁。
启动策略：
- `std::launch::async`: 强制异步，系统必须开辟一个新线程去立即执行这个任务。
- `std::launch::defered`: 延迟执行，不开新线程，直到调用`future.get()`时，任务才会在当前调用`get()`的线程中同步执行。
```cpp
int complex_math() {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return 100;
}

void test_async() {
    // 启动异步任务，明确要求新开线程（std::launch::async）
    std::future<int> result_future = std::async(std::launch::async, complex_math);
    
    std::cout << "主线程：任务已提交，不用自己管理线程对象\n";
    
    // 阻塞等待并获取结果
    int value = result_future.get();
    std::cout << "主线程：异步任务结果是 " << value << "\n";
}
```



### 第三阶段：原子操作与无锁编程初探（高并发的利器）

当锁成为系统的性能瓶颈时，我们需要下探到硬件指令级别的无锁（Lock-Free）编程。

* **核心知识点：**
    * **原子类型：** `std::atomic` 的工作原理，理解它不仅保证了操作的不可分割性，还影响着内存的可见性。
    * **CAS 核心机制：** 熟练掌握 Compare-And-Swap 原语，深入对比 `compare_exchange_weak`（容易假失败，通常放循环里）与 `compare_exchange_strong` 的应用场景。
    * **ABA 问题：** 理解无锁数据结构中经典的 ABA 缺陷及其解决方案（如 Hazard Pointers 或带有版本号的指针）。
* **代码示例解析（无锁栈的压栈操作）：**

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
        }
    }
};
```

### 第四阶段：C++ 内存模型与硬件级调优（大厂硬核考察点）

这是极具挑战性的一章。你需要跳出语言本身，从 CPU 缓存和指令重排的角度思考并发。

* **核心知识点：**
    * **指令重排序：** 编译器优化重排与 CPU 执行重排，理解为什么单线程正确的代码在多线程下会崩溃。
    * **Memory Order（内存序）：** 精准掌握 `memory_order_relaxed`、`acquire`、`release`、`acq_rel` 以及默认的 `seq_cst`（顺序一致性）。学会使用 Acquire-Release 语义来建立跨线程的 happens-before 关系。
    * **伪共享 (False Sharing)：** 理解 CPU 缓存行 (Cache Line) 的工作原理。掌握如何使用 C++11 的 `alignas` 关键字对齐数据，避免不同线程频繁修改同一缓存行造成的性能雪崩。

#### 4.1指令重排序


- 编译器优化: 为了更好利用寄存器，偷偷打乱代码执行顺序。
- CPU乱序执行: 把不相互依赖的机器指令放到不同的流水线中并发执行。

#### 4.2C++内存模型与 std::memory_order
- `std::memory_order_deq_cst`: `std::atomix`的默认内存序，不仅保证当前变量的原子性，还会在变量前后强行加上内存屏障，禁止任何重排序，保证所有线程看到的所有操作顺序是一样的。但是频繁清空CPU流水线和同步缓存，性能损失极大。
- `acruire-release`语义: memorey_order_release : 用于写操作，保证这一行代码之前的所有内存读写操作，不允许被重排到这一行之后。  memorey_order_acquire : 用于读操作，保证这一行代码之后的所有内存读写操作，不允许被重排到这一行之前。
- `memory_order_relaxed` 松散模型: 只保证变量本身的读取写入是原子的，完全不干涉指令重排。

#### 4.3 伪共享与缓存行

使用`alignas`强制内存对齐。

### 第五阶段：工程实战——工业级线程池设计

理论必须落地于工程。你需要亲手撸一个健壮的现代 C++ 线程池，它要具备以下特性：

* 使用 `std::vector<std::thread>` 管理工作线程。
* 使用 `std::queue` 配合互斥锁和条件变量构建任务队列。
* 利用变参模板 (Variadic Templates)、`std::bind` / Lambda、`std::future` 实现任意参数和返回值的任务提交机制。

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

    // 重点：泛型任务提交（支持任意参数和返回值）
    template<class F, class... Args>
    auto enqueue(F&& f, Args&&... args) 
        -> std::future<typename std::result_of<F(Args...)>::type> 
    {
        using return_type = typename std::result_of<F(Args...)>::type;

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
        for (std::thread &worker : workers) worker.join();
    }

private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queue_mutex;
    std::condition_variable condition;
    bool stop;
};
```