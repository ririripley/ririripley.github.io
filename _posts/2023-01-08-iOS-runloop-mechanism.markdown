---
author: ripley
comments: false
date: 2023-01-08 12:11:08+00:00
layout: post
slug: iOSDevelopment
title: iOS Runloop Mechanism
wordpress_id: 304
categories:
- Tech
tags:
description: Runloop Mechanism
---
## **Runloop**  
1)What is Run Loop?     
A run loop is an event processing loop that you use to schedule work and coordinate the receipt of incoming events.     
The purpose of a run loop is to keep your thread busy when there is work to do and put your thread to sleep when there is none.    
The pseudocode of a run loop:    
```
function do_loop() {  
    do {  
        var message = get_next_message();  
        process_message(message);  
    } while (message != quit);  
}  
// By staying in the while loop, the function will not exit and keep running.    
```  
2)How Run Loops are used in iOS application?  
```  
int main(int argc, char * argv[]) {  
    @autoreleasepool {  
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));  
    }  
}       
```  
UIApplicationMain declaration:    
```  
UIKIT_EXTERN int UIApplicationMain(int argc, char *argv[], NSString * __nullable principalClassName, NSString * __nullable delegateClassName);  
```  
The main function is the entry of an iOS application, in the **UIApplicationMain** function, a run loop is initialized     
and running in the main thread. Therefore, **UIApplicationMain** function never returns.  
3)Source Code of Run Loop  
As you can see, there is **do while** implementation which causeds infinite loop inside a run loop.  
```  
// in DefaultMode  
void CFRunLoopRun(void) {    /* DOES CALLOUT */  
    int32_t result;  
    do {  
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);  
        CHECK_FOR_FORK();  
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);  
}     
```   
4)What is the relationship between run loop and thread?  
```  
(1) Each thread has only one unique run loop corresponding to it.       
(2) The run loop of the main thread has been initialized and started automatically, while runloops of sub threads require    
developers to create and start.           
(3) Run loop will be created by the system when being obtained for the first time and destroyed when its thread terminates.    
  
//Example  
  
// Foundation    
[NSRunLoop currentRunLoop]; // obtain the runloop corresponding to the current thread    
[NSRunLoop mainRunLoop]; // obtain the runloop corresponding to the main thread  
  

// Run the run loop  
[[NSRunLoop currentRunLoop] run];   
  
//Core Foundation  
CFRunLoopGetCurrent(); // obtain the runloop corresponding to the current thread  
CFRunLoopGetMain(); // obtain the runloop corresponding to the main thread     
```
5)How to get threads?   
```
Obtain main thread:    
pthread_main_thread_np()   
[NSThread mainThread]  
  
Obtain current thread:  
// You can only obtain the run loop of a sub thread inside the sub thread   
pthread_self()  
[NSThread currentThread]   
```
6)How is the run loop corresponding to a thread created?  
```  
// A global Dictionary，key : pthread_t， value : CFRunLoopRef  
static CFMutableDictionaryRef loopsDic;  
  
static CFSpinLock_t loopsLock;  
  
// Get the run roop corresponding to a thread  
CFRunLoopRef _CFRunLoopGet(pthread_t thread) {  
    OSSpinLockLock(&loopsLock);  
  
    if (!loopsDic) {  
        // How is the run loop corresponding to the main thread created?  
          
        // Create a global dictionary  
        loopsDic = CFDictionaryCreateMutable();  
        // create a run loop corresponding tot the main thread  
        CFRunLoopRef mainLoop = _CFRunLoopCreate();  
        // store in the dictionary: ( key: main thread   value: main run loop )   
        CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);  
    }  
      
    // try to get the loop from the global dictionary  
    CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));  
  
    // fail to get the loop in the dict, then creates one  
    if (!loop) {  
        loop = _CFRunLoopCreate();    
        CFDictionarySetValue(loopsDic, thread, loop);  
        // register a callback which dealloc the run loop when its thread terminates  
        _CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);  
    }  
  
    OSSpinLockUnLock(&loopsLock);  
    return loop;  
}  

CFRunLoopRef CFRunLoopGetMain() {  
    return _CFRunLoopGet(pthread_main_thread_np());  
}  

CFRunLoopRef CFRunLoopGetCurrent() {  
    return _CFRunLoopGet(pthread_self());  
}  
```  
7)Anatomy of A Run Loop  
![avatar](https://ririripley.github.io/assets/img/StructureOfARunLoop.jpeg)  
Source code :    
```  
/// Run in default mode  
void CFRunLoopRun(void) {  
CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);  
}  
  
/// Run in specified mode and set a time limit  
int CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean stopAfterHandle) {  
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);    
}  

/// Implementation of a run loop  
int CFRunLoopRunSpecific(runloop, modeName, seconds, stopAfterHandle) {  
  
    /// find the corresponding mode according to the mode name   
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(runloop, modeName, false);  
      
    /// If there is no source/timer/observer in the code, then return.  
    if (__CFRunLoopModeIsEmpty(currentMode)) return;  
      
    /// 1. Notify the Observers that it is about to entry the RunLoop  
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopEntry);  
      
    /// entry the run loop  
    __CFRunLoopRun(runloop, currentMode, seconds, returnAfterSourceHandled) {  
          
        Boolean sourceHandledThisLoop = NO;  
        int retVal = 0;  
        do {  
   
            /// 2. Notify the Observers that RunLoop is about to trigger timer handler  
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);  
              
            /// 3. Notify the Observers that RunLoop is about to trigger Source0 (non-port) handler  
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);  
              
            /// Execute the blocks added in the run loop     
            __CFRunLoopDoBlocks(runloop, currentMode);  
              
            /// 4. execute Source0 (non-port) handlers  
            sourceHandledThisLoop = __CFRunLoopDoSources0(runloop, currentMode, stopAfterHandle);  
              
            /// Execute the blocks added to the run loop  
            __CFRunLoopDoBlocks(runloop, currentMode);  
   
            /// 5. If there is any Source1 (port) which is ready, deal with Source1 directly and then jump to hanld the messages  
            if (__Source0DidDispatchPortLastTime) {  
                Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)  
                if (hasMsg) goto handle_msg;  
            }  
            
            /// Notify the Observers that RunLoop is about to sleep  
            if (!sourceHandledThisLoop) {  
                __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);  
            }  
              
            /// 7.Receive messages from specified port and store them in a buffer from which the outside can obtain.  
            /// __CFRunLoopServiceMachPort:     
            /// Run loop utilizes mach_msg method to receive messages. If no message is received, the thread will be put in sleep status.    
            /// until following events happen:  
            /// • A port-based event source  
            /// • Time is up for a times  
            /// • The time limit of the RunLoop is exceeded.  
            /// • Manually wake up by others       
               
            /// mach_msg method can be used to send or receive messages   
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {  
                mach_msg(msg, MACH_RCV_MSG, port); // thread wait for receive msg  
            }  
   
            /// 8. Notify the Observers that RunLoop has been waken up  
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);  
              
            /// handle the received messages    
            handle_msg:  
   
            /// 9.1 If time is up for a timer, then trigger the call back of the timer.  
            if (msg_is_timer) {  
                __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())  
            }   
   
            /// 9.2 If there is any block dispatch to main_queue, then execute the block    
            else if (msg_is_dispatch) {  
                __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);  
            }   
   
            /// 9.3 If there is any Source 1 event, deal with it  
            else {
                CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);  
                sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);  
                if (sourceHandledThisLoop) {  
                    mach_msg(reply, MACH_SEND_MSG, reply);  
                }  
            }  
            
            /// Execute the blocks added in the run loop  
            __CFRunLoopDoBlocks(runloop, currentMode);  
              
   
            if (sourceHandledThisLoop && stopAfterHandle) {  
                /// parameter: return after dealing with source events  
                retVal = kCFRunLoopRunHandledSource;  
            } else if (timeout) {  
                /// parameter: time limit  
                retVal = kCFRunLoopRunTimedOut;  
            } else if (__CFRunLoopIsStopped(runloop)) {  
                /// forcibly stopped by the outside caller  
                retVal = kCFRunLoopRunStopped;  
            } else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {  
                /// there is no source/timer/observer  
                retVal = kCFRunLoopRunFinished;  
            }  
              
            /// If there exists no time limit && there exits source events in current mode && runloop is not forcibly stopped,    
            /// then continue the loop   
        } while (retVal == 0);  
    }  
    
    /// 10. Notify the observers that RunLoop is about to exit  
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);  
}  
```
8)Process Of Run Loop:    
![avatar](https://ririripley.github.io/assets/img/inputSourceSequence.png)  
The figure can be summarized in this example:  
The main thread of the application runs, main run loop is created automatically in kCFRunLoopDefaultMode.      
At this time, assume that we create a timer which will fire in 10 seconds.   
Currently the run loop of the main thread of the current application is put in sleep.   
5 seconds later, we touch the screen. IOKit.framework will generate correspinding IOHIDEvent.       
Then the sysem will send the IOHIDEvent to SpringBoard.app(A running processs).    
```   
SpringBoard.app only receive Button, Touch, Acceleration Event and dispatch these events based on current status of the screen:    
1)If there is no application running active in the screen(e.g. you are browsing in the desktop), then the source 0 handler of the run loop     
of SpringBoard main thread will be triggered, which will dispatch the events to desktop system to handle.   
2) If there is any application active, SpringBoard.app will send the events to the corresponding App.app through mach port.    
On receiving messages from mach port, the run loop of the main thread of the App.app will wake up and then handle source1 callback: **IOHIDEventSystemClientQueueCallback**.     
__IOHIDEventSystemClientQueueCallback() will generate source0 which further trigger source0 callback: **_UIApplicationHandleEventQueue**.   
_UIApplicationHandleEventQueue() will convert IOHIDEvent to UIEvent and dispatch them to UIWindow by [UIApplication sendEvent:] method. Then comes the hit-test    
procress to find the first responder and dispatching UIEvent to it.   
``` 
SpringBoard.app will send messages to current app. The run loop of the main thread of the app wakes up.   
Then the run loop notifies the observers that it is about to trigger timer handler.    
After that, the run loop notifies the observers that it is about to trigger source 0 handler. Then the run loop executes the source 0 handler    
which is _UIApplicationHandleEventQueue call.    
Then the run loop executes the blocks added in the run loop.   
Next, the run loop check if there is any Source1 (port) which is ready.    
```
1)If the answer is yes, the run loop deals with Source1 directly and then check if the run roop should exit.    
If the answer is yes, the run loop exits. Otherwise, the run loop continues.   
2)If the answer is no, the run loop will be put into sleep until being waken up.    
``` 
As time passes by, time's up for the timer which wakes up the run loop and the run loop executes the timer handler. After that,    
the run loop executes blocks added to the run loop. After that, again, the run loop check if it should exit.   
If the answer is yes, the run loop exits. Otherwise, the run loop continues.   
The process chain can be described as follows:      
``` 
sleep -> source0 -> wake up -> source1 - > source 0 -> sleep    
```
8.1)Mode:   
There are 5 modes in total.   
```
1.kCFRunLoopDefaultMode：Default mode of the App, usually the main thread works in kCFRunLoopDefaultMode by default.   
2. UITrackingRunLoopMode：The mode set while tracking in controls takes place. e.g.Used to track the touch event of ScrollView so as to ensure scrolling the UI without being influenced by other modes   
3. UIInitializationRunLoopMode: The initial mode when the app starts. This mode is no more used later.   
4. GSEventReceiveRunLoopMode: A mode which accepts system events. It is an internal mode which the outside seldom requires.   
5. kCFRunLoopCommonModes: Objects added to a run loop using this value as the mode are monitored by all run loop modes that have been declared as a member of the set of “common” modes with CFRunLoopAddCommonMode.   
```
A run loop includes multiple modes. Each mode include multiple mode items.     
Every time calling **CFRunLoopRunSpecific**, we have to specify one mode which is called the Current Mode.    
We can only swtich to another mode by exiting the run loop. In this way, mode items in different modes are seperated and will not    
interfere each other.   
8.2)Mode Item:   
![avatar](https://ririripley.github.io/assets/img/StructureOfARunLoop.jpeg)   
Source, timer, observer are all mode items. An item can simultaneously added to multiple modes. However, it makes no difference     
between adding an item to the same mode one time or multiple times. If there is no item added to one mode at all, the run loop      
will exit.   
8.2.1)Source   
```   
Source0：touch event by the users (non-port based)   
Source1：port-based  communication among kernel, threads   
(Source 1 sometimes will dispatch some event in source 0)   
```   
8.3)Mode Switch:     
For example, when scrolling the textfield, the mode of the run loop will switch to **UITrackingRunLoopMode** automatically.    
When the scrolling stops, the run loop will switch back to NSDefaultRunLoopMode.   
   
