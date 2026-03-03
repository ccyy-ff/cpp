# 中心缓存(CentralCache)

# 中心缓存介绍

## CentralCache的定位和作用

```cpp
// 使用无锁的span信息存储
struct SpanTracker {
    std::atomic<void*> spanAddr{nullptr};
    std::atomic<size_t> numPages{0};
    std::atomic<size_t> blockCount{0};
    std::atomic<size_t> freeCount{0}; // 用于追踪spn中还有多少块是空闲的，如果所有块都空闲，则归还span给PageCache
};

class CentralCache
{
public:
    static CentralCache& getInstance()
    {
        static CentralCache instance;
        return instance;
    }

    void* fetchRange(size_t index);
    void returnRange(void* start, size_t size, size_t index);

private:
    // 相互是还所有原子指针为nullptr
    CentralCache();
    // 从页缓存获取内存
    void* fetchFromPageCache(size_t size);

    // 获取span信息
    SpanTracker* getSpanTracker(void* blockAddr);

    // 更新span的空闲计数并检查是否可以归还
    void updateSpanFreeCount(SpanTracker* tracker, size_t newFreeBlocks, size_t index);

private:
    // 中心缓存的自由链表
    std::array<std::atomic<void*>, FREE_LIST_SIZE> centralFreeList_;

    // 用于同步的自旋锁
    std::array<std::atomic_flag, FREE_LIST_SIZE> locks_;
    
    // 使用数组存储span信息，避免map的开销
    std::array<SpanTracker, 1024> spanTrackers_;
    std::atomic<size_t> spanCount_{0};

    // 延迟归还相关的成员变量
    static const size_t MAX_DELAY_COUNT = 48;  // 最大延迟计数
    std::array<std::atomic<size_t>, FREE_LIST_SIZE> delayCounts_;  // 每个大小类的延迟计数
    std::array<std::chrono::steady_clock::time_point, FREE_LIST_SIZE> lastReturnTimes_;  // 上次归还时间
    static const std::chrono::milliseconds DELAY_INTERVAL;  // 延迟间隔

    bool shouldPerformDelayedReturn(size_t index, size_t currentCount, std::chrono::steady_clock::time_point currentTime);
    void performDelayedReturn(size_t index);
};
```

主要作用：

* 作为`ThreadCache`和`PageCache`之间的中间层
* 管理从`PageCache`获取的内存块
* 为多个`ThreadCache`提供内存分配服务
* 实现内存的跨线程复用

## 核心实现原理

### 从中心缓存获取内存块返回给线程本地缓存

