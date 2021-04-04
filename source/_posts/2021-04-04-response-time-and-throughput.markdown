---
layout: post
title: "Response Time and Throughput"
date: 2021-04-04 13:43:32 -0700
comments: true
categories:
---

For the discussion of this post, response time is the time between a service receiving a request and returning a response. It is the sum of waiting time and processing time. Waiting time is how long the request waits in queues before being processed. Processing time is the time to actually do the work of the request. Throughput is the number of requests that are completed per unit time. This post discusses how they can be possibly related.

<!-- more -->

## Lower Processing Time & Higher Throughput
If we reduce the processing time, the throughput might be higher. For example, the throughput is 10 requests per second if the processing time is 100ms CPU time assuming it's a single CPU system. If the processing time is reduced to 10ms CPU time, the throughput is increased to 100 requests per second.

## Higher Processing Time & Lower Throughput
This is the opposite of lower processing time & higher throughput. This is undesirable since we lose both processing time and throughput.

## Lower Processing Time & Lower Throughput
By optimizing the part of the system that's not the throughput bottleneck, the throughput keeps the same or is even lower. For example, the request processing time is 100ms CPU time and 10ms IO time. Since the request is CPU bound, reducing the IO time will bring down the overall processing time but throughput will still be the same: 10 requests per second. If reducing the IO time comes with the cost of increasing the CPU time (say CPU time becomes 101ms and IO time becomes 5ms), the throughput is actually lower even though the processing time is also lower.

## Higher Processing Time & Higher Throughput
If we can reduce the time spent on the bottleneck part of the system at the cost of increasing overall processing time, we can get higher throughput. Lets still use the request with 100ms CPU time and 10ms IO time as an example. If we can reduce the CPU time to 90ms at the cost of increasing the IO time to 30ms, the throughput is higher now even though the overall processing time is also higher.

## Higher Waiting Time & Higher Throughput
The maximum throughput we can get is 10 requests per second if the processing time is 100ms CPU time. To achieve that maximum throughput, we need to make sure the CPU is busy all the time (i.e. 100% utilization). In other words, there is always a request in the queue waitting to be processed as soon as the CPU finishes the current request. Basically we want requests to wait for CPU instead of CPU waiting for requests. By having requests wait for CPU, we incur some waiting time to achieve the higher (maximum) throughput.

## Lower Waiting Time & Lower Throughput
This is the opposite of higher waiting time & higher throughput. To make sure the waiting time is zero, we need to make sure the CPU is idle and there is no other requests in the queue by the time when the request is received. By having CPU idle for some time, the throughput will be lower than the maximum throughput.
