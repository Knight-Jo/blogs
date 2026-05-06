+++
date = '2026-03-04T15:19:33+08:00'
draft = false
title = 'Async_logger'
+++



## 异步日志系统

本文介绍一种高性能异步日志系统的设计与实现，核心思想是将日志写入分为前端和后端两部分，前端负责快速地将日志消息写入内存缓冲区，后端线程负责定期将缓冲区中的日志批量写入磁盘。
设计双缓冲机制:
- 前端缓冲： 包含 currentBuffer_(当前写入) 和 nextBuffer_(预分配)
- 后端队列: buffers_ 存储待写入磁盘的缓冲区列表
- 数据流向：前端 append -> 内存缓冲(currentBuffer_) -> 后端交换缓冲(buffers_) -> 磁盘写入

通过交换缓冲指针，前端和后端几乎不需要等待对方，极大提升了日志系统的吞吐量和响应速度。

### 前端核心逻辑

业务线程使用 append 将日志行写入内存缓冲，不被磁盘 I/O 卡住，真正的磁盘写入交给后台线程完成,保证前端低延迟。

```cpp
// 调用此函数解决前端把LOG_XXX<<"..."传递给后端，后端再将日志消息写入日志文件
void AsyncLogging::append(const char *logline, int len)
{
    // - 进入临界区，保护共享的缓冲指针和队列。
    // - 这里的锁很短，只包含内存操作，所以开销可控。
    std::lock_guard<std::mutex> lg(mutex_);

    // 快路径，缓冲区剩余的空间充足,直接追加
    if (currentBuffer_->avail() > static_cast<size_t>(len)) { 
        // - 检查当前大缓冲是否还有足够空间。
        // - 如果能放下，就直接把日志追加进去（纯内存拷贝）。
        currentBuffer_->append(logline, len);
    } else {
        // 慢路径，当前缓冲区满了，需要切换缓冲区。
        buffers_.push_back(std::move(currentBuffer_));

        // 优先使用预分配的 nextBuffer_，避免频繁分配内存带来的性能抖动。
        if (nextBuffer_) {
            currentBuffer_ = std::move(nextBuffer_);
        } else {
            currentBuffer_.reset(new LargeBuffer);
        }
        currentBuffer_->append(logline, len);
        // - 通知后端写线程：队列里有可写数据了。
        cond_.notify_one();
    }
}
```
- 锁低开销: 锁内只做内存操作,指针移动和内存拷贝，不设计系统I/O调用
- 平滑性能抖动： 备用缓冲区缓冲区切换，降低内存分配带来的性能波动

### 后台线程写逻辑
后端线程通过交换缓冲队列实现批量写盘，极大提高了吞吐量

```cpp
void AsyncLogging::threadFunc()
{
    // output 写入磁盘接口, 负责滚动与落盘
    LogFile output(basename_, rollSize_);
    
    // 预分配复用池：交换时几乎不需要动态分配内存，降低抖动
    BufferPtr newbuffer1(new LargeBuffer); 
    BufferPtr newbuffer2(new LargeBuffer); 

    newbuffer1->bzero();
    newbuffer2->bzero();
    
    BufferVector buffersToWrite;
    buffersToWrite.reserve(16); // 预留足够空间，用于和前端 buffers_ 交换

    while (running_) {
        {
            std::unique_lock<std::mutex> lg(mutex_);
            if (buffers_.empty()) {
                // 如果前端没有新数据，等待条件变量或超时唤醒
                // 超时唤醒机制保证了日志定期 flush，避免低频日志长时间滞留内存
                cond_.wait_for(lg, std::chrono::seconds(3));
            }
            buffers_.push_back(std::move(currentBuffer_));
            
            // 立刻给前端换上一个空缓冲，确保前端不必等待后端写盘
            currentBuffer_ = std::move(newbuffer1);
            if (!nextBuffer_) {
                nextBuffer_ = std::move(newbuffer2);
            }
            
            // 关键设计：通过 swap 转移所有权，锁内只交换指针，不做拷贝。
            // 此后锁释放，前端可继续写入，后端在锁外执行昂贵的磁盘 I/O。
            buffersToWrite.swap(buffers_);
        }

        // 锁外操作：遍历待写队列，通过 LogFile 接口真正写入磁盘
        for (auto &buffer : buffersToWrite) {
            output.append(buffer->data(), buffer->length());
        }

        // 防止后端积压过多导致内存暴涨，丢弃多余的缓冲（极少发生）
        if (buffersToWrite.size() > 2) {
            buffersToWrite.resize(2);
        }
        
        // 缓冲复用池思想：
        // 把写完的缓冲 reset 成空缓冲，再作为 newbuffer1/newbuffer2 备用。
        // 避免了循环中的 new/delete，提升系统长期运行的稳定性。
        if (!newbuffer1) {
            newbuffer1 = std::move(buffersToWrite.back());
            buffersToWrite.pop_back();
            newbuffer1->reset();
        }
        if (!newbuffer2) {
            newbuffer2 = std::move(buffersToWrite.back());
            buffersToWrite.pop_back();
            newbuffer2->reset();
        }
        
        buffersToWrite.clear(); // 清空后端本地缓冲队列
        output.flush();         // 触发系统层面的刷盘
    }
    output.flush(); // 线程退出前，确保残留数据落盘
}

```

