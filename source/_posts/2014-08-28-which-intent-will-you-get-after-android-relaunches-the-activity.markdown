---
layout: post
title: "Which Intent Will You Get After Android Relaunches the Activity"
date: 2014-08-28 17:28:06 -0400
comments: true
categories: 
---

Imagine that you launch a `singleTop` activity using intent A and then launch it again using intent B. As a result, activity's `onNewIntent` method is called and inside this method you call `setIntent` to store the new intent. After that, Android decides to relaunch the activity. Now we have the question: which intent will you get after Android relaunches the activity? The original one(A) or the latest one(B)?

<!-- more -->

The answer is you may get either one, and it depends on why Android relaunches the activity. In some situations, Android will relaunch your activity because Android kills it(indeed the process) to reclaim memory and then you navigate back to it.
In this case, you will get the original intent instead of the latest one. Android will also relaunch your activity(it's the default behavior) due to the configuration change and you'll get the latest intent. Now let's discuss these two cases separately and see why we get different intents in these two cases.

## Killed Process
In Android, activities, taskes and processes are managed by `ActivityManagerService` and each activity has a corresponding `ActivityRecord`. As we can see, `ActivityRecord` stores lots of information related to the activity.

``` java
/** ActivityRecord.java **/

/**
 * An entry in the history stack, representing an activity.
 */
final class ActivityRecord {
    .....

    final ActivityManagerService service; // owner
    final ActivityInfo info;              // all about me
    final Intent intent;                  // the original intent that generated us
    TaskRecord task;                      // the task this is in
    ActivityRecord resultTo;              // who started this entry, so will get our reply
    Bundle  icicle;                       // last saved activity state

    .....
}
```

When you launch a `singleTop` activity using intent A, some process(maybe the Launcher) sends a request to `ActivityManagerService`(through Binder RPC) and `ActivityManagerService` creates an `ActivityRecord`. Now the intent variable of the new `ActivityRecord` object is A. Then `ActivityManagerService` may create the application process(if not exists) and asks the application process(through Binder RPC) to launch the specified `Activity` using intent A. In the application process, `ActivityThread` receives and handles the request. It creates an `Activity` object and calls lifecycle methods.

Then you launch the activity again using intent B and `ActivityManagerService` receives the request. It finds that there's already an `ActivityRecord` on the top of the stack, so it decides to deliver the new intent B to this existing activity. Here we have to notice that the intent variable of the `ActivityRecord` object is still A and it's immutable(final). After `ActivityManagerService` sends the new intent request, `ActivityThread` receives it and calls activity's `onNewIntent` method.

At some point, Android decides to kill the application process due to low memory. After that, you navigate back to this activity and `ActivityManagerService` has to relaunch it using the information stored in `ActivityRecord`. As you can see, `ActivityRecord` only stores the original inten(A) and the latest one(B) is lost. Why? Becuase the latest intent is only stored in the `Activity` and now the activity(process) is killed! So, in this case, you'll get the original intent after Android relaunches the activity.

## Configuration Change
After you launch the activity again using intent B, you decide to rotate the screen. Now we have the configuration change. When `ActivityManagerService` detects configuration changes, it relaunches the current activity by sending a request to `ActivityThread`. `ActivityThread` handles the request and relaunches the activity.

```java
/** ActivityThread.java **/
private void handleRelaunchActivity(ActivityClientRecord tmp) {
    ......

    ActivityClientRecord r = mActivities.get(tmp.token);
    Intent currentIntent = r.activity.mIntent;

    // Need to ensure state is saved.
    if (!r.paused) {
        performPauseActivity(r.token, false, r.isPreHoneycomb());
    }
    if (r.state == null && !r.stopped && !r.isPreHoneycomb()) {
        r.state = new Bundle();
        r.state.setAllowFds(false);
        mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state);
    }

    handleDestroyActivity(r.token, false, configChanges, true);
    
    r.activity = null;
    r.window = null;

    ......
    handleLaunchActivity(r, currentIntent);
}
```

As we can see, `ActivityThread` finds the activity and stores its latest intent in a variable. Then it destroys the activity and relaunches the activity using the latest intent. Because the application process is not killed in this case, the latest intent is not lost and we can get it after Android relaunches the activity.

## Conclusion
Because you don't know which intent you will get after Android relaunches the activity, you should use `savedInstanceState` to restore activity's state. In fact, `ActivityRecord` stores activity's `savedInstanceState` in its icicle variable, so it won't be lost!

## Reference
https://groups.google.com/forum/#!topic/android-developers/vrLdM5mKeoY<br />
The source code of ActivityThread, ActivityManagerService, ActivityStackSupervisor, ActivityStack, ActivityRecord, TaskRecord and ProcessRecord.