```cpp
void* CentralCache::fetchRange(size_t index)
{
    // 索引检查，当索引大于等于FREE_LIST_SIZE时，说明申请内存过大应直接向系统申请
    if (index >= FREE_LIST_SIZE) 
        return nullptr;

    // 自旋锁保护
    while (locks_[index].test_and_set(std::memory_order_acquire))
    {
        std::this_thread::yield(); // 添加线程让步，避免忙等待，避免过度消耗CPU
    }

    void* result = nullptr;
    try 
    {
        // 尝试从中心缓存获取内存块
        result = centralFreeList_[index].load(std::memory_order_relaxed);

        if (!result)
        {
            // 如果中心缓存为空，从页缓存获取新的内存块
            size_t size = (index + 1) * ALIGNMENT;
            result = fetchFromPageCache(size);

            if (!result)
            {
                locks_[index].clear(std::memory_order_release);
                return nullptr;
            }

            // 将获取的内存块切分成小块
            char* start = static_cast<char*>(result);

            // 计算实际分配的页数
            size_t numPages = (size <= SPAN_PAGES * PageCache::PAGE_SIZE) ? 
                             SPAN_PAGES : (size + PageCache::PAGE_SIZE - 1) / PageCache::PAGE_SIZE;
            // 使用实际页数计算块数
            size_t blockNum = (numPages * PageCache::PAGE_SIZE) / size;
            
            if (blockNum > 1) 
            {  // 确保至少有两个块才构建链表
                for (size_t i = 1; i < blockNum; ++i) 
                {
                    void* current = start + (i - 1) * size;
                    void* next = start + i * size;
                    *reinterpret_cast<void**>(current) = next;
                }
                *reinterpret_cast<void**>(start + (blockNum - 1) * size) = nullptr;
                
                // 保存result的下一个节点
                void* next = *reinterpret_cast<void**>(result);
                // 将result与链表断开
                *reinterpret_cast<void**>(result) = nullptr;
                // 更新中心缓存
                centralFreeList_[index].store(
                    next, 
                    std::memory_order_release
                );
                
                // 使用无锁方式记录span信息
                // 做记录是为了将中心缓存多余内存块归还给页缓存做准备。考虑点：
                // 1.CentralCache 管理的是小块内存，这些内存可能不连续
                // 2.PageCache 的 deallocateSpan 要求归还连续的内存
                size_t trackerIndex = spanCount_++;
                if (trackerIndex < spanTrackers_.size())
                {
                    spanTrackers_[trackerIndex].spanAddr.store(start, std::memory_order_release);
                    spanTrackers_[trackerIndex].numPages.store(numPages, std::memory_order_release);
                    spanTrackers_[trackerIndex].blockCount.store(blockNum, std::memory_order_release); // 共分配了blockNum个内存块
                    spanTrackers_[trackerIndex].freeCount.store(blockNum - 1, std::memory_order_release); // 第一个块result已被分配出去，所以初始空闲块数为blockNum - 1
                }
            }
        } 
        else 
        {
            // 保存result的下一个节点
            void* next = *reinterpret_cast<void**>(result);
            // 将result与链表断开
            *reinterpret_cast<void**>(result) = nullptr;
            
            // 更新中心缓存
            centralFreeList_[index].store(next, std::memory_order_release);

             // 更新span的空闲计数
            SpanTracker* tracker = getSpanTracker(result);
            if (tracker)
            {
                // 减少一个空闲块
                tracker->freeCount.fetch_sub(1, std::memory_order_release);
            }
        }
    }
    catch (...) 
    {
        locks_[index].clear(std::memory_order_release);
        throw;
    }

    // 释放锁
    locks_[index].clear(std::memory_order_release);
    return result;
}
```

### 将线程本地缓存多余的内存块归还给中心缓存

CentralCache提供给ThreadCache调用的接口

```cpp
void CentralCache::returnRange(void* start, size_t size, size_t index)
{
    if (!start || index >= FREE_LIST_SIZE) 
        return;

    size_t blockSize = (index + 1) * ALIGNMENT;
    size_t blockCount = size / blockSize;    

    while (locks_[index].test_and_set(std::memory_order_acquire)) 
    {
        std::this_thread::yield();
    }

    try 
    {
        // 1. 将归还的链表连接到中心缓存
        void* end = start;
        size_t count = 1;
        while (*reinterpret_cast<void**>(end) != nullptr && count < blockCount) {
            end = *reinterpret_cast<void**>(end);
            count++;
        }
        void* current = centralFreeList_[index].load(std::memory_order_relaxed);
        *reinterpret_cast<void**>(end) = current; // 头插法（将原有链表接在归还链表后边）
        centralFreeList_[index].store(start, std::memory_order_release);
        
        // 2. 更新延迟计数
        size_t currentCount = delayCounts_[index].fetch_add(1, std::memory_order_relaxed) + 1;
        auto currentTime = std::chrono::steady_clock::now();
        
        // 3. 检查是否需要执行延迟归还
        if (shouldPerformDelayedReturn(index, currentCount, currentTime))
        {
            performDelayedReturn(index);
        }
    }
    catch (...) 
    {
        locks_[index].clear(std::memory_order_release);
        throw;
    }

    locks_[index].clear(std::memory_order_release);
}
```

将ThreadCache中多余内存块归还给CentralCache之后，会进行检查是否需要归还内存块给PageCache。

这里的检查归还总体经过两层：

