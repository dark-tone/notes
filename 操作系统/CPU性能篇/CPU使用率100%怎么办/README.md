# 常用参数
- user（通常缩写为 us），代表用户态 CPU 时间。注意，它不包括下面的 nice 时间，但包括了 guest 时间。
- nice（通常缩写为 ni），代表低优先级用户态 CPU 时间，也就是进程的 nice 值被调整为 1-19 之间时的 CPU 时间。这里注意，nice 可取值范围是 -20 到 19，数值越大，优先级反而越低。
- system（通常缩写为 sys），代表内核态 CPU 时间。
- idle（通常缩写为 id），代表空闲时间。注意，它不包括等待 I/O 的时间（iowait）。
- iowait（通常缩写为 wa），代表等待 I/O 的 CPU 时间。
- irq（通常缩写为 hi），代表处理硬中断的 CPU 时间。
- softirq（通常缩写为 si），代表处理软中断的 CPU 时间。
- steal（通常缩写为 st），代表当系统运行在虚拟机中的时候，被其他虚拟机占用的 CPU 时间。
- guest（通常缩写为 guest），代表通过虚拟化运行其他操作系统的时间，也就是运行虚拟机的 CPU 时间。
- guest_nice（通常缩写为 gnice），代表以低优先级运行虚拟机的时间。

而我们通常所说的 CPU 使用率，就是除了空闲时间外的其他时间占总 CPU 时间的百分比。CPU使用率 = 1 - 空闲时间/总CPU时间。

# CPU使用率相关命令
## top
显示了系统总体的 CPU 和内存使用情况，以及各个进程的资源使用情况。但并没有细分进程的用户态 CPU 和内核态 CPU。
> wa：使用率过高时，需要考虑IO的性能是否有瓶颈，可以在使用iostat、sar等命令做进一步分析；<br>
> hi：过高时，表示硬中断占比过大。可以分析文件/proc/interrupts文件<br>
> si：常见的软中断一般都是和网络有关。


## ps
只显示了每个进程的资源使用情况。
```
//-A 列出所有的进程
//-w 显示加宽可以显示较多的资讯
//-au 显示较详细的资讯
//-aux 显示所有包含其他使用者的进程
ps -au
```
## pidstat
```
# 每隔1秒输出一组数据，共输出5组
$ pidstat 1 5
15:56:02      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
15:56:03        0     15006    0.00    0.99    0.00    0.00    0.99     1  dockerd
...
Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
Average:        0     15006    0.00    0.99    0.00    0.00    0.99     -  dockerd
```

# 排查工具
## GDB
GDB 并不适合在性能分析的早期应用，因为 GDB 调试程序的过程会中断程序运行，这在线上环境往往是不允许的。

## perf
### perf top
```
$ perf top
Samples: 833  of event 'cpu-clock', Event count (approx.): 97742399
Overhead  Shared Object       Symbol
   7.28%  perf                [.] 0x00000000001f78a4
   4.72%  [kernel]            [k] vsnprintf
   4.32%  [kernel]            [k] module_get_kallsym
   3.65%  [kernel]            [k] _raw_spin_unlock_irqrestore
...
```
- 第一列 Overhead ，是该符号的性能事件在所有采样中的比例，用百分比来表示。
- 第二列 Shared ，是该函数或指令所在的动态共享对象（Dynamic Shared Object），如内核、进程名、动态链接库名、内核模块名等。
- 第三列 Object ，是动态共享对象的类型。比如 [.] 表示用户空间的可执行程序、或者动态链接库，而 [k] 则表示内核空间。
- 最后一列 Symbol 是符号名，也就是函数名。当函数名未知时，用十六进制的地址来表示。

### perf record与perf report
perf top 虽然实时展示了系统的性能信息，但它的缺点是并不保存数据，也就无法用于离线或者后续的分析。而 perf record 则提供了保存数据的功能，保存后的数据，需要你用 perf report 解析展示。

在实际使用中，我们还经常为 perf top 和 perf record 加上 **-g** 参数，开启调用关系的采样，方便我们根据调用链来分析性能问题。

# 参考
《Linux性能优化实战》
