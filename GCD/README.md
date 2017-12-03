# GCD

Grand Central Dispatch  只需将需要执行的任务追加到dispatch queue中 那么GCD就能生成必要的线程去计划执行任务 至于线程管理则是让系统的内核去处理 开发者无需关注

一个cpu核同一时间只能处理一条无分叉路经的cpu命令列 也称为线程

当出现多条无分叉路经时 可以称为多线程

可是 一个cpu核可以每隔一段时间 去切换执行cpu路经 这个称为上下文切换

由于可以在某个线程和其他线程之间反复多次进行上下文切换 所以看上去1个cpu核可以能够并列执行处理多个线程一样 假如是多个cpu核的话 就真的是多个cpu核并列执行多个线程

 ## GCD的api

1. Dispatch queue

* serial dispatch queue — 一个队列只有一个线程 串行
* concurrent dispatch queue — 根据系统安排会有几个线程 并发执行

2. api (有create的api都会产生对象 不过该对象受arc管理)

* dispatch_queue_create
* dispatch_get_main_queue — 获取主队列
* dispatch_get_global_queue — 获取全局队列 是concurrent的 根据优先级的不同而有不同的队列
* dispatch_set_target_queue — 既可以设置队列的优先级 也可以作成某个队列的执行阶层
* dispatch_after — 是指延迟指定时间后再将任务追加到队列中
* dispatch_group_create
* dispatch_group_async
* dispatch_group_notify — 是指通知任务组任务都完成时
* dispatch_group_wait — 是指在指定时间等待后 判断该任务是否已执行完
* dispatch_barrier_async — 是指追加前队列的任务都执行后 再将所需要的任务追加到队列中并且**无需等待**该任务执行直接追加后面的任务到队列里
* dispatch_barrier_sync — 是指追加前队列的任务都执行后 再将所需要的任务追加到队列中并且**等待**该任务执行完再追加后面的任务到队列里
* dispatch_apply — 是指当循环执行指定次数的任务 并等待执行完成
* dispatch_suspend/dispatch_resume 挂起队列／恢复队列 （挂起时队列中所有任务将停止）
* dispatch_semaphore_create
* dispatch_semaphore_wait 
* dispatch_semaphore_signal  — 基于任务精度下的控制并发执行 通过 dispatch_semaphore_wait判断指定时间内是否能够继续执行下一次任务 执行完则调用dispatch_semaphore_signal通知下一次的任务执行
* dispatch_once
* dispatch_io_create
* dispatch_io_read — 异步读取文件

## GCD实现

当在队列中执行时 libdispatch从该queue自身的fifo队列中取出dispatch continuation 然后调用pthread_workqueue_additem_np函数 将该queue自身、符合其优先级的workqueue信息以及执行dispatch continuation的回调函数等传递给参数

pthread_workqueue_additem_np函数使用workq_kernreturn系统调用 通知workqueue增加应当执行的项目 根据该通知 xnu内核基于系统状态判断是否要生成线程 如果是overcommit优先级的global dispatch queue,workqueue始终生成线程

workqueue的线程执行pthread_workqueue函数 该函数调用libdispatch的回调函数 在该回调函数中执行加入到dispatch continuation的block

block执行结束后 进行通知dispatch group结束、释放dispatch continuation等处理 开始准备执行加入到global dispatch queue中的下一个block