* 首先是returnRange调用次数(也就是ThreadCache归还内存给CentralCache次数)和时间的检查。
* 其次，还要保证CentralCache中要归还的空闲内存块能够拼成完整的内存页（如分配时的内存页一样）。

满足上述两个条件才将CentralCache中的对应空闲内存归还给PageCache。

```cpp
// 检查是否需要执行延迟归还
bool CentralCache::shouldPerformDelayedReturn(size_t index, size_t currentCount, 
    std::chrono::steady_clock::time_point currentTime)
{
    // 基于计数和时间的双重检查
    if (currentCount >= MAX_DELAY_COUNT)
    {
        return true;
    }
    
    auto lastTime = lastReturnTimes_[index];
    return (currentTime - lastTime) >= DELAY_INTERVAL;
}
```

```cpp
// 执行延迟归还
void CentralCache::performDelayedReturn(size_t index)
{
    // 重置延迟计数
    delayCounts_[index].store(0, std::memory_order_relaxed);
    // 更新最后归还时间
    lastReturnTimes_[index] = std::chrono::steady_clock::now();
    
    // 统计每个span的空闲块数
    std::unordered_map<SpanTracker*, size_t> spanFreeCounts;
    void* currentBlock = centralFreeList_[index].load(std::memory_order_relaxed);
    
    while (currentBlock)
    {
        SpanTracker* tracker = getSpanTracker(currentBlock);
        if (tracker)
        {
            spanFreeCounts[tracker]++;
        }
        currentBlock = *reinterpret_cast<void**>(currentBlock);
    }
    
    // 更新每个span的空闲计数并检查是否可以归还
    for (const auto& [tracker, newFreeBlocks] : spanFreeCounts)
    {
        updateSpanFreeCount(tracker, newFreeBlocks, index);
    }
}
```

```cpp
void CentralCache::updateSpanFreeCount(SpanTracker* tracker, size_t newFreeBlocks, size_t index)
{
    size_t oldFreeCount = tracker->freeCount.load(std::memory_order_relaxed);
    size_t newFreeCount = oldFreeCount + newFreeBlocks;
    tracker->freeCount.store(newFreeCount, std::memory_order_release);
    
    // 如果所有块都空闲，归还span
    if (newFreeCount == tracker->blockCount.load(std::memory_order_relaxed))
    {
        void* spanAddr = tracker->spanAddr.load(std::memory_order_relaxed);
        size_t numPages = tracker->numPages.load(std::memory_order_relaxed);
        
        // 从自由链表中移除这些块
        void* head = centralFreeList_[index].load(std::memory_order_relaxed);
        void* newHead = nullptr;
        void* prev = nullptr;
        void* current = head;
        
        while (current)
        {
            void* next = *reinterpret_cast<void**>(current);
            if (current >= spanAddr && 
                current < static_cast<char*>(spanAddr) + numPages * PageCache::PAGE_SIZE)
            {
                if (prev)
                {
                    *reinterpret_cast<void**>(prev) = next;
                }
                else
                {
                    newHead = next;
                }
            }
            else
            {
                prev = current;
            }
            current = next;
        }
        
        centralFreeList_[index].store(newHead, std::memory_order_release);
        PageCache::getInstance().deallocateSpan(spanAddr, numPages);
    }
}
```

## 设计特点

1. 批量管理

```cpp
static const size_t SPAN_PAGES = 8;  // 固定8页的批量申请
```

原因：

* 减少向`PageCache`的请求次数
* 提高内存分配效率
* 降低锁竞争

2. 细粒度锁

* 减少线程竞争
* 提高并发性能
* 避免全局锁的性能瓶颈

## 工作流程

