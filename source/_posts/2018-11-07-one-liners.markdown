---
layout: post
title: "One-Liners"
date: 2018-11-07 22:06:28 -0800
comments: true
categories: 
---

1. [Get Java GC Related Application Stopped Time](#get_java_gc_stw_time)
<!-- more -->

## <a id="get_java_gc_stw_time"></a>Get Java GC Related Application Stopped Time
If Java application is started with `-XX:+PrintGCApplicationStoppedTime`, gc log will contain the application stopped time (STW) caused by different reasons (e.g. GC is just one common reason). This one-liner parses the gc log and only prints out the stopped time caused by GC. <br/>
Source: https://github.com/giltene/jHiccup
``` Bash
// gc log with +PrintGCTimeStamps
awk -F": " '/\[GC/ {t = $1; l = 1; while ((l == 1) && index($0, "Total time") == 0) { l = getline; } if (l == 1) {print t*1000.0, $3*1000.0;}}' gc.log

// gc log with +PrintGCTimeStamps and +PrintGCDateStamps
awk -F": " '/\[GC/ {t = $2; l = 1; while ((l == 1) && index($0, "Total time") == 0) { l = getline; } if (l == 1) {print t*1000.0, $4*1000.0;}}' gc.log
```
