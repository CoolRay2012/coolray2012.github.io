---
title: Linux Alarm Timer
date: 2018-11-07 19:00:53
tags: [linux, time, powermanagement]
categories: linux
---

# Alarmtimer definition
```
clock
-> clock source
-> clock event
  -> timer
    -> alarmtimer
```

## timer vs. alarm timer
timer setup on system runtime, e.g. hrtimer which would be stopped when system suspend/idle
alarm timer can still work during idle/suspend (by RTC device), wake system from suspend

* CLOCK_REALTIME_ALARM
* CLOCK_BOOTTIME_ALARM

# Implementation
Block
![](linuxalarmtimer.jpg)

## Initialization
```
alarmtimer_init()
-> struct k_clock alarm_clock
: init k_clock for alarmtimer
-> alarmtimer_rtc_timer_init()
-> posix_timers_register_clock(CLOCK_REALTIME_ALARM, &alarm_clock);
-> posix_timers_register_clock(CLOCK_BOOTTIME_ALARM, &alarm_clock);
: register k_clock for posix timer
-> init alarm bases (alarm_bases[i].timer.function = alarmtimer_fired
-> alarmtimer_rtc_interface_setup()
-> platform_driver_register()
: register suspend callback
-> wakeup_source_register
```

## Set & del
```
alarm_timer_set-->alarm_start-->alarmtimer_enqueue-->timerqueue_add
alarm_timer_del-->alarm_try_to_cancel-->alarmtimer_remove-->timerqueue_del
```