8.4)CFRunLoopObserver:   
A CFRunLoopObserver provides a general means to receive callbacks at different points within a running run loop.    
In contrast to sources, which fire when an asynchronous event occurs, and timers, which fire when a particular time passes,    
observers fire at special locations within the execution of the run loop, such as before sources are processed or before the run loop goes to sleep,   
waiting for an event to occur.    
Observers can be either one-time events or repeated every time through the run loop’s loop.   
Each run loop observer can be registered in only one run loop at a time, although it can be added to multiple run loop modes within that run loop.   
```
- (void)viewDidLoad   
{   
  [super viewDidLoad];   
  self.view = [[MyView alloc] initWithFrame:(CGRectMake(20, 20, 100, 100))];   
  
     
  /// create a CFRunLoopObserver which monitor all types of status of the run loop    
  CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(CFAllocatorGetDefault(), kCFRunLoopAllActivities, YES, 0,    
    ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {   
        switch (activity) {   
              case kCFRunLoopEntry:   
              NSLog(@"enter RunLoop ");   
                  break;   
              case kCFRunLoopBeforeTimers:   
              NSLog(@"RunLoop is about to dealt with Timers");   
                   break;   
              case kCFRunLoopBeforeSources:   
              NSLog(@"RunLoop is about to deal with Sources");   
                  break;   
              case kCFRunLoopBeforeWaiting:   
              NSLog(@"RunLoop is about to sleep");   
                  break;   
              case kCFRunLoopAfterWaiting:   
              NSLog(@"RunLoop has waken up");   
                  break;   
              case kCFRunLoopExit:   
              NSLog(@"RunLoop has exit");   
                  break;   
              default:   
                  break;   
        }   
  });   
   
    // add observer to current run loop in kCFRunLoopDefaultMode mode    
    CFRunLoopAddObserver(CFRunLoopGetCurrent(), observer, kCFRunLoopDefaultMode);   
    CFRelease(observer);   
  }   
   
```   
9)AutoReleasePool drain in the run loop   
```   
1)In the run loop of the main thread of UIApplication, two observers are registered.        
2) The first observer will elicit _wrapRunLoopWithAutoreleasePoolHandler() when noticing the upcoming         
entering the loop. This observer has the highest priority which makes sure its handler is executed first.         
3) The second observer monitors the event of BeforeWaiting(sleep) and Exit(exit the loop).      
When noticing BeforeWaiting, it executes _objc_autoreleasePoolPop to pop the old pool and _objc_autoreleasePoolPush to push a new pool.          
When noticing Exit, it executes _objc_autoreleasePoolPop() to pop the pool.       
This observer has the lowest priority which makes sure that its handlers are executed after all other handlers.        
An autoreleasePool is created in the very begnning of the run loop and released before the run loop exit, which means that     
developers don't need to do such work themselves.      
```
10)Relationship between GCD and run loop   
There is no direct relationship between run loop and GCD. When dispatch_async(dispatch_get_main_queue(), <#^(void)block#>) is called,    
libDispatch will send messages to the run loop of the main thread by mach port. In this way, the run loop will wake up on receiving messages.     
The run loop will obtain the block from messages and execute the block in **CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE** handler.    
However, this operation will work when block is added to the main thread. When it comes to dipatching events to other threads, libDispatch     
handles the work.    
In summarize, the run loop of the main thread will ask whether there is any work from GCD, if there is, the run loop will execute.   
11)Mach_msg   
Mach provides IPC (inter-process communication) function.   
Mach_msg() is a system call which will put current thread in sleep.    
What does sleep mean?    
Thread leaves the CPU and stops its execution for a period of time. During this time, it's not consuming CPU time, so the CPU can be executing other tasks.    
When Thread is sleeping and is interrupted, the method throws an InterruptedException exception immediately and doesn't wait until the sleeping time finishes.   
Voluntary context switches occur when the thread sleep. At the same time, some port will be observed. When Kernel send messages to the port, the thread will wake up restarting the run     
loop and continue dealing with events.   
12)How does Gesture Recognizer work?   
As we mentioned above, the source 0 handlers will call **UIApplicationHandleEventQueue** method.    
_UIApplicationHandleEventQueue() will convert IOHIDEvent to UIEvent and dispatch them to UIWindow by **UIApplication sendEvent:** method. Then comes the hit-test   
procress to find the first responder and dispatching UIEvent to it.    
During this process, if a GR is recognized, it can prevent the delivery of touchesEnded:withEvent: to the view and instead call   
**touchesCancelled:withEvent:**. Then the GR will be marked as dirty.    
A observer has been registered to the run loop which monitors**kCFRunLoopBeforeWaiting** event.    
The handler of the observer is **UIGestureRecognizerUpdateObserver()** which will dispose all the dirty GRs and execute      
the corresponding callbacks of the GRs. Also, the **UIGestureRecognizerUpdateObserver()** method also handles       
stauts changes of GRs (i.e. initilization / destruction / recognition status).      
13)How does NSTimer work?      
13.1)Basic Mechanism:       
```
NSTimer will record the fire date. The run loop will check whether the NSTimer reaches the fire date and decide whether to excute      
the timer handler. In other words, the working of NSTimer relies on a run loop.     
```    
13.2)Example:     
In the following example, MyViewController instance maintains a strong reference to target the NStimer instance     
and the NStimer instance maintains a strong reference the MyViewController instance, which creates recyclic reference leading     
to memory leak.    
```    
@implementaion MyViewController     
- (void)viewDidLoad    
{    
    [super viewDidLoad];        
    self.timer = [NSTimer scheduledTimerWithTimeInterval:10.0         
                    target:self        
                    selector:@selector(timerFired)         
                    userInfo:nil repeats:YES];        
}    
- (void) timerFired {    
    [self.view setNeedsDisplay];    
    NSLog(@"timer: %@",  @"timer fired");    
}    
@end     
```
13.3)    
The following method creates a timer and schedules it on the current run loop in the default mode. In other words, if the run loop     
switch to another mode, the timer does not work before switching back to the default mode.     
```    
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti     
                                     target:(id)aTarget     
                                   selector:(SEL)aSelector     
                                   userInfo:(id)userInfo     
                                    repeats:(BOOL)yesOrNo;    
// The timer maintains a strong reference to target until it (the timer) is invalidated.    
```    
13.4)    
The following method stops the timer from ever firing again and requests its removal from its run loop.     
This is the only method for the NSRunLoop object removes its strong reference to the timer,     
either just before the invalidate method returns or at some later point.    
If it was configured with target and user info objects, the receiver removes its strong references to those objects as well.    
```
- (void)invalidate;    
```
13.5) Recyclic Reference of NSTimer     
In MyViewController example, we can add such a method to avoid memory leak.     
```
- (void) stop {    
    [self.timer invalidate];    
  }    
```  
But the premise is that **stop** method must be called.     
In order to avoid the recylic reference, there are 3 ways which we can utilize in our code. The core of these 3 ways is to     
avoid strong reference from NStimer to ViewController(NStimer owner).    
```      
1) Capsulate NSTimer in a class wrapper.     
viewController -strong--> class wrapper --strong--> NStimer ---strong --> class wrapper     
+ [viewController dealloc] method calling [NStimer invalidate].    
2) Using block parameter provided by system api  // Recommended Way     
ViewController1 intance object ——strong——> NSTimer instance object ——strong——> NSTimer class object;    
NSTimer instance object ——strong——> block (in the heap) ——weak——> ViewController1 intance object;    
+ [viewController dealloc] method calling [NStimer invalidate].    
3) Using Proxy      
ViewController1  instance object —strong—> NSTimer instance object —strong—>  PFProxy instance object —weak—> ViewController1  instance object    
+ [viewController dealloc] method calling [NStimer invalidate] + Runtime Methos Normal Fowaring.    
```  
13.5.1) NSTimer wrapper(gurantee the calling of **invalidate** method)     
```    
//PFTimer.h文件    
#import <Foundation/Foundation.h>    
@interface PFTimer : NSObject    
- (void)startTimer;    
- (void)stopTimer;    
@end    
    
//PFTimer.m文件    
#import "PFTimer.h"    
@implementation PFTimer {    
    NSTimer *_timer;    
}    
    
- (void)stopTimer{    
    if (_timer == nil) {    
        return;    
    }    
    [_timer invalidate];    
    _timer = nil;    
}    
    
- (void)startTimer{    
    _timer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(work) userInfo:nil repeats:YES];    
}    
    
- (void)work{    
    NSLog(@"正在计时中。。。。。。");    
}    
    
- (void)dealloc{     
   NSLog(@"%s",__func__);    
    [_timer invalidate];    
    _timer = nil;    
}    
@end    
    
//ViewController1.m文件    
#import "ViewController1.h"    
#import "PFTimer.h"    
    
@interface ViewController1 ()    
@property (nonatomic, strong) PFTimer *timer;    
    
@end    
@implementation ViewController1    
    
- (void)viewWillDisappear:(BOOL)animated {    
    [super viewWillDisappear:animated];    
}    

- (void)viewDidLoad {    
    [super viewDidLoad];    
    self.title = @"VC1";    
    self.view.backgroundColor = [UIColor whiteColor];    
        
    PFTimer *timer = [[PFTimer alloc] init];    
    self.timer = timer;    
    [timer startTimer];    
}    
    
- (void)dealloc {    
    [self.timer stopTimer];    
    NSLog(@"%s",__func__);    
}    
    
```    
13.5.2) Using Block in the api      
Three NSTimer APIs with block parameter:    
In these APIs, the timer itself is passed as the parameter to this block when executed to aid in avoiding cyclical references.    
```    
+ (NSTimer *)timerWithTimeInterval:(NSTimeInterval)interval repeats:    
  (BOOL)repeats block:(void (^)(NSTimer *timer))block    
  API_AVAILABLE(macosx(10.12), ios(10.0), watchos(3.0), tvos(10.0));    
    
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)interval repeats:    
  (BOOL)repeats block:(void (^)(NSTimer *timer))block    
  API_AVAILABLE(macosx(10.12), ios(10.0), watchos(3.0), tvos(10.0));    
    
- (instancetype)initWithFireDate:(NSDate *)date interval:    
  (NSTimeInterval)interval repeats:(BOOL)repeats block:(void (^)(NSTimer *timer))block    
  API_AVAILABLE(macosx(10.12), ios(10.0), watchos(3.0), tvos(10.0));    
```    
Example:    
```    
// NSTimer+PFSafeTimer.h文件    
#import <Foundation/Foundation.h>    
@interface NSTimer (PFSafeTimer)    
+ (NSTimer *)PF_ScheduledTimerWithTimeInterval:(NSTimeInterval)timeInterval block:    
(void(^)(void))block repeats:(BOOL)repeats;    
@end    

// NSTimer+PFSafeTimer.m文件    
#import "NSTimer+PFSafeTimer.h"    
@implementation NSTimer (PFSafeTimer)    
    
+ (NSTimer *)PF_ScheduledTimerWithTimeInterval:(NSTimeInterval)timeInterval block:(void(^)(void))block repeats:(BOOL)repeats {    
    // Here, we use [block copy] to copy the block to heap so as to guarantee the existence of the block    
    // Here, the NStimer instance object have strong refer to NSTimer Class object (Since NSTimer Class object does not need to be released, it is ok)    
    return [NSTimer scheduledTimerWithTimeInterval:timeInterval target:self selector:@selector(handle:) userInfo:[block copy] repeats:repeats];    
}    
    
+ (void)handle:(NSTimer *)timer {    
    void(^block)(void) = timer.userInfo;    
    if (block) {    
        block();    
    }    
}    
@end    
    
// ViewController1.m 文件    
#import "ViewController1.h"    
#import "PFTimer.h"    
#import "NSTimer+PFSafeTimer.h"    
    
@interface ViewController1 ()    
@property (nonatomic, strong) NSTimer *timer1;    
@end    
    
@implementation ViewController1    
    
- (void)viewWillDisappear:(BOOL)animated {        
    [super viewWillDisappear:animated];    
}    
    
- (void)viewDidLoad {    
    [super viewDidLoad];    
    self.title = @"VC1";    
    self.view.backgroundColor = [UIColor whiteColor];    
    __weak typeof(self) weakSelf = self;    
    self.timer1 = [NSTimer PF_ScheduledTimerWithTimeInterval:1.0 block:^{       
        __strong typeof(self) strongSelf = weakSelf;    
        [strongSelf timerHandle];       
    } repeats:YES];    
}    
    
- (void)timerHandle {    
        
     NSLog(@"正在计时中。。。。。。");    
}    
    
- (void)dealloc {    
    [self.timer invalidate];    
    NSLog(@"%s",__func__);    
}        
```    
The reference graph can be described as follows:    
```    
ViewController1 intance object ——strong——> NSTimer instance object ——strong——> NSTimer class object;    
NSTimer instance object ——strong——> block (in the heap) ——weak——> ViewController1 intance object;                
```    
In this way, the reference count of the ViewController1 intance object is only 1. The reference count of     
the NSTimer is 2. (Both the ViewController1 intance object and NSRunloop object have strong reference to the NStimer instance object.)      
And when the ViewController1 intance object is released by the outside owrner, it calls the dealloc method     
which then invalidate the NSTimer leading to that the run loop releases the NSTimer and NSTimer release the block. No memory leak exists.      
13.5.3) Using NSProxy    
```      
//PFProxy.h 文件    
#import <Foundation/Foundation.h>    
@interface PFProxy : NSProxy    
- (instancetype)initWithObjc:(id)object;    
+ (instancetype)proxyWithObjc:(id)object;    
@end    
    
//PFProxy.m 文件    
@interface PFProxy()    
@property (nonatomic, weak) id object;    
@end    
    
@implementation PFProxy    
- (instancetype)initWithObjc:(id)object {    
    self.object = object;    
    return self;    
}    
    
+ (instancetype)proxyWithObjc:(id)object {    
    return [[self alloc] initWithObjc:object];    
}    
    
- (void)forwardInvocation:(NSInvocation *)invocation {    
    if ([self.object respondsToSelector:invocation.selector]) {    
        [invocation invokeWithTarget:self.object];    
    }    
}    
    
- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {    
    return [self.object methodSignatureForSelector:sel];    
}    
@end    
    

//ViewController1.m 文件    
#import "ViewController1.h"    
#import "PFProxy.h"    
    
@interface ViewController1 ()    
@property (nonatomic, strong) NSTimer *timer2;    
@end    
    
@implementation ViewController1    
    
- (void)viewWillDisappear:(BOOL)animated {    
    [super viewWillDisappear:animated];    
}    
    
- (void)viewDidLoad {    
    [super viewDidLoad];    
    self.title = @"VC1";    
    self.view.backgroundColor = [UIColor whiteColor];    
    PFProxy *proxy = [[PFProxy alloc] initWithObjc:self];    
    self.timer2 = [NSTimer scheduledTimerWithTimeInterval:1.0 target:proxy selector:@selector(timerHandle) userInfo:nil repeats:YES];    
}    
    
- (void)timerHandle {        
     NSLog(@"正在计时中。。。。。。");    
}    
    
- (void)dealloc {    
    [self.timer2 invalidate];    
    self.timer2 = nil;    
    NSLog(@"%s",__func__);    
}    
@end    
```    
Let me explain how the proxy works. When time is up for the NStimer instance object, the **proxy** target receive     
message **timeHandle**. The PFProxy class object utilizes dynamic method calling of the Objective-C runtime mechanism.     
By **Normal-Forwarding** messages to **self.object** which is exactly the ViewController1 instance object. The timerHandle is called.      
The reference graph can be described as follows:    
ViewController1  instance object ——> NSTimer instance object ——>  PFProxy instance object   ====> ViewController1  instance object    
Therefore, once the outside owner release the ViewController1  instance object, the ViewController1  instance object dealloc leading to      
the NSTimer instance object invalidating. The invalidation of the NSTimer instance object remove itself from the run loop and remove its     
strong reference to the proxy object. In this way, no memory leak exits.     
    