```cpp
// 1. ThreadCache请求内存
void* ThreadCache::fetchFromCentralCache(size_t index) {
    return CentralCache::getInstance().fetchRange(index);
}

// 2. CentralCache处理请求
void* CentralCache::fetchRange(size_t index) {
    // 如果centralFreeList_中有可用内存
    if (result = centralFreeList_[index].load()) {
        return result;
    }
    
    // 否则从PageCache获取新内存
    result = fetchFromPageCache(size);
    // 切分并构建链表
    return result;
}

// 3. 内存回收
void CentralCache::returnRange(void* start, size_t size, size_t index) {
    // 将内存块插入到对应的自由链表
    *reinterpret_cast<void**>(start) = centralFreeList_[index].load();
    centralFreeList_[index].store(start);
}
```

## 为什么这么设计

1. 三级缓存的必要性
   1. ThreadCache：无锁，快速分配
   2. CentralCache：平衡点，内存复用
   3. PageCache：系统对接，大块管理
2. 批量处理的优势
   1. 减少锁竞争
   2. 提高缓存命中率
   3. 降低系统调用开销
3. 内存规格管理

```cpp
size_t size = (index + 1) * ALIGNMENT;  // 8字节对齐
```

原因：

* 减少内存碎片
* 提高内存利用率
* 简化管理逻辑

性能考虑

* 空间效率：批量申请和释放
* 时间效率：细粒度锁和原子操作
* 并发性能：多线程友好设计
* 缓存友好：连续内存布局

这种设计在以下场景特别有效：

* 高并发应用
* 频繁的内存分配/释放
* 多线程环境
* 对性能要求高的系统

`CentralCache`的实现体现了内存管理中"平衡"的思想，在效率、并发性、复杂度等多个维度之间取得了很好的平衡。

# 项目完整代码

```cpp
// 使用无锁的span信息存储
struct SpanTracker {
    std::atomic<void*> spanAddr{nullptr};
    std::atomic<size_t> numPages{0};
    std::atomic<size_t> blockCount{0};
    std::atomic<size_t> freeCount{0}; // 用于追踪spn中还有多少块是空闲的，如果所有块都空闲，则归还span给PageCache
};

class CentralCache
{
public:
    static CentralCache& getInstance()
    {
        static CentralCache instance;
        return instance;
    }

    void* fetchRange(size_t index);
    void returnRange(void* start, size_t size, size_t index);

private:
    // 相互是还所有原子指针为nullptr
    CentralCache();
    // 从页缓存获取内存
    void* fetchFromPageCache(size_t size);

    // 获取span信息
    SpanTracker* getSpanTracker(void* blockAddr);

    // 更新span的空闲计数并检查是否可以归还
    void updateSpanFreeCount(SpanTracker* tracker, size_t newFreeBlocks, size_t index);

private:
    // 中心缓存的自由链表
    std::array<std::atomic<void*>, FREE_LIST_SIZE> centralFreeList_;

    // 用于同步的自旋锁
    std::array<std::atomic_flag, FREE_LIST_SIZE> locks_;
    
    // 使用数组存储span信息，避免map的开销
    std::array<SpanTracker, 1024> spanTrackers_;
    std::atomic<size_t> spanCount_{0};

    // 延迟归还相关的成员变量
    static const size_t MAX_DELAY_COUNT = 48;  // 最大延迟计数
    std::array<std::atomic<size_t>, FREE_LIST_SIZE> delayCounts_;  // 每个大小类的延迟计数
    std::array<std::chrono::steady_clock::time_point, FREE_LIST_SIZE> lastReturnTimes_;  // 上次归还时间
    static const std::chrono::milliseconds DELAY_INTERVAL;  // 延迟间隔

    bool shouldPerformDelayedReturn(size_t index, size_t currentCount, std::chrono::steady_clock::time_point currentTime);
    void performDelayedReturn(size_t index);
};
```

