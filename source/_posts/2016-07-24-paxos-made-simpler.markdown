---
layout: post
title: "Paxos Made Simpler"
date: 2016-07-24 15:42:15 -0700
comments: true
published: false
categories: 
---

```
class Replica(i, n)
  int id = i
  int seq = i
  int replicaNum = n
  int nextApplied = 0
  Instance[] instances = []
  int leader = null

  def run
    while true
      if leader == null || leader == id
        bool batchMode = false
        while leader == id
          if batchMode
            send Accept(id, seq, nextApplied, newValue)
            wait until received Acknowledge from majority replicas or Reject from one replica
            if received at least one Reject
              leader = null
              seq += replicaNum
            else
              send Commit(id, seq, nextApplied, newValue)
              nextApplied++
          else
            send Propose(id, seq, nextApplied)
            wait until received Promise from majority replicas or Reject from one replica
            if received at least one Reject
              leader = null
              seq += replicaNum
              continue
            if all value from promise are null
              // the next instance must be null as well since current instance hasn't been committed no one will write to the next instance
              batchMode = true
              value = newValue
            else
              value = Promise(_, _, seq, value) with the highest seq
            send Accept(id, seq, nextApplied, value)
            wait until received Acknowledge from majority replicas or Reject from one replica
            if received at least one Reject
              leader = null
              seq += replicaNum
            else
              send Commit(id, seq, nextApplied, value)
              nextApplied++
        receive m
          case Promise
          case Acknowledge
      else receive m
        switch m
          case Propose
          case Accept
          case Commit
      else
        // timeout: leader died
        leader = null

  def propose(m)
    if m.seq < seq
      send Reject
    else
      seq = m.seq
      leader = m.sender
      send Promise(id, m.instanceIndex, instances[m.instanceIndex].seq, instances[m.instanceIndex].value)

  def accept(m)
    if m.seq < seq
      send Reject
    else
      assert m.seq == seq
      instances[m.instanceIndex].seq = m.seq
      instances[m.instanceIndex].value = m.value
      send Acknowledge

  def commit(m)
    instances[m.instanceIndex].seq = m.seq
    instances[m.instanceIndex].value = m.value
    instances[m.instanceIndex].committed = true

class Instance
  int seq = null
  int value = null
  bool committed = false

class Propose(sender, seq, instanceIndex)
  int sender = sender
  int seq = seq
  int instanceIndex = instanceIndex

class Promise(sender, instanceIndex, seq, value)
  int sender = sender
  int instanceIndex = instanceIndex
  int seq = seq
  int value = value

class Accept(sender, seq, instanceIndex, value)
  int sender = sender
  int seq = seq
  int instanceIndex = instanceIndex
  int value = value

class Commit(sender, seq, instanceIndex, value)
  int sender = sender
  int seq = seq
  int instanceIndex = instanceIndex
  int value = value

class Acknowledge

class Reject
```
