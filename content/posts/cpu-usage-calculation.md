---
title: "Cpu usage calculation using proc files"
date: 2021-06-12T14:44:55+03:00
draft: false
---

Experiment methodology
=================================================

**1.** Creating random(for obtaining different checksum results) dummy test .txt files according to given size and count

**2.** Creating shell script that runs given hash algorithm N times

**3.** Reading /proc/stat and /proc/pid/stat files at the beginning and end of the hash calculation(pid value is the process id of the shell script which executes checksum N times)

**4.** Calculating CPU usage of the process for the specific time interval

Calculation with proc files
===========================================================

**1.** Reading below values from /proc/pid/stat file: (# indicates the column number in the stat file)

* Utime (#14): Amount of time that this process has been scheduled in user mode, measured in clock ticks
* Stime (#15): Amount of time that this process has been scheduled in kernel mode, measured in clock ticks
* Cutime (#16): Amount of time that this process’s waited-for children have been scheduled in user mode, measured in clock ticks
* Cstime (#17): Amount of time that this process’s waited-for children have been scheduled in kernel mode, measured in clock ticks

**2.** Calculating process’ total cpu usage for beginning and end values:

* prev\_proc\_cpu_total = utime + stime + cutime + cstime
* after\_proc\_cpu_total = utime + stime + cutime + cstime

**Note:** Since shell commands(in this case hash calculation command) are executed as child process in the shell script, we have to consider to calculate the total cpu usage of the process.

**3.** Reading below values from the first line of /proc/stat file:

* User:Normal processes executing in user mode
* Nice:Niced processes executing in user mode
* System:Processes executing in kernel mode
* Idle:Twiddling thumbs
* Iowait:Waiting for I/O to complete
* Irq:Servicing interrupts
* Softirq:Servicing softirqs

**4.** Calculating total cpu usage for beginning and end values:

* prev\_cpu\_total = user + nice + system + idle + iowait + irq + softirq
* after\_cpu\_total = user + nice + system + idle + iowait + irq + softirq

**5.** Calculating total cpu usage of the process relative to total cpu usage:

* proc\_cpu\_usage = 100 * (after\_proc\_cpu\_total – prev\_proc\_cpu\_total) / (after\_cpu\_total – prev\_cpu\_total)

Alternative method
=========================================

    /usr/bin/time bash myshell.sh
    

Above command gives us accumulated times for process and its all child processes. Cpu usage percentage can also be extracted using this command.

Results
===================

**md5sum command**

Fixed file size: 10mb

| File count | CPU usage |
| --- | --- |
| 30  | %48 |
| 60  | %49 |
| 120 | %49 |
| 300 | %46 |
| 500 | %47 |

* * *

Fixed file size: 30mb

| File count | CPU usage |
| --- | --- |
| 30  | %49 |
| 60  | %49 |
| 120 | %48 |
| 300 | %48 |
| 500 | %48 |

* * *

**sha256sum command**

Fixed file size: 10mb

| File count | CPU usage |
| --- | --- |
| 30  | %49 |
| 60  | %49 |
| 120 | %49 |
| 300 | %49 |
| 500 | %48 |

* * *

Fixed file size: 30mb

| File count | CPU usage |
| --- | --- |
| 30  | %49 |
| 60  | %49 |
| 120 | %49 |
| 300 | %49 |
| 500 | %49 |
