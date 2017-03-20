---
layout: post
title: "Two Phase Locking"
date: 2017-03-19 21:07:50 -0700
comments: true
published: false
categories: 
---

#Concurrency Control
Concurrency control guarantees isolation in ACID. The goal of database concurrency control is to ensure that the execution of transactions is serializable, which means the concurrent execution of transactions produces the same output and has the same effect on the database as some serial execution.

A database transaction is a series of reads and writes on data records.

For a record A and two concurrent transactions Ti, Tj:
    If Ri happens before Wj, then the concurrent execution of Ti and Tj must produce the same output and has the same effect on the database as serial execution Ti, Tj.
    If Wi happens before Rj, then the serial execution must be Ti, Tj.
    If Wi happens before Wj, then the serial execution must be Ti, Tj.
    If Ri happens before Rj, then the serial execution can be Ti, Tj or Tj, Ti.
From above we can see that from RW and WW order on a record, we can infer the total order of two concurrent transactions if the execution is serializable.

To guarantee serializability, database concurrency control mechanisms need to make sure that for each common record that these two transactions access, the inferred orders must be the same.

If based on the read, write orders on record A, the inferred order is Ti, Tj and based on record B, the order is Tj, Ti, then the concurrent execution is not serializable as we cannot find a serial execution that produces the same output and has the same effect on the database.
Example:
A: Ri, Wj -> Ti, Tj
B: Wj, Wi -> Tj, Ti
Concurrency control mechanisms need to guarantee above example won't happen.
Two phase locking is proven to be able to guarantee serializability. Read will get the readlock and write will have the writelock. Once a transaction surrenders ownership of a lock, it can never obtain additional locks. The above example won't happen if we use two phase locking because it's a deadlock and one transaction must abort and restart.

Another concurrency control mechanism is timestamp ordering.
