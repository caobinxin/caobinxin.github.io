# __cacheline_aligned_in_smp

```c
struct worker_pool{
    ...
    atomic_t nr_running __cacheline_aligned_in_smp;
    ...
} __cacheline_aligned_in_smp;
```

__cacheline_aligned_in_smp 的意思是：

首先我们先说一下　nr_running 是啥意义：

用于管理worker的创建和销毁，表示正在运行中的worker数量。在进程调度中唤醒进程时（try_to_wake_up()），其他cpu有可能会同时访问该成员，该成员频繁在多核之间读写，因此让该成员独占一个缓冲行，**避免多核cpu在读写该成员时引发其他临近成员　颠簸　现象,这就是所谓的　缓存行伪共享问题**.





