---
layout: post
author: Edward Burden
title : Thread Pool
project :  C++
category : Updates
thumbnail: Thread-Pool.png
github: https://github.com/EdwardBurden/ThreadPool
excerpt: A simple and easy to use threadpool.
---

<style>
.alloc-code { background: rgba(127, 127, 127, 0.12); color: inherit; padding: 1rem; border-radius: 0.5rem; overflow-x: auto; font-size: 0.875rem; }
.alloc-code code { color: inherit; background: none; }
mark.hl { color: inherit; padding: 0.05rem 0.2rem; border-radius: 0.2rem; }
mark.hl-1 { background: rgba(0, 123, 255, 0.30); }
mark.hl-2 { background: rgba(40, 167, 69, 0.32); }
mark.hl-3 { background: rgba(255, 153, 0, 0.38); }
mark.hl-4 { background: rgba(156, 39, 176, 0.30); }
</style>

One thing I felt I hadn't really looked into in C++ was multithreading and concurrency, and how to make my code "thread safe", which was something I had seen but didn't fully understand. So I gave myself the challenge of making a simple and highly usable thread pool that I could place into a game engine or any other codebase and it would just work. I also wanted to understand what each bit did and why.

There were already some good resources online, like [this GeeksforGeeks article](https://www.geeksforgeeks.org/cpp/thread-pool-in-cpp/) and [this GitHub repo](https://github.com/progschj/ThreadPool/tree/master). But instead of just copying them, I really wanted to make something unique.

## The worker thread

I started by getting the class structure down for both the worker thread and the thread pool itself. I wanted to keep an OOP approach, so the worker thread holds the thread it was assigned, along with any other info, like its index and active status. I also moved the run logic into this class.

```cpp
class WorkerThread
{
public:
    WorkerThread(ThreadPool* pool, int index);
    WorkerThread(const WorkerThread&) = delete;
    WorkerThread& operator=(const WorkerThread&) = delete;
    WorkerThread(WorkerThread&&) = delete;
    WorkerThread& operator=(WorkerThread&&) = delete;
    ~WorkerThread();

    void Run();
    bool IsActive() const { return m_isActive; }

private:
    std::atomic<bool> m_isActive{ false };
    ThreadPool* m_pool;
    std::thread m_thread;
    int m_index;
};
```

## The thread pool

The thread pool then has a vector of these workers and a queue of tasks. When creating the thread pool, you need to specify how many workers you want.

```cpp
class ThreadPool
{
public:

    friend class WorkerThread;

    ThreadPool(size_t size);
    ThreadPool(const ThreadPool&) = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;
    ThreadPool(ThreadPool&&) = delete;
    ThreadPool& operator=(ThreadPool&&) = delete;
    ~ThreadPool();

    template<typename F, typename... Args>
    auto EnqueueTask(F&& f, Args&&... args)
    {
        using returnType = std::invoke_result_t<F, Args...>;
        std::packaged_task<returnType()> task(std::bind(std::forward<F>(f), std::forward<Args>(args)...));
        std::shared_ptr<std::packaged_task<returnType()>> pointer = std::make_shared<std::packaged_task<returnType()>>(std::move(task));
        std::future<returnType> future = pointer->get_future();
        {
            std::unique_lock<std::mutex> lock(m_queueLock);
            m_tasks.emplace([pointer]()
                {
                    (*pointer)();
                });
        }
        m_condition.notify_one();
        return future;
    }

    bool HasTasks();
    bool WorkersActive();
    void WaitForAllTasksComplete();

private:
    std::atomic<bool> m_stopFlag{ false };
    std::vector<std::unique_ptr<WorkerThread>> m_workers;
    std::queue<std::function<void()>> m_tasks;
    size_t m_size;
    std::mutex m_queueLock;
    std::condition_variable m_condition;
};
```

## Keeping it thread safe

The task queue is shared between the code adding work and every worker taking it, so it needs protecting. A mutex guards the queue, and a condition variable puts idle workers to sleep and wakes them when there is work.

### Adding a task

`EnqueueTask` pushes the task while holding `m_queueLock`, releases it, and only then calls `notify_one()` to wake a single worker. That way the worker isn't woken just to be blocked by the mutex.

### The worker loop

Each worker's `Run` function loops, taking tasks off the queue:

<pre class="alloc-code"><code>inline void WorkerThread::Run()
{
    while (true)
    {
        std::function&lt;void()&gt; currentTask;
        {
            <mark class="hl hl-1">std::unique_lock&lt;std::mutex&gt; lock(m_pool-&gt;m_queueLock);</mark>
            <mark class="hl hl-2">m_pool-&gt;m_condition.wait(lock, [this]</mark>
                {
                    <mark class="hl hl-2">auto conditionPassed = !m_pool-&gt;m_tasks.empty() || m_pool-&gt;m_stopFlag;</mark>
                    <mark class="hl hl-2">return conditionPassed;</mark>
                });
            <mark class="hl hl-2">if (m_pool-&gt;m_stopFlag)</mark>
                <mark class="hl hl-2">return;</mark>

            <mark class="hl hl-1">currentTask = std::move(m_pool-&gt;m_tasks.front());</mark>
            <mark class="hl hl-1">m_pool-&gt;m_tasks.pop();</mark>
        }
        m_isActive = true;

#ifdef THREADPOOL_DEBUG
        {
            <mark class="hl hl-4">std::lock_guard lock(consoleMutex);</mark>
            printf("%i Start\n", m_index);
        }
#endif //THREADPOOL_DEBUG
        currentTask();
#ifdef THREADPOOL_DEBUG
        {
            <mark class="hl hl-4">std::lock_guard lock(consoleMutex);</mark>
            printf("%i End\n", m_index);
        }
#endif //THREADPOOL_DEBUG
        m_isActive = false;
        <mark class="hl hl-3">m_pool-&gt;m_condition.notify_all();</mark>
    }
}</code></pre>

**The mutex.** Each worker <mark class="hl hl-1">takes a <code>std::unique_lock</code> on the queue mutex</mark>, so only one thread touches `m_tasks` at a time, and <mark class="hl hl-1">moves the task out and pops it</mark> while holding the lock. The braces release the lock before the task runs. Otherwise only one task could run at a time.

**The condition variable.** With nothing to do, a worker <mark class="hl hl-2">waits on the condition variable</mark> instead of polling the queue. It wakes when <mark class="hl hl-2">there is a task or the stop flag is set</mark>. The condition is rechecked on every wake, which guards against spurious wake ups and another worker taking the task first. While it sleeps, `wait` releases the mutex. If the stop flag is set, <mark class="hl hl-2">the worker returns</mark>.

**Finishing.** When a task completes, <mark class="hl hl-3"><code>notify_all</code> wakes everything waiting</mark> so it can check again whether the work is done. `m_isActive` is atomic, so it needs no mutex.

### A second mutex, just for printing

In debug builds, a separate mutex stops output from different threads mixing together, with <mark class="hl hl-4">each print wrapped in a <code>std::lock_guard</code></mark>. The constructor does the same, and it also clamps the worker count between 1 and `std::thread::hardware_concurrency()`.

<pre class="alloc-code"><code>//ThreadPool
inline ThreadPool::ThreadPool(size_t size)
{
    unsigned int max = std::thread::hardware_concurrency();
    if (max == 0)
        max = 1;
    m_size = std::clamp(static_cast&lt;unsigned int&gt;(size), 1u, max);
    m_workers.reserve(m_size);
    for (size_t i = 0; i &lt; m_size; i++)
    {
        m_workers.emplace_back(std::make_unique&lt;WorkerThread&gt;(this, i));
    }
#ifdef THREADPOOL_DEBUG
    {
        <mark class="hl hl-4">std::lock_guard lock(consoleMutex);</mark>
        printf("Pool Alive Size:%zu\n", m_size);
    }
#endif //THREADPOOL_DEBUG
}</code></pre>