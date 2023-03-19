---
title: slurm--高吞吐量计算管理指南
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出，来源：https://github.com/SchedMD/slurm/blob/master/doc/html/high_throughput.shtml
published: true
date: 2023-03-19T15:39:25.346Z
tags: slurm
editor: markdown
dateCreated: 2022-12-30T14:39:18.457Z
---

这篇文章包含了Slurm管理员的信息，专门针对高吞吐量计算，即执行许多短作业。为高吞吐量计算获得最佳性能需要一些调整。

## 性能结果
Slurm已经被验证可以在持续的基础上每秒执行500个简单的批处理作业，并在更高的水平上进行短期的突发活动。实际性能取决于要执行的作业和使用的硬件和配置。

## 系统配置
一些系统配置参数可能需要修改，以支持大量打开的文件和有大量信息突发的TCP连接。可以使用`/etc/rc.d/rc.local`或`/etc/sysctl.conf`脚本进行修改，这样在重启后能够保留修改。

- **/proc/sys/fs/file-max：**同时打开的文件的最大数量，我们推荐的限制是至少32832个。
- **/proc/sys/net/ipv4/tcp_max_syn_backlog：**保留在内存中的SYN请求的最大数量，我们还没有从3路握手中获得第三个数据包。对于内存超过128Mb的系统，默认值为1024，对于低内存机器，默认值为128。如果服务器出现过载，可以尝试增加这个数字。
- **/proc/sys/net/ipv4/tcp_syncookies：**当内核为特定套接字的同步积压队列溢出时，用于向主机发送syncookies。默认值是0，它禁用了这个功能。将该值设置为1。
- **/proc/sys/net/ipv4/tcp_synack_retries：**对一个SYN请求重发多少次SYN,ACK回复。换句话说，这告诉系统要尝试建立一个由其他主机启动的被动TCP连接多少次。这个变量是一个整数，但在任何情况下都不应该大于255。每次重传大约需要30到40秒。默认值为5，这导致被动TCP连接的超时时间约为180秒，一般来说是令人满意的。
- **/proc/sys/net/core/somaxconn：**socket listen()积压的限制，在用户空间称为SOMAXCONN。默认值为128。这个值应该被大幅提高，以支持请求的爆发。例如，为了支持1024个请求的爆发，将somaxconn设置为1024。
- **/proc/sys/net/ipv4/ip_local_port_range：**识别可用的外部端口，这些端口用于许多Slurm通信。这个值可以提高以支持大量的通信。例如，将数值 "32768 65535 "写进ip_local_port_range文件，以使该范围的端口可用。

发送队列长度（txqueuelen）可能也需要用ifconfig命令来修改，对于一个拥有非常大的集群的站点来说，推荐将值设置为4096（例如，`ifconfig txqueuelen 409`）。

## Munge配置

默认情况下，Munge守护进程以两个线程运行，但更多的线程数可以提高其吞吐量。我们建议用10个线程来启动Munge守护进程，以支持高吞吐量（例如 `munged --num-reads 10`）。

## 用户限制
对slurmctld守护进程有效的`ulimit`值应该对内存大小、打开的文件数和堆栈大小设置得相当高。

## Slurm配置
几个Slurm配置参数应该被调整以反映高吞吐量计算的需要。下面描述的修改并不适用所有环境，但这些是你可能要考虑的配置选项，以获得更高的吞吐量。

