---
layout: post
title: "android.os.Message.what"
date: 2014-01-19 18:36
comments: true
categories: Debug
---
Send an android.os.Message whose *what* is 0 and then remove it, which unexpectedly removes all posted Runnables

<!-- more -->

## Scenario
I write a Runnable and post it to the main thread, however the Runnable is not executed.

``` java
// callback from the background thread
private void callback() {
    mHandler.post(new Runnable() {
        // update UI in the main thread
    });
}
```

## Bug
After posting the Runnable, someone executes the following code:

``` java
// CUSTOM_MESSAGE_WHAT is 0
mHandler.removeMessages(CUSTOM_MESSAGE_WHAT)
```
and it not only removes the custom message but also removes all posted Runnables in the MessageQueue. The reason is that **android.os.Handler wraps every posted Runnable in an android.os.Message whose *what* is 0** which conflicts with the *what* value of the custom message.

``` java
/** Handler.java **/
public final boolean post(Runnable r) {
    return sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    // if not specified otherwise, m.what is 0 which is the default value
    return m;
}
```

## Solution
Never use 0 as the *what* value of custom messages

## Reference
The source code for android.os.Handler, android.os.Message and android.os.MessageQueue
