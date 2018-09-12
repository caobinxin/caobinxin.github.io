# binder之open分析

## 框架

路径：FireNow-Nougat/kernel/drivers/android/binder.c

```c
device_initcall(binder_init); /*mount_init() 可以作为模块随时可以卸载  device_initcall() 和系统融为一体，这里可以将其替换为 mount_init()*/


static int __init binder_init(void)
{
    int ret;
    char *device_name, *device_names;
    struct binder_device *device;
    struct hlist_node *tmp;

    binder_deferred_workqueue = create_singlethread_workqueue("binder");
    if (!binder_deferred_workqueue)
        return -ENOMEM;

 
    /*
     * Copy the module_parameter string, because we don't want to
     * tokenize it in-place.
     */
    device_names = kzalloc(strlen(binder_devices_param) + 1, GFP_KERNEL);
    if (!device_names) {
        ret = -ENOMEM;
        goto err_alloc_device_names_failed;
    }
    strcpy(device_names, binder_devices_param);

    
    /*根据加载binder驱动时，传入的参数（binder 设备的名字），来在内核中注册不同的binder设备*/
    while ((device_name = strsep(&device_names, ","))) {
        ret = init_binder_device(device_name);
        if (ret)
            goto err_init_binder_device_failed;
    }

    return ret;

}

```

现在遗留两个问题：

* 加载驱动时，创建的这个 binder 工作队列是干啥用的？

  ```c
  binder_deferred_workqueue = create_singlethread_workqueue("binder");
  ```

* 为啥要创建多个binder 设备？这样做的意义是？

  ```c
      while ((device_name = strsep(&device_names, ","))) {
          ret = init_binder_device(device_name);
          if (ret)
              goto err_init_binder_device_failed;
      }
  ```

### init_binder_device(device_name);

```c
static int __init init_binder_device(const char *name)
{
    int ret;
    struct binder_device *binder_device;

    binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
    if (!binder_device)
        return -ENOMEM;

    binder_device->miscdev.fops = &binder_fops;/*方法集合，关键*/
    binder_device->miscdev.minor = MISC_DYNAMIC_MINOR;/*让kernel动态分配次设备号，因为是混杂设备，默认的主设备号是10*/
    binder_device->miscdev.name = name;

    binder_device->context.binder_context_mgr_uid = INVALID_UID;/*这里将UID初始化为-1 无效*/
    binder_device->context.name = name;

    ret = misc_register(&binder_device->miscdev);/*混杂设备就是 字符设备的封装形式*/
    if (ret < 0) {
        kfree(binder_device);
        return ret;
    }

    hlist_add_head(&binder_device->hlist, &binder_devices);    /*将binder设备加入 设备s列表中*/                                                                                                              

    return ret;
}

```

### binder_fops

```c
static const struct file_operations binder_fops = {
    .owner = THIS_MODULE,
    .poll = binder_poll,
    .unlocked_ioctl = binder_ioctl,
    .compat_ioctl = binder_ioctl,
    .mmap = binder_mmap,
    .open = binder_open,
    .flush = binder_flush,
    .release = binder_release,
};

```

### 框架小结

首先binder设备驱动，从本质来说他是一个字符设备驱动，只不过这里是用主设备号为10的混杂设备框架，来简化注册过程。并且我们从方法集合中，还可以得到一个结论，这里没有实现 write 和 read  接口。之所以没有实现这两个接口，只因为，我们通过ioctl接口，可以将write和read玩的很花哨。

## open

```c
static int binder_open(struct inode *nodp, struct file *filp)                                                                                                                
{   
    struct binder_proc *proc; /*该结构体，和proc文件系统有关，将当前binder设备的一些状态信息，反馈在proc文件系统中，但我看了3399并没有发现proc文件夹下有binder相关的信息，这个还需要继续深究*/
    struct binder_device *binder_dev;
    
    binder_debug(BINDER_DEBUG_OPEN_CLOSE, "binder_open: %d:%d\n",
             current->group_leader->pid, current->pid);
    
    proc = kzalloc(sizeof(*proc), GFP_KERNEL);
    if (proc == NULL)
        return -ENOMEM;
    get_task_struct(current); 
    proc->tsk = current; /*保存 调用open接口的  进程信息task_struct*/
    INIT_LIST_HEAD(&proc->todo);/*初始化这个todo链表*/
    init_waitqueue_head(&proc->wait);/*初始化这个等待队列*/
    proc->default_priority = task_nice(current);
    binder_dev = container_of(filp->private_data, struct binder_device,
                  miscdev);/*这里比较奇怪， 什么时候 private_data和 miscdev 关联起来的*/                 
    proc->context = &binder_dev->context;
    
    binder_lock(__func__);    
    
    binder_stats_created(BINDER_STAT_PROC);
    hlist_add_head(&proc->proc_node, &binder_procs);
    proc->pid = current->group_leader->pid;
    INIT_LIST_HEAD(&proc->delivered_death);
    filp->private_data = proc;

    binder_unlock(__func__);

    if (binder_debugfs_dir_entry_proc) {
        char strbuf[11];

        snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
        /*
         * proc debug entries are shared between contexts, so
         * this will fail if the process tries to open the driver
         * again with a different context. The priting code will
         * anyway print all contexts that a given PID has, so this
         * is not a problem.
         */
        proc->debugfs_entry = debugfs_create_file(strbuf, S_IRUGO,
            binder_debugfs_dir_entry_proc,
            (void *)(unsigned long)proc->pid,
            &binder_proc_fops);
    }                                                                                                                                                                        

    return 0;
}

```

open函数主要做一下几个工作：

1. 给 struct binder_proc *proc 分配空间
2. 并将当前的进程信息保存到 proc中
3. 将struct binder_proc 放到 filp->private_data 中，这样我们就能通过 filp来找到 我们保存的 proc（进程）相关的信息
4. 将proc->proc_node 挂在 binder_procs上

## 总结

1. binder驱动是用的字符设备框架
2. struct binder_proc 结构体中，保存的是此次打开binder设备 的进程的信息（此时就记录了UID PID等信息），我们在binder简介中提到过，之所以要使用binder，其中一个主要考虑就是安全（这里我猜想我们已经拿到当前进程的所有信息，那是否就可以 利用uid和pid　来做权限的筛选）
3. 将proc　进程信息　保存在　字符设备的私有数据域中，这样我们就能随时获取到
4. 还有对一些　链表的初始化

