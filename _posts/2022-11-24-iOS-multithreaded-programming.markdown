---
author: ripley
comments: false
date: 2022-11-24 12:11:08+00:00
layout: post
slug: iOSDevelopment
title: iOS multithreaded programming
wordpress_id: 304
categories:
- Tech
tags:
description: iOS multithreaded programming
---
## **iOS - multithreaded programming**
1) NSLock      
Encapsulation of pthread_mutex(mutually excluded) in C++.      
- (BOOL)tryLock;   -> pthread_mutex_trylock // try to get the lock, return true if succeed, otherwise return false.(will not wait)     
- (BOOL)lockBeforeDate:(NSDate *)limit;  //   try to wait and get the lock before date, return false if failed.(wait for a while)  
- (void)lock;  -> pthread_mutex_lock // wait until get the lock.      
- (void)unlock;  -> pthread_mutex_unlock // release the lock     
```
NSLock *lock = [[NSLock alloc] init];  
for (int i = 0; i < 200000; i++) {  
    dispatch_async(dispatch_get_global_queue(0, 0), ^{  
        [lock lock];  
        self.mArray = [NSMutableArray array];  
        [lock unlock];  
    });  
}     
```   
2) NSRecursiveLock  
Encapsulation of pthread_mutex(mutually excluded) in C++（PTHREAD_MUTEX_RECURSIVE）.    
In the same thread, the lock can be obtained mutiple times, on when the lock is release can other thread     
obtain the lock.  
```      
NSRecursiveLock *lock = [[NSRecursiveLock alloc] init];  

dispatch_async(dispatch_get_global_queue(0, 0), ^{  
    static void (^recursiveMethod)(int);  
    recursiveMethod = ^(int value){  
        [lock lock];  
        if (value > 0) {  
            NSLog(@"value==%d",value);  
            recursiveMethod(value - 1);  
        }  
        [lock unlock];  
    };  
    recursiveMethod(5);  
});  
/*      
* The printing result is as follows:            
* value== 5  
* value== 4  
* value== 3  
* value== 2  
* value== 1     
 */  
```   
3)@synchronized  
```      
- (void)run {  
    @synchronized (self) {  
        NSLog(@"s1");  
        @synchronized (self) {  
            NSLog(@"s2");  
        }  
    }  
}  
The printing result is :  
s1   
s2   
```    
```    
@synchronized(obj) {  
    //action  
}  
It is equal to :  
objc_sync_enter(obj)    
//action   
objc_sync_exit(obj)
Explanation:    
objc_sync_enter will create a struct called SyncData as follows:  
//objc-sync.mm  
typedef struct SyncData {    
    struct SyncData* nextData;    
      
    DisguisedPtr<objc_object> object;  // associated with mutex      
  
    int32_t threadCount;  // number of THREADS using this block    
     
    recursive_mutex_t mutex;    
} SyncData;  

@synchronized(obj) will create SyncData which will be stored in a hash table using obj as the key. In the   
SyncData struct, a phread_mutex of PTHREAD_MUTEX_RECURSIVE type is associated with the object.     
objc_sync_enter -> phread_mutex lock   
objc_sync_exit -> phread_mutex unlock   
Based on the mechanism of @synchronized, it can be indicated that the efficiency of @synchronized is low due to the  
search in hash table.  
```      
4)condition variables
pthread_cond_wait   
pthread_cond_signal
```      
pthread_cond_wait:  
put the calling thread in the waiting thread list, and release the lock. When this methods return, obtain the lock again.  
pthread_cond_signal:  activate one of the waiting threads.    
pthread_cond_broadcast:  activate all of the waiting threads.    
```
4.1)NSCondition is actually an encapsulation of  pthread_cond.  
```
init: 
pthread_mutex_init(mutex, nil)  
pthread_cond_init(cond, nil)  
- (void)lock; // pthread_mutex_lock(mutex)    
- (void)unlock; // pthread_mutex_unlock(mutex)    
- (void)wait;   //   pthread_cond_wait(cond, mutex)      
- (BOOL)waitUntilDate:(NSDate *)limit; // pthread_cond_timedwait(cond, mutex, &timeout)      
- (void)signal; // pthread_cond_signal(cond)  
- (void)broadcast; // pthread_cond_broadcast(cond)      
```
4.2)NSConditionLock is actually an encapsulation of NSCondition.      
```
NSConditionLock makes use of NSCondition wait and broadcast function and while loop combined with a NSInteger   
variables to make sure the execution order of tasks in multiple threads.  
``` 
init  
internal var _cond = NSCondition()  
internal var _value: Int  
-void unlock;  
``` 
open func unlock(withCondition condition: Int) {    
_cond.lock()    
_thread = nil  
_value = condition  
_cond.broadcast()  
_cond.unlock()  
}  
``` 
-void lock;  
``` 
open func lock(whenCondition condition: Int, before limit: Date) -> Bool {  
    _cond.lock()  
    while _thread != nil || _value != condition {  
        if !_cond.wait(until: limit) {  
            _cond.unlock()  
            return false  
        }  
    }  
    _thread = pthread_self()  
    _cond.unlock()  
    return true  
}    
As you can see, lock and unlock methods of _cond only works inside the function to avoid race condition, the    
essense of the method lies in _cond.wait and while loop which makes the thread stuck.    
``` 
5)os_unfair_lock
``` 
The thread will be put into sleep while waiting to obtain the lock.  
```   
6)NSDistributedLock  
``` 
To compromise the conflict between multiple processes and exec.  
```     
7)NSCache & NSMutableDictionary & NSMutableArray
``` 
NSMutableDictionary and NSMutableArray are not thread-safe. They are not designed to be simultaneously read or written    
by multiple threads. So you have to use serial threads if multiple threads needs to visit the data.   
NSCache is thread-safe.  It makes use of some policies so as to delete some data if the application is short of memory.    
``` 
8)What is the difference between concurrency and parallelism?
```
5.2)dispatch_group_wait:  
Paramllelism means that multiple tasks are executing at the same time. That’s two processes running on a dual core, i.e.  
As for concurrency, mutiple tasks seem to be executing at the same time, but actually they are not. The execution of   
theses task take turns through context switch.  
```
9)What is the advantage of multithreaded programming?  
```
Multi-threaded programming makes full use of the multicore feature of processor to improve the efficiency.  
However, the creation of a thread requires certain memory and context switch between different threads takes time and  
memory.   
```
10)NSOperation & NSOperationQueue    
10.1) How to use NSOperations     
```
1)subclass NSOperation    
NSOperation: an abstract class that represents the code and data associated with a single task.  
- (void)start;  // The default implementation of this method updates the execution state of the operation    
  and calls the receiver’s main method.  
- (void)main;  //   The default implementation of this method does nothing.   
  You should override this method to perform the desired task.
      
Example of self-defined NSOoperation:      
@interface CustomerOperation : NSOperation      
@end    
  
@implementation CustomerOperation  
- (void)main{  
    if(!self.isCancelled){  
        for (int i = 0; i < 4; i++) {    
            [NSThread sleepForTimeInterval:2];    
            NSLog(@"%d--%@", i ,[NSThread currentThread]);    
        }  
    }  
}  
@end  

Usage:        

CustomerOperation *operation = [[CustomerOperation alloc]init];          
[operation start];  
       
Printing Result:  
2020-03-19 20:28:54.473676+0800 ThreadDemo[47267:12811915] 0--<NSThread: 0x600001289040>{number = 1, name = main}  
2020-03-19 20:28:56.474363+0800 ThreadDemo[47267:12811915] 1--<NSThread: 0x600001289040>{number = 1, name = main}  
2020-03-19 20:28:58.474708+0800 ThreadDemo[47267:12811915] 2--<NSThread: 0x600001289040>{number = 1, name = main}  
2020-03-19 20:29:00.476058+0800 ThreadDemo[47267:12811915] 3--<NSThread: 0x600001289040>{number = 1, name = main}    
As can be seen, the task is always executed in the same thread which calls the NSOperation method.  
```
```
2)NSInvocationOperation  
- initWithTarget:selector:object:  // Returns an NSInvocationOperation object initialized with the specified target and selector.  

Usage:     
-(void)invocationOperation{      
    NSInvocationOperation *operation = [[NSInvocationOperation alloc]initWithTarget:self selector:@selector(operation) object:nil];        
    [operation start];   
}  
-(void)operation{    
    for (int i = 0; i < 5; i++) {      
        [NSThread sleepForTimeInterval:2];      
        NSLog(@"%d--%@",i,[NSThread currentThread]);      
    }  
}  
Printing Result:  
2020-03-19 17:09:46.189458+0800 ThreadDemo[44995:12677738] 0--<NSThread: 0x600000ba9e40>{number = 1, name = main}    
2020-03-19 17:09:48.190629+0800 ThreadDemo[44995:12677738] 1--<NSThread: 0x600000ba9e40>{number = 1, name = main}  
2020-03-19 17:09:50.191219+0800 ThreadDemo[44995:12677738] 2--<NSThread: 0x600000ba9e40>{number = 1, name = main}  
2020-03-19 17:09:52.192556+0800 ThreadDemo[44995:12677738] 3--<NSThread: 0x600000ba9e40>{number = 1, name = main}  
2020-03-19 17:09:54.193900+0800 ThreadDemo[44995:12677738] 4--<NSThread: 0x600000ba9e40>{number = 1, name = main}  
As can be seen, the task is always executed in the same thread which calls the NSOperation method.    
```
```
3)NSBlockOperation  
  
+ (instancetype)blockOperationWithBlock:(void (^)(void))block; //  Creates and returns an NSBlockOperation object and adds the specified block to it.    
- addExecutionBlock:  // Adds the specified block to the receiver’s list of blocks to perform.        
Usage:    
-(void)blockOperationDemo{    
    NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{        
        for (int i = 0; i < 5; i++) {  
            [NSThread sleepForTimeInterval:2];  
            NSLog(@"%d--%@",i,[NSThread currentThread]);    
        }  
    }];  
    [operation start];    
}    
Printing Result :  
2020-03-19 17:19:38.673513+0800 ThreadDemo[45160:12689966] 0--<NSThread: 0x600001081100>{number = 1, name = main}    
2020-03-19 17:19:40.675074+0800 ThreadDemo[45160:12689966] 1--<NSThread: 0x600001081100>{number = 1, name = main}  
2020-03-19 17:19:42.676649+0800 ThreadDemo[45160:12689966] 2--<NSThread: 0x600001081100>{number = 1, name = main}  
2020-03-19 17:19:44.677073+0800 ThreadDemo[45160:12689966] 3--<NSThread: 0x600001081100>{number = 1, name = main}  
2020-03-19 17:19:46.677379+0800 ThreadDemo[45160:12689966] 4--<NSThread: 0x600001081100>{number = 1, name = main}  
    
As can be seen, when using the blockOperationWithBlock method alone, the task is always executed in the same thread which calls the NSOperation method.    
However, when using addExecutionBlock to add blocks for the operation, all the blocks will be added to a thread queue with   
default priority and these blocks may be executed in different threads(this is determined by the system).  
When all the blocks are finished, the operation will  mark its state as finished.  

Example:   
- (void)blockOperationDemo {  
    NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{  
        for (int i = 0; i < 2; i++) {  
            [NSThread sleepForTimeInterval:2];  
            NSLog(@"blockOperation--%@", [NSThread currentThread]);  
        }  
    }];  
    [operation addExecutionBlock:^{  
        for (int i = 0; i < 2; i++) {  
            [NSThread sleepForTimeInterval:2];  
            NSLog(@"executionBlock1--%@", [NSThread currentThread]);  
        }  
    }];  
    [operation addExecutionBlock:^{  
        for (int i = 0; i < 2; i++) {  
            [NSThread sleepForTimeInterval:2];  
            NSLog(@"executionBlock2--%@", [NSThread currentThread]);  
        }  
    }];  
    [operation addExecutionBlock:^{  
        for (int i = 0; i < 2; i++) {  
            [NSThread sleepForTimeInterval:2];  
            NSLog(@"executionBlock3--%@", [NSThread currentThread]);  
        }  
    }];  
    [operation addExecutionBlock:^{  
        for (int i = 0; i < 2; i++) {  
            [NSThread sleepForTimeInterval:2];  
            NSLog(@"executionBlock4--%@", [NSThread currentThread]);  
        }  
    }];  
    [operation addExecutionBlock:^{  
        for (int i = 0; i < 2; i++) {  
            [NSThread sleepForTimeInterval:2];  
            NSLog(@"executionBlock5--%@", [NSThread currentThread]);  
        }  
    }];    
    
    [operation start];    
}  
Printing Result:  
2020-03-19 17:40:08.102543+0800 ThreadDemo[45536:12708941] executionBlock4--<NSThread: 0x600002a1ab00>{number = 1, name = main}  
2020-03-19 17:40:08.102555+0800 ThreadDemo[45536:12709185] executionBlock2--<NSThread: 0x600002a57b80>{number = 8, name = (null)}  
2020-03-19 17:40:08.102555+0800 ThreadDemo[45536:12709191] executionBlock5--<NSThread: 0x600002ab8980>{number = 9, name = (null)}  
2020-03-19 17:40:08.102566+0800 ThreadDemo[45536:12709186] executionBlock3--<NSThread: 0x600002a7d440>{number = 4, name = (null)}  
2020-03-19 17:40:08.102570+0800 ThreadDemo[45536:12709184] executionBlock1--<NSThread: 0x600002a3aa80>{number = 6, name = (null)}  
2020-03-19 17:40:08.102576+0800 ThreadDemo[45536:12709187] blockOperation--<NSThread: 0x600002a7d600>{number = 5, name = (null)}  
2020-03-19 17:40:10.103970+0800 ThreadDemo[45536:12709187] blockOperation--<NSThread: 0x600002a7d600>{number = 5, name = (null)}  
2020-03-19 17:40:10.103970+0800 ThreadDemo[45536:12708941] executionBlock4--<NSThread: 0x600002a1ab00>{number = 1, name = main}  
2020-03-19 17:40:10.103970+0800 ThreadDemo[45536:12709185] executionBlock2--<NSThread: 0x600002a57b80>{number = 8, name = (null)}  
2020-03-19 17:40:10.103980+0800 ThreadDemo[45536:12709191] executionBlock5--<NSThread: 0x600002ab8980>{number = 9, name = (null)}  
2020-03-19 17:40:10.103971+0800 ThreadDemo[45536:12709186] executionBlock3--<NSThread: 0x600002a7d440>{number = 4, name = (null)}  
2020-03-19 17:40:10.103973+0800 ThreadDemo[45536:12709184] executionBlock1--<NSThread: 0x600002a3aa80>{number = 6, name = (null)}    
As can be seen, these blocks are executed in multiple threads.        
```
10.2) How to use NSOperationQueue  
```
[[NSOperationQueue alloc] init] -> Tasks added to this queue will be executed in subthreads.    
[NSOperationQueue mainQueue] -> Tasks added to this queue will be executed in the main thread.  
@property NSInteger maxConcurrentOperationCount; -> The maximum number of queued operations that can run at the same time.  If it is   
one, it means that the queue is a serial queue.    
```
```
- addOperation: // Adds the specified operation to the receiver.  
- addOperations:waitUntilFinished:  // Adds the specified operations to the queue. waitUntilFinished: If YES, the current thread is   
blocked until all of the specified operations finish executing. If NO, the operations are added to the queue and control returns immediately to the caller.      
  
- addOperationWithBlock:  Wraps the specified block in an operation and adds it to the receiver.    
// If the operations are all added to mainQueue,the tasks will be excecuted one by one instead of concurrently.        
```   
Communication among threads using NSOperation example:  
```   
-(void)threadCommunication{  
    NSOperationQueue *queue = [[NSOperationQueue alloc]init];    
    NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{      
        for (int i = 0; i < 4; i++) {   
            [NSThread sleepForTimeInterval:2];     
            NSLog(@"子线程--%@", [NSThread currentThread]);      
        }    
            
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{    
            for (int i = 0; i < 2; i++) {    
                [NSThread sleepForTimeInterval:2];   
                NSLog(@"主线程--%@", [NSThread currentThread]);      
            }     
        }];    
      
    }];  
    [queue addOperation:operation];    
}  
```   
Dependency of NSOperations:
```   
- addDependency: -> Makes the receiver dependent on the completion of the specified operation.    
- removeDependency: ->  Removes the receiver’s dependence on the specified operation.  
```   
```   
- (void)addDependency {  
  NSOperationQueue *queue = [[NSOperationQueue alloc] init];  
  
  NSBlockOperation *operation1 = [NSBlockOperation blockOperationWithBlock:^{  
  for (int i = 0; i < 2; i++) {  
  [NSThread sleepForTimeInterval:2];  
  NSLog(@"1---%@", [NSThread currentThread]);  
  }  
  }];  
  NSBlockOperation *operation2 = [NSBlockOperation blockOperationWithBlock:^{  
  for (int i = 0; i < 2; i++) {  
  [NSThread sleepForTimeInterval:2];  
  NSLog(@"2---%@", [NSThread currentThread]);  
  }  
  }];  
  NSBlockOperation *operation3 = [NSBlockOperation blockOperationWithBlock:^{  
  for (int i = 0; i < 2; i++) {  
  [NSThread sleepForTimeInterval:2];  
  NSLog(@"3---%@", [NSThread currentThread]);  
  }  
  }];  
    
  [operation1 addDependency:operation2];  
  [operation1 addDependency:operation3];  
  NSArray *opList = @[operation1,operation2,operation3];  
  NSArray *dependencies = [operation1 dependencies];  
  NSLog(@"dependencies-%@",dependencies);   
  [queue addOperations:opList waitUntilFinished:YES];  
  // operation1 depends on operation2 and operation3, therefore operation2 and operation3 will be executed before operation1.  
```   
Priority of NSOperation(added in the same NSOperationQueue)    
The execution order of different NSOperations should first obey the dependency relationship.  For NSOperations which are in ready state,    
the queue priority determines their execution order.    
```
@property NSOperationQueuePriority queuePriority;  -> The execution priority of the operation in an operation queue. NSOperationQueuePriorityNormal by default.  
```
Example :
```
- (void)addDependency {  
  NSOperationQueue *queue = [[NSOperationQueue alloc] init];  

  NSBlockOperation *operation1 = [NSBlockOperation blockOperationWithBlock:^{  
  for (int i = 0; i < 2; i++) {  
  [NSThread sleepForTimeInterval:2];  
  NSLog(@"1---%@", [NSThread currentThread]);  
  }  
  }];  
  NSBlockOperation *operation2 = [NSBlockOperation blockOperationWithBlock:^{  
  for (int i = 0; i < 2; i++) {  
  [NSThread sleepForTimeInterval:2];  
  NSLog(@"2---%@", [NSThread currentThread]);  
  }  
  }];  
  NSBlockOperation *operation3 = [NSBlockOperation blockOperationWithBlock:^{  
  for (int i = 0; i < 2; i++) {  
  [NSThread sleepForTimeInterval:2];  
  NSLog(@"3---%@", [NSThread currentThread]);  
  }  
  }];  
  [operation1 addDependency:operation2];  
  [operation1 addDependency:operation3];  
  operation1.queuePriority = NSOperationQueuePriorityVeryHigh;  
  NSArray *opList = @[operation1,operation2,operation3];  
  NSArray *dependencies = [operation1 dependencies];  
  NSLog(@"dependencies-%@",depedndencies);  
  [queue addOperations:opList waitUntilFinished:YES];  
  NSLog(@"end");  
  // The queuePriority should obey the dependency relationship. Therefore, even though operation1 has the highest priority  
  in the queue, it is still executed last.  
  }
```
Status of NSOperation  
```
isReady → isExecuting → isFinished  
isReady: what it depends on are all finished  
isFinished: meaning it is finshed or canceled  
```
11)What is the difference between OSSpinLock and mutual exclusive lock?
```
OSSpinLock : busy-waiting type, keep asking until successfully get the lock therefore occupying the CPU.  
Mutual exclusive lock like mutex: sleep-waiting type, blocked if failed trying to get the lock,   
then context switch happens and the current thread trying to get the lock will be put in the waiting threads queue.  
```  
12) What are the requirements for deadlock?
```  
Occupy the resource and won’t release it until finishing the task.        
The resource cannot be shared.  
More than one threads are waiting for the same resources.  
```    
13) What should be paid attention to while multithread programming?  
```    
Avoid deadlock + Avoid creating too many threads which may lead to large memory occupation and low efficiency due to context switch.  
```      
14) Examples of deadlock.
```      
dispatch_sync + serial thread    
Recursively call dispatch_once    
```
15) Thread Status  
![avatar](https://ririripley.github.io/assets/img/thread_status.png)  
### **Reference**  
https://juejin.cn/post/7031824170883383333    
https://juejin.cn/post/7017703842594684935  
https://blog.csdn.net/jimmy_w/article/details/38568425    
http://blog.chinaunix.net/uid-11572501-id-3456343.html  
https://juejin.cn/post/7031564506908082183  
https://juejin.cn/post/6844904085980725255  
https://juejin.cn/post/6844904097326465038   









