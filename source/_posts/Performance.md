---
title: Android Performance
date: 2022-06-28 20:06:56
categories: Android
tags: 性能优化
---

# 分析Android Performance问题

## CPU问题

查看cpu频率：

`cat /sys/devices/system/cpu/cpux/cpufreq/scaling_curfreq`

## TOP信息

> 查看TOP消息，查看后台是否干净，查看idle是否异常

`system 1%(1),IOW 10%(2)`

- (1)该值高就需要查看进程，看CPU占比

  ```shell

  $ adb shell top -d 1 -m 10 -t


  *"-d 1": 每一秒刷一次

  *"-m 10": 输出前10个

  *"-t": 输出线程

  ```

  如果需要查看进程的call stack，使用如下命令

  ```shell

   $ adb shell touch /data/anr/traces.txt

   $ adb shell kill -3 [PID](进程号)

  ```
- (2)如果IOW偏高，先关闭怀疑的进程，如果还是高，需要查看是否系统IO占用高

## Systrace



## Bugreport



## Vmstat工具

**vmstat: virtual Memory Statistics (虚拟内存统计)**, 是一个很有价值的监控工具，可以实时提供memory、block IO和CPU状态信息。它用来对系统整体的情况进行统计，通过它可以快速了解当前系统运行情况及可能碰到的问题。

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

-**r:**(running)运行队列中的进程数，这个值超过CPU的个数就会出现瓶颈

-**b：**(blocked)被阻塞的进程数，通常是进程等待IO操作

### Memory

- swpd 虚拟内存大小，大于0，表示物理内存不足，可能出现泄漏，或者后台activity或者jobs太多
- free 可用物理内存
- buff 系统用作buffers的内存
- cache 系统用作cache的内存数，可以理解为缓存

### Swap

- si 每秒从磁盘读入内存的大小，如果大于0，表示物理内存不足或者内存泄漏
- so 每秒从内存写入磁盘的大小

### IO

- bi 块设备接收的块数量 代表io操作，如copy
- bo 块设备每秒发送的块数量，如读取文件

一般bi bo都要接近0，否则io过于频繁

### System

- in(interrupts) 在delay(默认为1s)时间内系统产生的中断数，值过大，可以通过 `cat /proc/interrupts`查看哪个模块中断产生过多
- cs(context switch) 在delay时间内，系统上下文切换次数，这个值越小越好，太大证明

  CPU大部分时间浪费在上下文切换去了，一般需要查看中断和线程调度

### CPU

- us 用户占用的CPU时间比，占比高，需要查看进程是否有闭环代码
- sy 系统时间占用CPU时间比，太高，表示系统调用时间长，需要关心IO等操作
- id idle闲置CPU时间比
- wa IO wait CPU等待IO完成的时间占比，值太大，对系统负担大

### Tips

1. 如果r值经常大于1或者更大，并且id常小于40%，说明CPU负荷过重。
2. 当手机正常使用时，有较小的free是好事，说明cache使用更有效率, 除非此时有不断的写入swap，disk (so, bo). cache值如果偏大，且bi值小，表示data存于cache中不必块读取，使用效率高。Android中当内存不够时会由oom killer来根据优先级顺序将不太重要的后台cached进程杀掉来释放这段内存。当该值忽大忽小时，需要注意是否有cache被清理出去，cache对系统的性能和流畅性影响很大。
3. 如果swapd值大于0，而si，so都显示0，此时系统性能还是正常的。
4. 如果free值很少或者接近0并不代表内存不够用，此时如果si，so也很少（大多时候是0）系统性能不会受到影响的。
5. 如果bi，bo出现较大值，而si，so却维持在0值，表示系统IO负载过重，要查看IO处理或者Rom是否有异常。
6. 如果bi，bo和si，so同时为较大数值，表示内存swapping频繁，RAM太小。
7. bi或bo出现0的次数太过频繁，除非是系统处于闲置状态，否则需要查看I/O方面是否出现问题
8. 进程kswapd是负责确保闲置内存可被释放，每次启动扫描会尝试释放32个pages，并且一直在重复这个程序直到限制内存数值高于pages_high（核心参数）