- **AccountingStorageType：**通过使用 `accounting_storage/none` 插件，禁用存储会计。关闭accounting ，对性能的改善微乎其微。如果使用SlurmDBD，可以通过设置slurmdbd.conf中的`CommitDelay`选项来提高速度。
- **JobAcctGatherType：**禁用作业accounting 信息的收集将提高作业的吞吐量，通过使用`jobacct_gather/none`插件来禁用accounting 信息的收集。
- **JobCompType：**禁用作业完成信息的记录将提高作业的吞吐量。通过使用`jobcomp/none`插件禁用作业完成信息的记录。
- **MaxJobCount：**控制在任何时间点上slurmctld守护进程记录中可以有多少作业 (pending, running, suspended or completed[temporarily])。默认值是10,000。
- **MessageTimeout：**控制等待消息响应的时间。默认值是10秒。虽然slurmctld守护进程是高度线程化的，但它的响应速度取决于负载。这个值可能需要增加一些。
- **MinJobAge：**控制已完成作业的记录多久可以从slurmctld内存中清除，从而在squeue命令中不可见。工作运行的记录将保留在accounting 记录和日志中。默认值是300秒。如果可能的话，这个值应该减少到几秒钟。与在slurmctld守护进程的内存中保留旧作业相比，对旧作业使用核算记录可以提高作业的吞吐率。
- **PriorityType：**优先级/builtin比其他选项快得多，但只按先进先出（FIFO）的方式调度作业。
- **SchedulerParameters：**有许多调度参数可用。
  - 设置选项 `batch_sched_delay` 将控制批处理作业的调度可以延迟多长时间。这只影响批处理作业。例如，如果每秒有许多作业被提交，试图调度每个作业的开销会对作业的提交速度产生不利影响。默认值是3秒。
  - 设置选项`defer`将避免在作业提交时试图单独安排每个作业，而是推迟到以后可能同时安排多个作业的时候。当大量作业（几百个）同时提交时，这个选项可能会提高系统的响应速度，但它会延迟单个作业的启动时间。
  - `sched_min_interval`是另一个配置参数，用于控制调度逻辑的运行频率。它仍然可以在每个作业提交、作业终止或其他可能允许启动新作业的状态变化中被触发。然而，这种触发不会导致调度逻辑立即启动，而只是在配置的sched_interval内。例如，如果sched_min_interval=2000000（微秒），100个作业在2秒的时间窗口内被提交，那么调度逻辑将被执行一次，而不是在sched_min_interval被设置为0（无延迟）的情况下执行100次。
  - 除了控制调度逻辑的执行频率，`default_queue_depth`配置参数还控制在每个调度器迭代中考虑启动多少个作业。`default_queue_depth`的默认值是100（作业），这在大多数情况下应该是不错的。
  - 如果使用大量作业，`sched/backfill`插件的开销相对较高。将`bf_max_job_test`配置为一个适度的规模（比如100个作业或更少），将`bf_interval`配置为30秒或更多，将限制回填调度的开销（注意：这两个参数的默认值都不错）。其他可用于调整回填调度的选项包括`bf_max_job_user`、`bf_resolution`和`bf_window`。
  - 下面是一组目前用于在一个集群上每秒持续运行数百个作业的调度参数。请注意，每个环境都是不同的，这组参数并不是在每一种情况下都能很好地工作，但它可以作为一个好的起点。
    - batch_sched_delay=20
    - bf_continue
    - bf_interval=300
    - bf_min_age_reserve=10800
    - bf_resolution=600
    - bf_yield_interval=1000000
    - partition_job_depth=500
    - sched_max_job_start=200
    - sched_min_interval=2000000
- **SchedulerType：**如果大多数工作是短期的，那么建议使用`sched/builtin`插件。它以先入先出（FIFO）的方式管理作业队列，并消除了用于按优先级排序的逻辑。
- **SlurmctldPort：**最好将slurmctld守护进程配置为在一个以上的端口接受传入的消息，以避免传入的消息因超过上述`SOMAXCONN`限制而被操作系统丢弃。当需要支持大量的同时请求时，建议使用两到十个端口。
- **PrologSlurmctld/EpilogSlurmctld：**在高吞吐量的环境中，不建议使用这两个端口。当它们被启用时，必须为每个作业启动（或作业阵列的任务）创建一个单独的slurmctld线程。目前的架构需要在每个线程中获取一个作业写锁，这是一个昂贵的操作，严重限制了调度器的吞吐量。
- **SlurmctldDebug：**更详细的日志记录会降低系统的吞吐量。设置为错误或信息，用于高吞吐量工作负载的常规操作。
- **SlurmdDebug：**更详细的日志记录将减少系统的吞吐量。设置为错误或信息，用于具有高吞吐量工作负荷的常规操作。
- **SlurmdLogFile：**建议写到本地存储。
- **TaskPlugin：**避免使用`task/cgroup`与`ConstrainRAMSpace`的组合，它比其他替代品更慢。同样，`task/affinity`似乎没有增加任何可衡量的开销。建议在任何情况下都使用`task/affinity`。
- **Other：**将日志、accounting 和其他开销配置到适合你环境的最低限度。

## SlurmDBD配置
关掉核算，对性能的提高微乎其微。如果使用SlurmDBD，可以通过设置slurmdbd.conf中的`CommitDelay`选项来提高速度。

你也可以考虑在slurmdbd.conf中设置`'Purge*'`选项来清除旧数据。一个典型的配置应该是这样的...

- PurgeEventAfter=12months
- PurgeJobAfter=12months
- PurgeResvAfter=2months
- PurgeStepAfter=2months
- PurgeSuspendAfter=1month
- PurgeTXNAfter=12months
- PurgeUsageAfter=12months