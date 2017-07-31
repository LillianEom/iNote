# ANR



## ANR分析

### ANR的分类：

1. **KeyDispatchTimeout –按键或触摸事件在特定时间内无响应；**
2. **BroadcastTimeout –BroadcastReceiver在特定时间内无法处理完成；**
3. **ServiceTimeout –Service在特定的时间内无法处理完成；**

### ANR的发生原因：

1. 应用自身引起，例如：
   - 主线程阻塞、IOWait等；
2. 其他进程间接引起，例如：
   - 当前应用进程进行进程间通信请求其他进程，其他进程的操作长时间没有反馈；
   - 其他进程的CPU占用率高，使得当前应用进程无法抢占到CPU时间片；

### ANR日志分析：

当发生 ANR 的时候 Logcat 中会出现提示：

```
04-06 15:58:46.215 23480-23483/com.example.testanr I/art: Thread[2,tid=23483,WaitingInMainSignalCatcherLoop,Thread*=0x7fa2307000,peer=0x12cb40a0,"Signal Catcher"]: reacting to signal 3
04-06 15:58:46.364 23480-23483/com.example.testanr I/art: Wrote stack traces to '/data/anr/traces.txt'
```

**ANR 的 Log 信息保存在：/data/anr/traces.txt，每一次新的ANR发生，会把之前的ANR信息覆盖掉。**

例如：

```
04-01 13:12:11.572 I/InputDispatcher( 220): Application is not responding:Window{2b263310com.android.email/com.android.email.activity.SplitScreenActivitypaused=false}.
5009.8ms since event, 5009.5ms since waitstarted
04-0113:12:11.572 I/WindowManager( 220): Input event 
dispatching timedout sending 
tocom.android.email/com.android.email.activity.SplitScreenActivity

04-01 13:12:14.123 I/Process( 220): Sending signal. PID: 21404 SIG:3---发生ANR的时间和生成trace.txt的时间
04-01 13:12:14.123 I/dalvikvm(21404):threadid=4: reacting to signal 3 ……
04-0113:12:15.872 E/ActivityManager( 220): ANR in com.android.email(com.android.email/.activity.SplitScreenActivity)
04-0113:12:15.872 E/ActivityManager( 220): Reason:keyDispatchingTimedOut  -----ANR的类型
04-0113:12:15.872 E/ActivityManager( 220): Load: 8.68 / 8.37 / 8.53 --CPU的负载情况
04-0113:12:15.872 E/ActivityManager( 220): CPUusage from 4361ms to 699ms ago ----CPU在ANR发生前的使用情况；备注：这个ago，是发生前一段时间的使用情况，不是当前时间点的使用情况；

04-0113:12:15.872 E/ActivityManager( 220): 5.5%21404/com.android.email: 1.3% user + 4.1% kernel / faults:
10 minor
04-0113:12:15.872 E/ActivityManager( 220): 4.3%220/system_server: 2.7% user + 1.5% kernel / faults: 11
minor 2 major
04-0113:12:15.872 E/ActivityManager( 220): 0.9%52/spi_qsd.0: 0% user + 0.9% kernel
04-0113:12:15.872 E/ActivityManager( 220): 0.5%65/irq/170-cyttsp-: 0% user + 0.5% kernel
04-0113:12:15.872 E/ActivityManager( 220): 0.5%296/com.android.systemui: 0.5% user + 0% kernel
04-0113:12:15.872 E/ActivityManager( 220): 100%TOTAL: 4.8% user + 7.6% kernel + 87% iowait----注意这行：注意87%的iowait
04-0113:12:15.872 E/ActivityManager( 220): CPUusage from 3697ms to 4223ms later:-- ANR后CPU的使用量
04-0113:12:15.872 E/ActivityManager( 220): 25%21404/com.android.email: 25% user + 0% kernel / faults: 191 minor
04-0113:12:15.872 E/ActivityManager( 220): 16% 21603/__eas(par.hakan: 16% user + 0% kernel
04-0113:12:15.872 E/ActivityManager( 220): 7.2% 21406/GC: 7.2% user + 0% kernel
04-0113:12:15.872 E/ActivityManager( 220): 1.8% 21409/Compiler: 1.8% user + 0% kernel
04-0113:12:15.872 E/ActivityManager( 220): 5.5%220/system_server: 0% user + 5.5% kernel / faults: 1 minor
04-0113:12:15.872 E/ActivityManager( 220): 5.5% 263/InputDispatcher: 0% user + 5.5% kernel
04-0113:12:15.872 E/ActivityManager( 220): 32%TOTAL: 28% user + 3.7% kernel
```

**从Logcat中可以得到以下信息：**

1. 导致ANR的包名（com.android.emai），类名（com.android.email.activity.SplitScreenActivity），进程PID（21404）
2. 导致ANR的原因：keyDispatchingTimedOut
3. 系统中活跃进程的CPU占用率，关键的一句：100%TOTAL: 4.8% user + 7.6% kernel + 87% iowait；表示CPU占用满负荷了，其中绝大数是被iowait即I/O操作占用了。我们就可以大致得出是io操作导致的ANR。

## ANR触发场景

1. InputDispatching Timeout ：输入事件分发超时 5s 未响应完毕；
2. BroadcastQueue Timeout ：前台广播在 10s 内、后台广播在 20 秒内未执行完成；
3. Service Timeout ：前台服务在20s内、后台服务在200秒内未执行完成；
4. ContentProvider Timeout ：内容提供者,在 publish 过超时 10s；



http://www.jianshu.com/p/af13abc5f0c8





