### 滚动与 flush

- 按大小滚动： 达到 rollsize 阈值触发新文件。
- 按时间滚动： 跨天时触发新文件。
- 按间隔 flush: 定期落盘，平衡性能与可靠性。

```cpp
void LogFile::appendInlock(const char *data, int len)
{
    file_->append(data, len);

    time_t now = time(NULL); // 当前时间
    ++count_;

    // 1. 判断是否需要滚动日志
    if (file_->writtenBytes() > rollsize_) {
        rollFile();
    } else if (count_ >= checkEveryN_){ // 达到写入次数阈值后，进行检查
        count_ = 0;

        // 基于时间周期滚动日志
        time_t thisPeriod = now / kRollPerSeconds_ * kRollPerSeconds_;
        if (thisPeriod != startOfPeriod_) {
            rollFile();
        }
    }

    // 2. 判断是否需要刷新日志（独立的刷新逻辑）
    if (now - lastFlush_ > flushInterval_) {
        lastFlush_ = now;
        file_->flush();
    }
}
```

两条刷新路径
- logfile 层 每次写入检查 flshInterval_
- asynclogging层，后端线程没滚写完执行output.flush(),以及一个超时刷新，避免长时间不落盘。

### 可靠性设计 
日志系统的可靠性设计目标是**尽量保证**日志消息不丢失，即使在异常情况下也能最大程度地保留日志数据。

当前实现中的“尽量保证”包括：
- **定期 flush**：`flushInterval_` 控制写入落盘的最大延迟。
- **后端循环刷盘**：每轮写入后立刻刷新，减少缓冲滞留。
- **FATAL 时强制 flush**：Logger 遇到 `FATAL` 会调用 flush 并终止进程，尽量把最后日志写出去。

仍可能丢失的场景：
- 进程被 `SIGKILL` 强制杀死，无法执行 flush。
- 机器掉电或系统崩溃，内存缓冲未写入磁盘。

如果要进一步降低丢失风险，可以考虑：
- 缩短 `flushInterval_`（代价是性能下降）。
- 在关键路径手动调用 flush。
- 重要日志使用同步写入或单独通道。

### 性能测试
使用脚本：

```bash
scripts/run_log_tests.sh bench
scripts/run_log_tests.sh stress
```

输出字段含义：
- throughput_mibps：吞吐量（MiB/s）
- avg_append_us：append 平均耗时（微秒）
- duration_s：运行时间（秒）
- total_mib：写入总量（MiB）

测试环境  windows10 wsl
cpu :  i9-11900F
内存 : 64 G
磁盘： ZHITAI TiPlus7100 1TB 
消息数： 200000
单条大小 : 256 Bytes
flush: 1s
rollsize : 256MB

| 用例 | 线程数 |  时长(s) | 总写入(MiB) | 吞吐(MiB/s) | avg_append(us) | 备注 |
| --- | --- | --- | --- | --- | --- | --- |
| | | | | | | |
| 1 | 1 | 0.0167828 | 48.8281 | 2909.42 | 0.180925 | |
| 2 | 2 | 0.0615902 | 97.6562 | 1585.58 | 0.69876 | |
| 3 | 4 | 0.137376s | 195.312 | 1421.74 | 2.20374 |
| 4 | 8 | 0.465275 | 390.625 | 839.558  | 4.91862 | 
| 5 | 16 | 1.52033 | 781.25 | 513.87  | 10.5288 | 
| 6 | 32 | 2.11957 | 1562.5 | 737.178  | 15.4019 | 

对比同步日志系统
| 用例 | 线程数 | 总写入(MiB) |  同步时长(s) | 同步吞吐(MiB/s) | 异步时长(s) | 异步吞吐(MiB/s) | 吞吐量提高| 备注 |
| --- | --- | --- |  --- | --- | --- | --- | --- | --- |
| 1 | 1  | 48.8281 | 0.0422227 |  1156.44 | 0.0167737 | 2910.99 | 2.5172 | 
| 1 | 2  | 97.6562 | 0.130061 |  750.848 | 0.0509452 | 1916.89 | 2.72876 | 
| 1 | 4  | 195.312 | 0.366934 |  532.282 | 0.152887| 1277.5 | 2.40004 | 
| 1 | 8  | 390.625 | 0.860877 | 453.752 | 0.482434 | 809.697 | 1.78445 |
| 1 | 16 | 781.25  | 2.41902 | 322.961 | 1.40663 | 555.405 | 1.71973 |
| 1 | 32 | 1562.5  | 5.02933 | 310.677  | 2.10941 | 740.73 | 2.38424|

异步日志吞吐量均为同步系统的 1.7～2.7 倍，优势显著。

随着线程增加，两者的吞吐均呈下降趋势，但异步系统的降速更缓，尤其在 32 线程时异步吞吐回升明显，表明其缓冲交换与复用策略有效缓解了高负载下的锁竞争。