## Suspend
when suspend , callback function get neareset alarm timer and setup RTC timeout, ensure RTC wakeup system when timeout.
```c
static int alarmtimer_suspend(struct device *dev)
{
...
    rtc = alarmtimer_get_rtcdev();-------------------------------------get rtc device
    /* If we have no rtcdev, just return */
    if (!rtc)
        return 0;
    /* Find the soonest timer to expire*/
    for (i = 0; i < ALARM_NUMTYPE; i++) {------------------------------go through ALARM_REALTIME & ALARM_BOOTTIME alarm_base，get nearest expires
        struct alarm_base *base = &alarm_bases[i];
        struct timerqueue_node *next;
        ktime_t delta;
        spin_lock_irqsave(&base->lock, flags);
        next = timerqueue_getnext(&base->timerqueue);
        spin_unlock_irqrestore(&base->lock, flags);
        if (!next)
            continue;
        delta = ktime_sub(next->expires, base->gettime());
        if (!min.tv64 || (delta.tv64 < min.tv64))---------------------get nearest
            min = delta;
    }
    if (min.tv64 == 0)
        return 0;
    if (ktime_to_ns(min) < 2 * NSEC_PER_SEC) {-----------------------abort suspend if the timer will occur near 2 secs
        __pm_wakeup_event(ws, 2 * MSEC_PER_SEC);
        return -EBUSY;
    }
    /* Setup an rtc timer to fire that far in the future */
    rtc_timer_cancel(rtc, &rtctimer);-------------------------------cancel current rtctimer
    rtc_read_time(rtc, &tm);
    now = rtc_tm_to_ktime(tm);
    now = ktime_add(now, min);
    /* Set alarm, if in the past reject suspend briefly to handle */
    ret = rtc_timer_start(rtc, &rtctimer, now, ktime_set(0, 0));---set rtctimer
    if (ret < 0)
        __pm_wakeup_event(ws, 1 * MSEC_PER_SEC);
    return ret;
}
```
# Debug node
## Timer list dump
```
: # cat /proc/timer_list
cat timer_list
Timer List Version: v0.8				----------- global info
HRTIMER_MAX_CLOCK_BASES: 4              ------------------- number of clock types that support high resolution timers
now at 1316194170694 nsecs
cpu: 0									----------- per CPU section
 clock 0:								----------- clock0作为MONOTONIC使用ktime_get获取当前时间，是timerkeeper的xtime和wall_to_monotonic之和。
  .base:       0000000000000000
  .index:      0
  .resolution: 1 nsecs
  .get_time:   ktime_get				----------- name of kernel function used to fetch time from this clock.
  .offset:     0 nsecs					----------- difference, in nsecs, between this clock and the reference clock for this clock.
active timers:							----------- in turn, a display of all the active high resolution timers queued to that clock.
 #0: <0000000000000000>, tick_sched_timer, S:03			------------------- timer state (eg, S:01), avail states, OR-able:
																										+		      0 - inactive
																										+		      1 - enqueued
																										+		      2 - callback
																										+		      4 - pending
																										+		      8 - migrate
 # expires at 1316200000000-1316200000000 nsecs [in 5829306 to 5829306 nsecs]	-------------1 - absolute expiration time, range format (start - end)
																					+		  2 - relative expiration time, range format (start - end)
 #1: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1316889487882-1316890936881 nsecs [in 695317188 to 696766187 nsecs]
 #2: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1316864812476-1316904812476 nsecs [in 670641782 to 710641782 nsecs]
 #3: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1316881761747-1316921761747 nsecs [in 687591053 to 727591053 nsecs]
 #4: root_task_group, sched_rt_period_timer, S:03
 # expires at 1317100423679-1317100423679 nsecs [in 906252985 to 906252985 nsecs]
 #5: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1317125485393-1317165485393 nsecs [in 931314699 to 971314699 nsecs]
 #6: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1321444163423-1321484163423 nsecs [in 5249992729 to 5289992729 nsecs]
 #7: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1330188330139-1330188380139 nsecs [in 13994159445 to 13994209445 nsecs]
 #8: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1333112288571-1333112338571 nsecs [in 16918117877 to 16918167877 nsecs]
 #9: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1336440576573-1336480576573 nsecs [in 20246405879 to 20286405879 nsecs]
 #10: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1336440639000-1336480639000 nsecs [in 20246468306 to 20286468306 nsecs]
 #11: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1481182087752-1481282087752 nsecs [in 164987917058 to 165087917058 nsecs]
 #12: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1544829176226-1544929176226 nsecs [in 228635005532 to 228735005532 nsecs]
 #13: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1570686019089-1570726019089 nsecs [in 254491848395 to 254531848395 nsecs]
 #14: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1570694170704-1570734170704 nsecs [in 254500000010 to 254540000010 nsecs]
 #15: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1800000782131-1800000832131 nsecs [in 483806611437 to 483806661437 nsecs]
 #16: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1808731739110-1808731789110 nsecs [in 492537568416 to 492537618416 nsecs]
 #17: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1838523297379-1838623297379 nsecs [in 522329126685 to 522429126685 nsecs]
 #18: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1870693332788-1870733332788 nsecs [in 554499162094 to 554539162094 nsecs]
 #19: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1883558731197-1883658731197 nsecs [in 567364560503 to 567464560503 nsecs]
 #20: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 2139525468202-2139625468202 nsecs [in 823331297508 to 823431297508 nsecs]
 #21: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 4844211969315-4844212019315 nsecs [in 3528017798621 to 3528017848621 nsecs]
 #22: sched_clock_timer, sched_clock_poll, S:01
 # expires at 5683772776159-5683772776159 nsecs [in 4367578605465 to 4367578605465 nsecs]
 #23: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 10842619667759-10842719667759 nsecs [in 9526425497065 to 9526525497065 nsecs]
 #24: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 14439407341958-14439507341958 nsecs [in 13123213171264 to 13123313171264 nsecs]
 #25: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 50004806629164-50004806679164 nsecs [in 48688612458470 to 48688612508470 nsecs]
 #26: <0000000000000000>, it_real_fn, S:01
 # expires at 86406261202860-86406261202860 nsecs [in 85090067032166 to 85090067032166 nsecs]
 #27: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 86440098387848-86440098437848 nsecs [in 85123904217154 to 85123904267154 nsecs]
 #28: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 87641315361549-87641415361549 nsecs [in 86325121190855 to 86325221190855 nsecs]
 #29: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 500005995047445-500005995097445 nsecs [in 498689800876751 to 498689800926751 nsecs]
 #30: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 2148782446625180-2148782546625180 nsecs [in 2147466252454486 to 2147466352454486 nsecs]
 #31: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 2147483651846523487-2147483651846573487 nsecs [in 2147482335652352793 to 2147482335652402793 nsecs]
 #32: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 2147484899529977698-2147484899530027698 nsecs [in 2147483583335807004 to 2147483583335857004 nsecs]
 #33: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 9223372036854775807-9223372036854775807 nsecs [in 9223370720660605113 to 9223370720660605113 nsecs]
 #34: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 9223372036854775807-9223372036854775807 nsecs [in 9223370720660605113 to 9223370720660605113 nsecs]
 clock 1:
  .base:       0000000000000000
  .index:      1
  .resolution: 1 nsecs
  .get_time:   ktime_get_real
  .offset:     1540293283962085417 nsecs
active timers:
 #0: <0000000000000000>, hrtimer_wakeup, S:01
 # expires at 1540294600766781632-1540294600766831632 nsecs [in 610525521 to 610575521 nsecs]
 clock 2:
  .base:       0000000000000000
  .index:      2
  .resolution: 1 nsecs
  .get_time:   ktime_get_boottime
  .offset:     13540077675778 nsecs
active timers:
 clock 3:								
  .base:       0000000000000000
  .index:      3
  .resolution: 1 nsecs
  .get_time:   ktime_get_clocktai
  .offset:     1540293283962085417 nsecs
active timers:
  .expires_next   : 1316200000000 nsecs ------------------- hrtimer_cpu_base part
  .hres_active    : 1
  .nr_events      : 79489
  .nr_retries     : 40
  .nr_hangs       : 0
  .max_hang_time  : 0
  .nohz_mode      : 2					------------------- tick_sched part, NOHZ_MODE_INACTIVE(0)、NOHZ_MODE_LOWRES(1)、NOHZ_MODE_HIGHRES(2)。
  .last_tick      : 1316170000000 nsecs
  .tick_stopped   : 0
  .idle_jiffies   : 4295068912
  .idle_calls     : 247515
  .idle_sleeps    : 166274
  .idle_entrytime : 1316169668507 nsecs
  .idle_waketime  : 1316158200538 nsecs
  .idle_exittime  : 1316169668507 nsecs
  .idle_sleeptime : 1175107096549 nsecs
  .iowait_sleeptime: 5539063536 nsecs
  .last_jiffies   : 4295068912
  .next_timer     : 1316560000000
  .idle_expires   : 1316560000000 nsecs
jiffies: 4295068915
...

Tick Device: mode:     1
Broadcast device					---------------- Broadcast Tick Device
Clock Event Device: arch_mem_timer
 max_delta_ns:   111848106728
 min_delta_ns:   1000
 mult:           82463372
 shift:          32
 mode:           3
 next_event:     1316200000000 nsecs
 set_next_event: arch_timer_set_next_event_virt_mem
 shutdown: arch_timer_shutdown_virt_mem
 oneshot stopped: arch_timer_shutdown_virt_mem
 event_handler:  tick_handle_oneshot_broadcast
 retries:        16
tick_broadcast_mask: 00
tick_broadcast_oneshot_mask: b4

Tick Device: mode:     1
Per CPU device: 0					---------------- Per CPU Tick Device
Clock Event Device: arch_sys_timer
 max_delta_ns:   111848106728
 min_delta_ns:   1000
 mult:           82463372
 shift:          32
 mode:           3
 next_event:     1316200000000 nsecs
 set_next_event: arch_timer_set_next_event_virt
 shutdown: arch_timer_shutdown_virt
 oneshot stopped: arch_timer_shutdown_virt
 event_handler:  hrtimer_interrupt
 retries:        260
...
```

