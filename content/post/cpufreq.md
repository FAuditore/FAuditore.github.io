---
title: "Cpufreq"
date: 2020-12-09T10:07:34+08:00
draft: true
---

cpufreq是一个动态调整cpu频率的模块，系统启动时生成一个文件夹/sys/devices/system/cpu/cpu0/cpufreq/，里面有几个文件

其中scaling_min_freq代表最低频率，scaling_max_freq代表最高频率，scalin_governor代表cpu频率调整模式，用它来控制CPU频率
其中 

1.performance ：

顾名思义只注重效率，将CPU频率固定工作在其支持的最高运行频率上，而不动态调节。

2.powersave：

将CPU频率设置为最低的所谓“省电”模式，CPU会固定工作在其支持的最低运行频率上。因此这两种governors 都属于静态governor，即在使用它们时CPU 的运行频率不会根据系统运行时负载的变化动态作出调整。这两种governors 对应的是两种极端的应用场景，使用performance governor 是对系统高性能的最大追求，而使用powersave governor 则是对系统低功耗的最大追求。

3.Userspace：

最早的cpufreq 子系统通过userspace governor为用户提供了这种灵活性。系统将变频策略的决策权交给了用户态应用程序，并提供了相应的接口供用户态应用程序调节CPU 运行频率使用。也就是长期以来都在用的那个模式。可以通过手动编辑配置文件进行配置

4.ondemand 

按需快速动态调整CPU频率， 一有cpu计算量的任务，就会立即达到最大频率运行，等执行完毕就立即回到最低频率;userspace是内核态的检测，用户态调整，效率低。而ondemand正是人们长期以来希望看到的一个完全在内核态下工作并且能够以更加细粒度的时间间隔对系统负载情况进行采样分析的governor。 在 ondemand governor 监测到系统负载超过 up_threshold 所设定的百分比时，说明用户当前需要 CPU 提供更强大的处理能力，因此 ondemand governor 会将CPU设置在最高频率上运行。但是当 ondemand governor 监测到系统负载下降，可以降低 CPU 的运行频率时，到底应该降低到哪个频率呢？ ondemand governor 的最初实现是在可选的频率范围内调低至下一个可用频率，例如 CPU 支持三个可选频率，分别为 1.67GHz、 1.33GHz 和 1GHz ，如果 CPU 运行在 1.67GHz 时 ondemand governor 发现可以降低运行频率，那么 1.33GHz 将被选作降频的目标频率。

5.conservative

与ondemand不同，平滑地调整CPU频率，频率的升降是渐变式的,会自动在频率上下限调整，和ondemand的区别 在于它会按需分配频率，而不是一味追求最高频率。

查看当前的调节器： 
`cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`

更改使用的调节器，需再更改scaling_governor文件： 
`echo conservative > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`