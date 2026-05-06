+++
date = '2026-05-04T15:19:33+08:00'
draft = true
title = 'Async_logger'
+++



## 异步日志系统


### 前端核心逻辑

业务线程只做内存写入，不被磁盘 I/O 卡住，真正的磁盘写入交给后台线程完成。

```cpp
// 调用此函数解决前端把LOG_XXX<<"..."传递给后端，后端再将日志消息写入日志文件
void AsyncLogging::append(const char *logline, int len)
{
    // - 进入临界区，保护共享的缓冲指针和队列。
    // - 这里的锁很短，只包含内存操作，所以开销可控。
    std::lock_guard<std::mutex> lg(mutex_);

    // 缓冲区剩余的空间足够写入
    if (currentBuffer_->avail() > static_cast<size_t>(len))
    { 
        // - 检查当前大缓冲是否还有足够空间。
        // - 如果能放下，就直接把日志追加进去（纯内存拷贝）。
        currentBuffer_->append(logline, len);
    }
    else
    {
        // - 如果放不下，说明当前缓冲“满了”。
        // - 把这个满缓冲移动进队列，交给后端写线程统一写盘。
        buffers_.push_back(std::move(currentBuffer_));

        if (nextBuffer_)
        {
            // - 尝试用“备用缓冲”顶上来继续写。
            // - 这样前端可以立刻继续写日志，不需要等待后端写盘。
            currentBuffer_ = std::move(nextBuffer_);
        }
        else
        {
            currentBuffer_.reset(new LargeBuffer);
        }
        currentBuffer_->append(logline, len);
        // - 通知后端写线程：队列里有可写数据了。
        // - 后端线程被唤醒后会批量写盘，提高吞吐。
        cond_.notify_one();
    }
}
```
- 锁开销只做内存操作，不写盘
- 使用缓冲区切换，保证性能相对稳定

### 后台线程写逻辑
后端线程通过交换缓冲和批量写盘实现高吞吐， 同时把锁的持有时间压缩到最短，保证前端线程几乎只做内存操作。
```cpp

void AsyncLogging::threadFunc()
{
    // output写入磁盘接口, 负责滚动与落盘。
    LogFile output(basename_, rollSize_);
    // 预分配，交换时几乎不需要分配内存，降低抖动
    BufferPtr newbuffer1(new LargeBuffer); // 生成新buffer替换currentbuffer_
    BufferPtr newbuffer2(new LargeBuffer); // 生成新buffer2替换newBuffer_，其目的是为了防止后端缓冲区全满前端无法写入

    newbuffer1->bzero();
    newbuffer2->bzero();
    // 缓冲区数组置为16个，用于和前端缓冲区数组进行交换
    BufferVector buffersToWrite;
    buffersToWrite.reserve(16);

    while (running_)
    {
        {
            // 互斥锁保护这样就保证了其他前端线程无法向前端buffer写入数据
            std::unique_lock<std::mutex> lg(mutex_);
            if (buffers_.empty())
            {
                // - 如果前端没有新数据，等待条件变量或超时唤醒。
                // - 超时唤醒可以保证日志定期 flush，不至于长时间不写盘。
                cond_.wait_for(lg, std::chrono::seconds(3));
            }
            buffers_.push_back(std::move(currentBuffer_));
            // - 立刻给前端换上一个空缓冲。
            // - 前端可以继续写日志，不必等待后端写盘。
            currentBuffer_ = std::move(newbuffer1);
            if (!nextBuffer_)
            {
                nextBuffer_ = std::move(newbuffer2);
            }
            // - 用交换把前端队列搬到后端本地，锁内操作很快。
            // - 之后写盘在锁外执行，避免阻塞前端。
            buffersToWrite.swap(buffers_);
        }
        // 从待写缓冲区取出数据通过LogFile提供的接口写入到磁盘中
        for (auto &buffer : buffersToWrite)
        {
            output.append(buffer->data(), buffer->length());
        }

        if (buffersToWrite.size() > 2)
        {
            buffersToWrite.resize(2);
        }
        // 复用空缓冲
        // - 把写完的缓冲 reset 成空缓冲，再作为 newbuffer1/newbuffer2 备用。
        // - 减少重复分配，提高稳定性。
        if (!newbuffer1)
        {
            newbuffer1 = std::move(buffersToWrite.back());
            buffersToWrite.pop_back();
            newbuffer1->reset();
        }
        if (!newbuffer2)
        {
            newbuffer2 = std::move(buffersToWrite.back());
            buffersToWrite.pop_back();
            newbuffer2->reset();
        }
        buffersToWrite.clear(); // 清空后端缓冲队列
        output.flush();         // 清空文件夹缓冲区
    }
    output.flush(); // 确保一定清空。
}
```

### 滚动与 flush

- 按大小滚动： 达到 rollsize 触发新文件。
- 按时间滚动： 跨天时触发新文件。
- 按间隔 flush: 定期罗盘，平衡性能与可靠性。

```cpp
void LogFile::appendInlock(const char *data, int len)
{
    file_->append(data, len);

    time_t now = time(NULL); // 当前时间
    ++count_;

    // 1. 判断是否需要滚动日志
    if (file_->writtenBytes() > rollsize_)
    {
        rollFile();
    }
    else if (count_ >= checkEveryN_) // 达到写入次数阈值后，进行检查
    {
        count_ = 0;

        // 基于时间周期滚动日志
        time_t thisPeriod = now / kRollPerSeconds_ * kRollPerSeconds_;
        if (thisPeriod != startOfPeriod_)
        {
            rollFile();
        }
    }

    // 2. 判断是否需要刷新日志（独立的刷新逻辑）
    if (now - lastFlush_ > flushInterval_)
    {
        lastFlush_ = now;
        file_->flush();
    }
}
```
日志写入非常频繁，如果每次写入都检查时间周期或做系统调用，带来CPU开销，这种开销是可避免的。
把按时间滚动的检查从“每条日志依次”降到 “每 N 条一次”。

两条刷新路径
- logfile 层 每次写入检查 flshInterval_
- asynclogging层，后端线程没滚写完执行output.flush(),以及一个超时刷新，避免长时间不落盘。

### 异常错误 
只能尽量降低丢失概率。

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
rollsize : 268435456

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