14)Mechanism of performSelector:withObject:afterDelay:      
Source Code Analysis :     
```    
- (void)performSelector:(SEL)aSelector    
             withObject:(id)argument    
             afterDelay:(NSTimeInterval)seconds {    
    // 1) Get the current run loop object              
    NSRunLoop *loop = [NSRunLoop currentRunLoop];    
    GSTimedPerformer *item;    
        
    // 2) Initialize a GSTimedPerformer instance object which holds information about     
    // messages which are due to be sent to objects at a particular time.    
        
    // item reference count + 1    
    item = [[GSTimedPerformer alloc] initWithSelector:aSelector    
                                               target:self    
                                             argument:argument    
                                                delay:seconds];    
                   
    // 3) Runloop own _timedPerformers array, add the GSTimedPerformer instance object to _timedPerformers array    
    // item reference count + 1                                                
    [[loop _timedPerformers] addObject:item];    
        
    // item reference count - 1    
    RELEASE(item);    
        
    // 4) Add the timer in the GSTimedPerformer instance object to run loop     
    // run loop own timer    
    [loop addTimer:item->timer forMode:NSDefaultRunLoopMode];    
}    
```
Source Code of GSTimedPerformer:    
```    
@interface GSTimedPerformer: NSObject {    
    @public    
    SEL selector;    
    id target;    
    id argument;    
    NSTimer *timer;    
}    
    
- (void)fire;    
    
- (id)initWithSelector:(SEL)aSelector    
  target:(id)target    
  argument:(id)argument    
  delay:(NSTimeInterval)delay;    
      
- (void)invalidate;    
@end    
    
@implementation GSTimedPerformer    
- (void)dealloc {    
      [self finalize];    
      TEST_RELEASE(timer);    
      // release target    
      RELEASE(target);    
      RELEASE(argument);    
      [super dealloc];    
  }    
    
- (void)finalize {    
    [self invalidate];    
  }    
    
- (void)fire {    
      DESTROY(timer);    
      [target performSelector:selector withObject:argument];    
      [[[NSRunLoop currentRunLoop] _timedPerformers] removeObjectIdenticalTo:self];    
  }    
    
- (id)initWithSelector:(SEL)aSelector    
  target:(id)aTarget    
  argument:(id)anArgument    
  delay:(NSTimeInterval)delay {    
      self = [super init];    
      if (self != nil) {    
          selector = aSelector;    
          // own target    
          target = RETAIN(aTarget);    
          argument = RETAIN(anArgument);    
          timer = [[NSTimer allocWithZone:NSDefaultMallocZone()] initWithFireDate:nil    
                    interval:delay    
                    target:self    
                    selector:@selector(fire)    
                    userInfo:nil    
                    repeats:NO];    
    }    
      return self;    
}    
    
- (void)invalidate {    
      if (timer != nil) {    
          [timer invalidate];    
          DESTROY(timer);    
      }    
  }    
  @end    
```    
The reference graph can be described as follows:    
![avatar](https://ririripley.github.io/assets/img/performSelector-delay-reference-graph.png)    
When time iis up, GSTimedPerformer call fire method which send messages to the target executing the method. Then     
the run loop's _timedPerformers array remove the reference to the GSTimedPerformer instance object.     
GSTimedPerformer is destroyed leading to dealloc method which further invokes finalize method. Finalize method invalidates the times.     
Since the run loop remove reference to the timer. Timer is destroyed then. No memory leak exists.      
Here comes another question: how to call performSelector:withObject:afterDelay: in sub threads?      
```    
1) We need to manually run the run loop in sub thread, otherwise, the performSelector:withObject:afterDelay: does not work.       
 [self performSelector:@selector(mySelector) withObject:myObject afterDelay:myDelay];    
 [[NSRunLoop currentRunLoop] run];    
2) Use dispatch mechanism     
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, self.interval);    
 dispatch_after(time, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0), ^{    
     [self sendMessage];    
 });     
```    
14)CADisplayLink    
The working of CADisplayLink depends on run loop. Once you associate the display link with a run loop,     
the system calls the selector on the target when the screen’s contents need to update.     
The target can read the display link’s timestamp property to retrieve the time the system displayed the previous frame.    
CADisplayLink is an observer of Screen Update events.     
Example:     
```
self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(handleDisplayLink:)];     
[self.displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];     
```
Here is an example utilizing CADisplayLink to calculate the frame rate.    
```    
[[CADisplayLink displayLinkWithTarget:self selector:@selector(displayLinkAction:)]  addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];    
    
- (void)displayLinkAction:(CADisplayLink *)link {    
    
    static NSTimeInterval lastTime = 0;    
    static int frameCount = 0;        
        
    if (lastTime == 0) { lastTime = link.timestamp; return; }    
        
    frameCount++;     
        
    NSTimeInterval passTime = link.timestamp - lastTime;    
        
    if (passTime > 1) {     
        
        int = frameCount / passTime; // frame rate = frame number / time    
            
        lastTime = link.timestamp; // reset    
            
        frameCount = 0; // reset    
            
        NSLog(@"%d", fps);    
    
}    
```
Btw, there is difference between frame rate and refresh rate.    
frame rate(FPS): A frame is a single still image. Frame rate is how many of these images are output in one second.    
refresh rate(Hz):the number of times per second your monitor can redraw the screen    
Frame rate will be limitted by refresh rate. For example, if refresh rate is only half of the frame rate, then      
every two images output by the GPU, only one of them is shown on the screen.    
15)UI refresh in the run loop     
When we call **setNeedsLayout** or **setNeedsDisplay** method of UIView / CALayer, the corresponding layer will be marked as dirty.          
Like the handling of gesture recognizers, a observer has been registered to the run loop which monitors **kCFRunLoopBeforeWaiting** and    
**kCFRunLoopExit** event. The handler of the observer will dispose all the dirty CALayer calling **layer.delegate drawLayer:inContext:**     
or **CALayer drawInContext:** methos. And the CALayer will commit the bit map to Backing Store. And finally GPU will receive and render the image.      
In this way, the UI refreshes.    
16)NSRunloop Usage Example    
16.1)Create long-lived threads    
Usually, a sub thread we create will be detroyed when tasks adding to it has completed. Here, we can create a long-lived thread     
by creating a run loop in the sub thread and adding source to prevent the run loop from exiting.      
```    
//Adds a port as an input source to the specified mode of the run loop.    
- (void)addPort:(NSPort *)aPort forMode:(NSRunLoopMode)mode;    
```     
```    
- (void)viewDidLoad {    
      [super viewDidLoad];    
      self.thread = [[NSThread alloc] initWithTarget:self selector:@selector(run) object:nil];    
      [self.thread start];    
  }    
    
- (void)run {    
      NSRunLoop *currentRl = [NSRunLoop currentRunLoop];    
          
      [currentRl addPort:[NSPort port] forMode:NSDefaultRunLoopMode];    
      [currentRl run];    
  }    
    
- (void)run2    
  {    
        NSLog(@"Long-live thread");    
  }    
    
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event    
  {    
    [self performSelector:@selector(run2) onThread:self.thread withObject:nil waitUntilDone:NO];    
  }    
```    
17)Asynchronous Drawing    
Asynchronous drawing is generating image in sub thread and return the generated image to the main thread which helps to     
decrease the work load of the main thread.    
![avatar](https://ririripley.github.io/assets/img/AsynchronousDrawing.png)    
Example:     
```
// Asynchronous Drawing     
// Generate CGImage in the sub thread     
dispatch_async(backgroundQueue, ^{    
    CGContextRef ctx = CGBitmapContextCreate(...);    
        // draw in context...    
        CGImageRef img = CGBitmapContextCreateImage(ctx);    
        CFRelease(ctx);    
        dispatch_async(mainQueue, ^{    
            // set the contents of layer in the main thread     
            layer.contents = img;    
    });    
});    
    

// Synchronous Drawing     
CGContextRef ctx = CGBitmapContextCreate(...);    
// draw in context...    
CGImageRef img = CGBitmapContextCreateImage(ctx);    
CFRelease(ctx);    
layer.contents = img;    
```    
18)Hangs Detection    
18.1)What are hangs?    
```
When an iOS app is unresponsive, users encounter an app freeze or app hang. No action can be taken as the user often stares at,     
or taps frantically on, a frozen screen.    
```
18.2)What may cause hangs?    
```
1) Time-consuming task is executed in the main thread, i.e., use UIGraphicsGetCurrentContext API to compute drawing in CPU.      
2) The main thread is blocked by or waiting for task in sub threads, i.e., calling dispatch_sync(dispatch_get_main_queue...).     
3) The main thread is waiting for system resouce, i.e., Data(contentsOf:) for IO.     
```
18.3)A Simple Hangs Detection Example    
```    
- (void)start{    
    [self registerObserver];    
    [self startMonitor];    
  }    
    
- (void)registerObserver{    
       // register a observer of the main run loop which monitors all the status of the run loop    
       // CallBack method is the call back of the observer       
      CFRunLoopObserverContext context = {0,(__bridge void*)self,NULL,NULL};    
      //NSIntegerMax : the lowest priority    
      CFRunLoopObserverRef observer = CFRunLoopObserverCreate(kCFAllocatorDefault,    
          kCFRunLoopAllActivities,    
          YES,    
          NSIntegerMax,    
          &CallBack,    
          &context);    
      CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);    
  }    
    
// Call back of the registered observer:     
static void CallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info)    
{    
    DSBlockMonitor *monitor = (__bridge DSBlockMonitor *)info;    
    monitor->activity = activity;    
        
    dispatch_semaphore_t semaphore = monitor->_semaphore;    
    // signal the semaphore    
    dispatch_semaphore_signal(semaphore);    
}    

- (void)startMonitor{    
  // Create a semaphore initialzed as 0    
  _semaphore = dispatch_semaphore_create(0);    
      
  dispatch_async(dispatch_get_global_queue(0, 0), ^{    
      while (YES) {    
          // the time limit is 1 second, if not signaled within the time limit, st != 0, then executing the self->timeout ++ logic    
          long st = dispatch_semaphore_wait(self->_semaphore, dispatch_time(DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC));    
          if (st != 0)    
          {    
            if (self->activity == kCFRunLoopBeforeSources || self->activity == kCFRunLoopAfterWaiting)    
              {    
                // self->_timeoutCount +1    
                if (++self->_timeoutCount < 2){    
                    NSLog(@"timeoutCount==%lu",(unsigned long)self->_timeoutCount);    
                    continue;    
                }    
                // self->_timeoutCount >=2 --> means hang exists      
                NSLog(@"Hang Detected !");    
              }    
          }    
        // reset self->_timeoutCount and do next hangs detection     
        self->_timeoutCount = 0;    
    }    
  });    
}    
```
### **Reference**    
https://juejin.cn/post/6844903540607942663    
https://imlifengfeng.github.io/article/487/    
https://toutiao.io/posts/119854/app_preview    
https://zhuanlan.zhihu.com/p/62605958    
https://github.com/liberalisman/iOS-InterviewQuestion-collection/blob/master/Runloop/7.%E7%AC%AC%E4%B8%83%E9%A2%98.md    
https://github.com/tbfungeek/iOS-Interviews/issues/156    
https://blog.csdn.net/u011774517/article/details/69396804    
https://blog.chenyalun.com/2018/09/30/PerformSelector%E5%8E%9F%E7%90%86/    
https://www.jianshu.com/p/466f709b8147    
https://www.avadirect.com/blog/frame-rate-fps-vs-hz-refresh-rate/    
https://juejin.cn/post/6844904110991360008#heading-20    
https://www.jianshu.com/p/ee62bbf38559    
https://juejin.cn/post/6844904110991360008    