## Timer stats
Kernel config `CONFIG_TIMER_STATS`
> echo [0|1] > /proc/timer_stats

```
Timer Stats Version: v0.2
Sample period: 42.997 s
--依次是：timer count、进程号、进程名称、超时函数、启动timer函数
2112, 1300 xxx_log_agent hrtimer_start_range_ns (hrtimer_wakeup)
43, 1299 xxx_log_agent hrtimer_start_range_ns (hrtimer_wakeup)
43, 1298 xxx_log_agent hrtimer_start_range_ns (hrtimer_wakeup)
43, 1301 xxx_log_agent hrtimer_start_range_ns (hrtimer_wakeup)
5, 1881 sample hrtimer_start (alarmtimer_fired)
116, 0 swapper hrtimer_start_range_ns (tick_sched_timer)
14, 1355 xxx_ipsec_proxy hrtimer_start_range_ns (hrtimer_wakeup)
28, 0 swapper hrtimer_start (tick_sched_timer)
17D, 0 swapper cpufreq_interactive_timer_resched.constprop.8 (cpufreq_interactive_timer)
8, 794 at_ctl hrtimer_start_range_ns (hrtimer_wakeup)
6, 0 swapper run_timer_softirq (sync_supers_timer_fn)
4, 0 swapper DWC_TIMER_SCHEDULE (timer_callback)
4, 0 swapper DWC_TIMER_SCHEDULE (timer_callback)
4, 0 swapper DWC_TIMER_SCHEDULE (timer_callback)
4, 0 swapper br_hello_timer_expired (br_hello_timer_expired)
3, 613 zx_wdt_thread msleep (process_timeout)
2D, 4 kworker/0:0 neigh_periodic_work (delayed_work_timer_fn)
2D, 4 kworker/0:0 neigh_periodic_work (delayed_work_timer_fn)
2, 1366 adbd DWC_TIMER_SCHEDULE (timer_callback)
1, 0 swapper igmp6_group_queried (igmp6_timer_handler)
1, 1371 dm_ci hrtimer_start_range_ns (hrtimer_wakeup)
1, 0 swapper igmp6_group_queried (igmp6_timer_handler)
1, 0 swapper igmp6_group_queried (igmp6_timer_handler)
2464 total events, 57.306 events/sec
```