```cpp
const std::chrono::milliseconds CentralCache::DELAY_INTERVAL{1000};

// 每次从PageCache获取span大小（以页为单位）
static const size_t SPAN_PAGES = 8;

CentralCache::CentralCache()
{
    for (auto& ptr : centralFreeList_)
    {
        ptr.store(nullptr, std::memory_order_relaxed);
    }
    for (auto& lock : locks_)
    {
        lock.clear();
    }
    // 初始化延迟归还相关的成员变量
    for (auto& count : delayCounts_)
    {
        count.store(0, std::memory_order_relaxed);
    }
    for (auto& time : lastReturnTimes_)
    {
        time = std::chrono::steady_clock::now();
    }
    spanCount_.store(0, std::memory_order_relaxed);
}

void* CentralCache::fetchRange(size_t index)
{
    // 索引检查，当索引大于等于FREE_LIST_SIZE时，说明申请内存过大应直接向系统申请
    if (index >= FREE_LIST_SIZE) 
        return nullptr;

    // 自旋锁保护
    while (locks_[index].test_and_set(std::memory_order_acquire))
    {
        std::this_thread::yield(); // 添加线程让步，避免忙等待，避免过度消耗CPU
    }

    void* result = nullptr;
    try 
    {
        // 尝试从中心缓存获取内存块
        result = centralFreeList_[index].load(std::memory_order_relaxed);

        if (!result)
        {
            // 如果中心缓存为空，从页缓存获取新的内存块
            size_t size = (index + 1) * ALIGNMENT;
            result = fetchFromPageCache(size);

            if (!result)
            {
                locks_[index].clear(std::memory_order_release);
                return nullptr;
            }

            // 将获取的内存块切分成小块
            char* start = static_cast<char*>(result);

            // 计算实际分配的页数
            size_t numPages = (size <= SPAN_PAGES * PageCache::PAGE_SIZE) ? 
                             SPAN_PAGES : (size + PageCache::PAGE_SIZE - 1) / PageCache::PAGE_SIZE;
            // 使用实际页数计算块数
            size_t blockNum = (numPages * PageCache::PAGE_SIZE) / size;
            
            if (blockNum > 1) 
            {  // 确保至少有两个块才构建链表
                for (size_t i = 1; i < blockNum; ++i) 
                {
                    void* current = start + (i - 1) * size;
                    void* next = start + i * size;
                    *reinterpret_cast<void**>(current) = next;
                }
                *reinterpret_cast<void**>(start + (blockNum - 1) * size) = nullptr;
                
                // 保存result的下一个节点
                void* next = *reinterpret_cast<void**>(result);
                // 将result与链表断开
                *reinterpret_cast<void**>(result) = nullptr;
                // 更新中心缓存
                centralFreeList_[index].store(
                    next, 
                    std::memory_order_release
                );
                
                // 使用无锁方式记录span信息
                // 做记录是为了将中心缓存多余内存块归还给页缓存做准备。考虑点：
                // 1.CentralCache 管理的是小块内存，这些内存可能不连续
                // 2.PageCache 的 deallocateSpan 要求归还连续的内存
                size_t trackerIndex = spanCount_++;
                if (trackerIndex < spanTrackers_.size())
                {
                    spanTrackers_[trackerIndex].spanAddr.store(start, std::memory_order_release);
                    spanTrackers_[trackerIndex].numPages.store(numPages, std::memory_order_release);
                    spanTrackers_[trackerIndex].blockCount.store(blockNum, std::memory_order_release); // 共分配了blockNum个内存块
                    spanTrackers_[trackerIndex].freeCount.store(blockNum - 1, std::memory_order_release); // 第一个块result已被分配出去，所以初始空闲块数为blockNum - 1
                }
            }
        } 
        else 
        {
            // 保存result的下一个节点
            void* next = *reinterpret_cast<void**>(result);
            // 将result与链表断开
            *reinterpret_cast<void**>(result) = nullptr;
            
            // 更新中心缓存
            centralFreeList_[index].store(next, std::memory_order_release);

             // 更新span的空闲计数
            SpanTracker* tracker = getSpanTracker(result);
            if (tracker)
            {
                // 减少一个空闲块
                tracker->freeCount.fetch_sub(1, std::memory_order_release);
            }
        }
    }
    catch (...) 
    {
        locks_[index].clear(std::memory_order_release);
        throw;
    }

    // 释放锁
    locks_[index].clear(std::memory_order_release);
    return result;
}

void CentralCache::returnRange(void* start, size_t size, size_t index)
{
    if (!start || index >= FREE_LIST_SIZE) 
        return;

    size_t blockSize = (index + 1) * ALIGNMENT;
    size_t blockCount = size / blockSize;    

    while (locks_[index].test_and_set(std::memory_order_acquire)) 
    {
        std::this_thread::yield();
    }

    try 
    {
        // 1. 将归还的链表连接到中心缓存
        void* end = start;
        size_t count = 1;
        while (*reinterpret_cast<void**>(end) != nullptr && count < blockCount) {
            end = *reinterpret_cast<void**>(end);
            count++;
        }
        void* current = centralFreeList_[index].load(std::memory_order_relaxed);
        *reinterpret_cast<void**>(end) = current; // 头插法（将原有链表接在归还链表后边）
        centralFreeList_[index].store(start, std::memory_order_release);
        
        // 2. 更新延迟计数
        size_t currentCount = delayCounts_[index].fetch_add(1, std::memory_order_relaxed) + 1;
        auto currentTime = std::chrono::steady_clock::now();
        
        // 3. 检查是否需要执行延迟归还
        if (shouldPerformDelayedReturn(index, currentCount, currentTime))
        {
            performDelayedReturn(index);
        }
    }
    catch (...) 
    {
        locks_[index].clear(std::memory_order_release);
        throw;
    }

    locks_[index].clear(std::memory_order_release);
}

// 检查是否需要执行延迟归还
bool CentralCache::shouldPerformDelayedReturn(size_t index, size_t currentCount, 
    std::chrono::steady_clock::time_point currentTime)
{
    // 基于计数和时间的双重检查
    if (currentCount >= MAX_DELAY_COUNT)
    {
        return true;
    }
    
    auto lastTime = lastReturnTimes_[index];
    return (currentTime - lastTime) >= DELAY_INTERVAL;
}

// 执行延迟归还
void CentralCache::performDelayedReturn(size_t index)
{
    // 重置延迟计数
    delayCounts_[index].store(0, std::memory_order_relaxed);
    // 更新最后归还时间
    lastReturnTimes_[index] = std::chrono::steady_clock::now();
    
    // 统计每个span的空闲块数
    std::unordered_map<SpanTracker*, size_t> spanFreeCounts;
    void* currentBlock = centralFreeList_[index].load(std::memory_order_relaxed);
    
    while (currentBlock)
    {
        SpanTracker* tracker = getSpanTracker(currentBlock);
        if (tracker)
        {
            spanFreeCounts[tracker]++;
        }
        currentBlock = *reinterpret_cast<void**>(currentBlock);
    }
    
    // 更新每个span的空闲计数并检查是否可以归还
    for (const auto& [tracker, newFreeBlocks] : spanFreeCounts)
    {
        updateSpanFreeCount(tracker, newFreeBlocks, index);
    }
}

void CentralCache::updateSpanFreeCount(SpanTracker* tracker, size_t newFreeBlocks, size_t index)
{
    size_t oldFreeCount = tracker->freeCount.load(std::memory_order_relaxed);
    size_t newFreeCount = oldFreeCount + newFreeBlocks;
    tracker->freeCount.store(newFreeCount, std::memory_order_release);
    
    // 如果所有块都空闲，归还span
    if (newFreeCount == tracker->blockCount.load(std::memory_order_relaxed))
    {
        void* spanAddr = tracker->spanAddr.load(std::memory_order_relaxed);
        size_t numPages = tracker->numPages.load(std::memory_order_relaxed);
        
        // 从自由链表中移除这些块
        void* head = centralFreeList_[index].load(std::memory_order_relaxed);
        void* newHead = nullptr;
        void* prev = nullptr;
        void* current = head;
        
        while (current)
        {
            void* next = *reinterpret_cast<void**>(current);
            if (current >= spanAddr && 
                current < static_cast<char*>(spanAddr) + numPages * PageCache::PAGE_SIZE)
            {
                if (prev)
                {
                    *reinterpret_cast<void**>(prev) = next;
                }
                else
                {
                    newHead = next;
                }
            }
            else
            {
                prev = current;
            }
            current = next;
        }
        
        centralFreeList_[index].store(newHead, std::memory_order_release);
        PageCache::getInstance().deallocateSpan(spanAddr, numPages);
    }
}

void* CentralCache::fetchFromPageCache(size_t size)
{   
    // 1. 计算实际需要的页数
    size_t numPages = (size + PageCache::PAGE_SIZE - 1) / PageCache::PAGE_SIZE;

    // 2. 根据大小决定分配策略
    if (size <= SPAN_PAGES * PageCache::PAGE_SIZE) 
    {
        // 小于等于32KB的请求，使用固定8页
        return PageCache::getInstance().allocateSpan(SPAN_PAGES);
    } 
    else 
    {
        // 大于32KB的请求，按实际需求分配
        return PageCache::getInstance().allocateSpan(numPages);
    }
}

SpanTracker* CentralCache::getSpanTracker(void* blockAddr)
{
    // 遍历spanTrackers_数组，找到blockAddr所属的span
    for (size_t i = 0; i < spanCount_.load(std::memory_order_relaxed); ++i)
    {
        void* spanAddr = spanTrackers_[i].spanAddr.load(std::memory_order_relaxed);
        size_t numPages = spanTrackers_[i].numPages.load(std::memory_order_relaxed);
        
        if (blockAddr >= spanAddr && 
            blockAddr < static_cast<char*>(spanAddr) + numPages * PageCache::PAGE_SIZE)
        {
            return &spanTrackers_[i];
        }
    }
    return nullptr;
}
```

