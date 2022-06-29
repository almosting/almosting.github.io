---
title: Android Performance
date: 2022-06-28 20:06:56
categories: Android
tags: 性能优化
---

# 分析 Android Performance 问题

## CPU 问题

查看 cpu 频率：

`cat /sys/devices/system/cpu/cpux/cpufreq/scaling_curfreq`

## TOP 信息

> 查看 TOP 消息，查看后台是否干净，查看 idle 是否异常

`system 1%(1),IOW 10%(2)`

- (1) 该值高就需要查看进程，看 CPU 占比

  ```shell

  $ adb shell top -d 1 -m 10 -t


  *"-d 1": 每一秒刷一次

  *"-m 10": 输出前10个

  *"-t": 输出线程

  ```

  如果需要查看进程的 call stack，使用如下命令

  ```shell

   $ adb shell touch /data/anr/traces.txt

   $ adb shell kill -3 [PID](进程号)

  ```
- (2) 如果 IOW 偏高，先关闭怀疑的进程，如果还是高，需要查看是否系统 IO 占用高

## Systrace



## Bugreport



## Vmstat 工具

**vmstat: virtual Memory Statistics (虚拟内存统计)**, 是一个很有价值的监控工具，可以实时提供 memory、block IO 和 CPU 状态信息。它用来对系统整体的情况进行统计，通过它可以快速了解当前系统运行情况及可能碰到的问题。

使用方式：

```shell

vmstat 1 10

usage: vmstat [-n][DELAY [COUNT]]

```

```markdown

procs -----------memory---------- ---swap-- -----io---- -system-- ----cpu----

 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa

 0  0      0  57080  50060 1429612   0    0   712    67    0  237 10  6 83  0

 0  0      0  57064  50060 1429612   0    0     0     0    0  181  0  0 100 0

 0  0      0  57064  50060 1429612   0    0     0     0    0  100  0  0 100 0

```

### Pocs

-**r:**(running) 运行队列中的进程数，这个值超过 CPU 的个数就会出现瓶颈

-**b：**(blocked) 被阻塞的进程数，通常是进程等待 IO 操作

### Memory

- swpd 虚拟内存大小，大于 0，表示物理内存不足，可能出现泄漏，或者后台 activity 或者 jobs 太多
- free 可用物理内存
- buff 系统用作 buffers 的内存
- cache 系统用作 cache 的内存数，可以理解为缓存

### Swap

- si 每秒从磁盘读入内存的大小，如果大于 0，表示物理内存不足或者内存泄漏
- so 每秒从内存写入磁盘的大小

### IO

- bi 块设备接收的块数量 代表 io 操作，如 copy
- bo 块设备每秒发送的块数量，如读取文件

一般 bi bo 都要接近 0，否则 io 过于频繁

### System

- in(interrupts) 在 delay(默认为 1s) 时间内系统产生的中断数，值过大，可以通过 `cat /proc/interrupts`查看哪个模块中断产生过多
- cs(context switch) 在 delay 时间内，系统上下文切换次数，这个值越小越好，太大证明

  CPU 大部分时间浪费在上下文切换去了，一般需要查看中断和线程调度

### CPU

- us 用户占用的 CPU 时间比，占比高，需要查看进程是否有闭环代码
- sy 系统时间占用 CPU 时间比，太高，表示系统调用时间长，需要关心 IO 等操作
- id idle 闲置 CPU 时间比
- wa IO wait CPU 等待 IO 完成的时间占比，值太大，对系统负担大

### Tips

1. 如果 r 值经常大于 1 或者更大，并且 id 常小于 40%，说明 CPU 负荷过重。
2. 当手机正常使用时，有较小的 free 是好事，说明 cache 使用更有效率，除非此时有不断的写入 swap，disk (so, bo). cache 值如果偏大，且 bi 值小，表示 data 存于 cache 中不必块读取，使用效率高。Android 中当内存不够时会由 oom killer 来根据优先级顺序将不太重要的后台 cached 进程杀掉来释放这段内存。当该值忽大忽小时，需要注意是否有 cache 被清理出去，cache 对系统的性能和流畅性影响很大。
3. 如果 swapd 值大于 0，而 si，so 都显示 0，此时系统性能还是正常的。
4. 如果 free 值很少或者接近 0 并不代表内存不够用，此时如果 si，so 也很少（大多时候是 0）系统性能不会受到影响的。
5. 如果 bi，bo 出现较大值，而 si，so 却维持在 0 值，表示系统 IO 负载过重，要查看 IO 处理或者 Rom 是否有异常。
6. 如果 bi，bo 和 si，so 同时为较大数值，表示内存 swapping 频繁，RAM 太小。
7. bi 或 bo 出现 0 的次数太过频繁，除非是系统处于闲置状态，否则需要查看 I/O 方面是否出现问题
8. 进程 kswapd 是负责确保闲置内存可被释放，每次启动扫描会尝试释放 32 个 pages，并且一直在重复这个程序直到限制内存数值高于 pages_high（核心参数）
