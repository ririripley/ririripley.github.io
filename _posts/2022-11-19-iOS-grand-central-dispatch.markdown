---
author: ripley
comments: false
date: 2022-11-19 12:11:08+00:00
layout: post
slug: iOSDevelopment
title: iOS Grand Central Dispatch API
wordpress_id: 304
categories:
- Tech
tags:
description: iOS Grand Central Dispatch API
---
## **iOS - GCD API**
GCD related functions:  
1) dispatch_barrier_async    
Using the dispatch_barrier_async function, you can add a task to a concurrent dispatch queue     
at the time all the tasks in the queue are finished. When the task added by the dispatch_barrier_async function         
is finished, the concurrent dispatch queue will be back to normal.    
![avatar](https://ririripley.github.io/assets/img/dispatch_barrier_async.png)  
```
dispatch_async(queue, blk0_for_reading);   
dispatch_async(queue, blk1_for_reading);  
dispatch_async(queue, blk2_for_reading);    
dispatch_async(queue, blk3_for_reading);  
     
/*    
* Writing data    
*   
* From now on, all the tasks should read the updated data. */   
dispatch_barrier_async(queue, blk_for_writing);     
  
dispatch_async(queue, blk4_for_reading);     
dispatch_async(queue, blk5_for_reading);   
dispatch_async(queue, blk6_for_reading);   
dispatch_async(queue, blk7_for_reading);    
```   
2) dispatch_after   
```      
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 3ull * NSEC_PER_SEC);   
dispatch_after(time, dispatch_get_main_queue(), ^{  
NSLog(@"waited at least three seconds.");  
});  
/*    
* It does not mean that the block will be executed exactly after 3 seconds.  
* This code works the same as if you added the Block to the main dispatch queue     
by the dispatch_async function three seconds later. */    
```   
3)dispatch_once  
The dispatch_once function is used to ensure that the specified task will be executed only once     
during the application’s lifetime.  
```      
static dispatch_once_t onceToken;  
static int initialized = NO;      
dispatch_once(&onceToken, ^{    
   initialized = YES;   
});   
/*      
* With the dispatch_once function, it works safely even in a multithreaded environment        
* to initialized only once.     
 */   
```      
4)dispatch_apply  
The dispatch_apply function is used to add a Block to a dispatch queue for a number of times,       
and then it waits until all the added tasks are finished.      
```      
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);    
dispatch_apply(10, queue, ^(size_t index) { NSLog(@"%zu", index);    
});   
NSLog(@"done");    
/*  The result is something like:        
*   4  
*   1  
*   0  
*   3  
*   5  
*   2  
*   6  
*   8   
*   9   
*   7        
*   done  
 */   
“done” must always be last because the dispatch_apply function waits for all the tasks to be finished.   
```
5)dispatch_group    
When a Block is associated with a dispatch group, the Block has ownership of the dispatch group by the dispatch_retain   
function as if a Block were added to the dispatch queue. When the Block is finished,   
the Block releases the dispatch group by the dispatch_release function.  
5.1)dispatch_group_notify:      
When it detects that all the tasks are finished, the finalizing task is added to the dispatch queue.    
```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);      
dispatch_group_t group = dispatch_group_create();  // a dispatch group is created.      
dispatch_group_async(group, queue, ^{NSLog(@"blk0");});     
dispatch_group_async(group, queue, ^{NSLog(@"blk1");});     
dispatch_group_async(group, queue, ^{NSLog(@"blk2");});    
dispatch_group_notify(group, dispatch_get_main_queue(), ^{NSLog(@"done");});
dispatch_group_async(group, queue, ^{NSLog(@"blk3");});        
dispatch_release(group);    
The result will be like:  
blk1  
blk2  
blk0  
done   
“done” must always be last because the dispatch_group_notify function waits for all the tasks in the group to be finished.       
```
5.2)dispatch_group_wait:  
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);  
long result = dispatch_group_wait(group, time);  
The second argument for the dispatch_group_wait function is a timeout to specify how long it waits.  
When the dispatch_group_wait function didn’t return zero, some tasks that are associated with the dispatch group  
were still running even after the specified time had passed. When it returned zero, all the tasks were finished.      
Wait means that when the dispatch_group_wait function is called, it doesn’t return from the function.  
```
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);       
long result = dispatch_group_wait(group, time);    
if (result == 0) {  
    NSLog(@"done");    
/*   
* All the tasks that are associated with the dispatch group are finished */      
} else {  
    NSLog(@"unfinished");         
/*   
* some tasks that are associated with the dispatch group are still running. */        
}  
“done” must always be printed when all the tasks in the groups has finished.    
```
5.3)dispatch_group_enter, dispatch_group_leave:    
The combination of these two methods equal to the effect of dispatch_group_async.  
dispatch_group_enter: group.value++  
dispatch_group_leave: group.value--
when group.value == 0, notify to   
(???wait returns before notify ??)  
```
dispatch_group_t group = dispatch_group_create();  
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);    
dispatch_group_enter(group);    
dispatch_async(queue, ^{  
    NSLog(@"task 1");  
    dispatch_group_leave(group);    
});   
dispatch_group_enter(group);    
dispatch_async(queue, ^{    
    NSLog(@"task 2");  
    dispatch_group_leave(group);    
});  
fdispatch_group_notify(group, dispatch_get_main_queue(), ^{  
    NSLog(@"done");  
});  
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);    
dispatch_group_async(group, queue, ^{  
    NSLog(@"task 3");    
});  
```
6)dispatch_semaphore  
A dispatch semaphore is a semaphore with a counter, which is called a counting semaphore in multithreaded programming.   
dispatch_semaphore_wait:    
When the counter is zero, the execution waits. When the counter is one and more, or the counter becomes one and more while it is waiting,    
it decreases the counter and returns from the dispatch_semaphore_wait function.    
dispatch_semaphore_signal:   
It increases the counter of the dispatch semaphore.  
```
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);    
long result = dispatch_semaphore_wait(semaphore, time); if (result == 0) {    
/*    
* The counter of the dispatch semaphore was more than one.  
* Or it became one and more before the specified timeout.  
* The counter is automatically decreased by one. *  
* Here, you can execute a task that needs a concurrency control. */    
} else {    
/*    
* Because the counter of the dispatch semaphore was zero,    
* it has waited until the specified timeout. */    
}    
```