# 项目细节思考

#### 为什么向中心缓存申请内存时是按照传入参数`index`申请？

结合下面代码给出回答：

```cpp
// Common.h
// 对齐数和大小定义
constexpr size_t ALIGNMENT = 8;
constexpr size_t MAX_BYTES = 256 * 1024; // 256KB
constexpr size_t FREE_LIST_SIZE = MAX_BYTES / ALIGNMENT; // ALIGNMENT等于指针void*的大小

static size_t getIndex(size_t bytes)
{   
    // 向上取整后-1
    return (bytes + ALIGNMENT - 1) / ALIGNMENT - 1;
}

// Central.h
void* fetchRange(size_t index);
private:
    // 中心缓存的自由链表
    std::array<std::atomic<void*>, FREE_LIST_SIZE> centralFreeList_;
    // 用于同步的自旋锁
    std::array<std::atomic_flag, FREE_LIST_SIZE> locks_;
```

通过Common.h给出的代码可以看出每个`index`对应一段范围的大小，比如`size`大小为1~8对应的`index`等于1，`size`大小为9~16对应的`index`等于2。。。以此类推，由此我们也可以得知数据结构`centralFreeList_`是怎么样的，如图：

![1737622995759-04921b21-ede6-4a7d-8afa-ae3b25edd72f.png](./img/SxzO2d3PPKBthxIn/1737622995759-04921b21-ede6-4a7d-8afa-ae3b25edd72f-361954.png)

这里介绍一下**中心缓存管理内存的数据结构**，其管理的内存都是`8B`对齐的。当线程本地缓存向其申请内存的时候，当申请内存不能够被`8`整除时就进行向上取整给其分配对应的内存，比如请求内存的大小为`5B`，那么中心缓存将会把大小为`8B`的内存分配给他，以此类推。最大能够向中心缓存申请的内存是`FREE_LIST_SIZE * 8B`也就是`MAX_BYTES`。申请超过这个大小的内存时则直接**向系统申请内存**。

解释到这里大家应该知道为什么向中心缓存申请内存时要按照参数`index`了吧，就是为了能够更方便地在管理内存的数组中找到对应大小的内存并取出给到线程本地缓存。


> 更新: 2025-07-03 19:40:41  
> 原文: <https://www.yuque.com/chengxuyuancarl/ooq1de/chtnm8pou3rfh8id>