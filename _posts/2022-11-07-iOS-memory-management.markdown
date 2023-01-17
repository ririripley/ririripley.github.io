---
author: ripley
comments: false
date: 2022-11-07 12:11:08+00:00
layout: post
slug: iOSDevelopment
title: iOS memory management part
wordpress_id: 304
categories:
- Tech
tags:
description: iOS memory management
---
## **iOS - Memory Management**
What are the rules to follow under an ARC-enabled environment?
```
1) Forget about using retain, release, retainCount, and autorelease.    
2) Follow the naming rule for methods related to object creation.      
3) Forget about calling dealloc explicitly.  
4) Forget about using Zone (NSZone).  
5) ‘id’ and ‘void*’ have to be cast explicitly.  
  
```
When the object is used through a variable qualified with __weak, will it be added to an autorelease pool?
```
The answer is YES. 
@autoreleasepool {
    id __strong obj = [[NSObject alloc] init];  
    _objc_autoreleasePoolPrint();  
    id __weak o = obj;  
    NSLog(@"before using __weak: retain count = %d", _objc_rootRetainCount(obj));   
    NSLog(@"class = %@", [o class]);    
    NSLog(@"after using __weak: retain count = %d", _objc_rootRetainCount(obj)); 
    _objc_autoreleasePoolPrint();      
}  
The result is :  
before using __weak: retain count = 1  
after using __weak: retain count = 2  
Here is the explanation:    
1) When we send messages to __weak var, the following operations will be executed.  
    id tmp = objc_loadWeakRetained(&obj); // obj reference the object   
    objc_autorelease(tmp);  
2) objc_loadWeakRetained(id *location) method will call obj->rootTryRetain() which will retain the object refernced   
by obj. The returned tmp references the same object as obj, objc_autorelease method will add tmp to the nearest autorelease pool.  
3) Then the __weak var can receive the message we sent.  
4) Finaly, the autorelease pool will drain which meaning all the objects referenced will release.  
5) Therefore, if we sent a lot of messages to __weak var, the same methods objc_loadWeakRetained and   
objc_autorelease will be called many times leading to repeated adding object to autorelease pool. This will   
be a great cost.   
Here is an intereting discovery:  
    id __strong obj = [[NSObject alloc] init];      
    id __weak o = obj;    
    _objc_autoreleasePoolPrint();  
    NSLog(@"before: retain count = %d", _objc_rootRetainCount(obj));  
    NSLog(@"1 %@", o);  
    NSLog(@"1: retain count = %d", _objc_rootRetainCount(obj));  
    NSLog(@"2 %@", o);  
    NSLog(@"2: retain count = %d", _objc_rootRetainCount(obj));  
    NSLog(@"3 %@", o);  
    NSLog(@"3: retain count = %d", _objc_rootRetainCount(obj));  
    NSLog(@"4 %@", o);  
    NSLog(@"4: retain count = %d", _objc_rootRetainCount(obj));  
    _objc_autoreleasePoolPrint();  
}
The result is :  
before : retain count = 1    
1: retain count = 1  
2: retain count = 1  
3: retain count = 1  
4: retain count = 1    
The retain count does not increase as we thought. This is because in the implementation of NSLog, it contains its own  
autorelease pool. The obj is added to the pool and the pool drains when the execution finishes. Therefore, it does not  
alter the retain count as we see.  
```
What is the difference between __weak and _Unsafe_Unretain？
```
(1) When the object referenced by __weak is dealloc, __weak var will be set as nil while _Unsafe_Unretain won't in which   
case __Unsafe_Unretain var will be a dangerous dangling pointer.  
(2) Hash table is used to guarantee __weak var to set to be nil, which also means that _Unsafe_Unretain is more efficient to use.  
``` 
How is __weak var set as nil when the object it references dealloc?  
![avatar](https://ririripley.github.io/assets/img/how__weak_set_as_nil.png)
What is the difference between retain, copy, assign, weak, and _Unsafe_Unretain ?
``` 
These keyword is related to the ownership qualifier.  
1) retain ->  strong  
2) copy ->  strong  (also depending on How its NSCopying protocol is implemented)  
3) assign will generate a setter which assigns the value to the instance variable directly,   
rather than copying or retaining it. This is best for primitive types like NSInteger and CGFloat,   
or objects you don't directly own, such as delegates.  
4) Unsafe_Unretain -> weak, but it won’t be set to nil if the destination object is deallocated.
```
How does ARC work in compilation and runtime?      
```
In compiling time: ARC inserts release and retain based on the context.  
In runtime: ARC will execute some extra methods like set __weak var to nil when the objects  
it referenced has been deallocated. Another example is whether to add an reference to autorelease pool  
based on the context(more specifically, obj_retatinAutoreleaseReturnValue and obj_AutoreleaseReturnValue).      
```
What is the difference between ARC and GC(garbage collection)?
```
1) Retain Cyle:   
GC can clean up entire object graphs, including retain cycles. ARC cannot handle retain cycles automatically.    
GC proceed in the background, so less memory management work is done as part of the regular application flow.    
2) Operation:  
GC works in the runtime as it will detect the unused object graphs (will eliminate retain-cycles) and     
remove them on an indeterminate time intervals. ARC injects a code into the executable to be executed "automatically" on  
unused objects depending on their reference count.  
3) Release Time:   
ARC lead to real-time, deterministic destruction of objects as they become unused. Because GC happens in the background,    
the exact time frame for object releases is undetermined.    
4) Performance:   
GC proceed in the background while ARC involves no background processing. When a GC happens,  
other threads in the application may be temporarily put on hold.  
```
When will the object automatically register to the autoreleasepool?  
```
When an object is returned from a method, the compiler checks if the method begins with alloc/new/copy/mutableCopy,   
and if not, the returned object is automatically registered to the autorelease pool.  
Exceptionally, any method whose name begins with init, doesn’t register the return value to autoreleasepool.  
However, if the object receives the message objc_retainAutoreleaseValue, it will not be autoreleased.
i.e. 
+ (id) array {
        return [[NSMutableArray alloc] init];
}

Equal to 

/* pseudo code by the compiler */
{  
    id __strong obj = [NSMutableArray array];  		
}  
+ (id) array  
{  
        id obj = objc_msgSend(NSMutableArray, @selector(alloc));  
        objc_msgSend(obj, @selector(init));  
        return objc_autoreleaseReturnValue(obj);  
}    
  
The pseudo code by the compiler is as follows:  
{   
    id obj = objc_msgSend(NSMutableArray, @selector(array));    
    objc_retainAutoreleasedReturnValue(obj);    
    objc_release(obj);    
}      
+ (id) array    
{  
        id obj = objc_msgSend(NSMutableArray, @selector(alloc));    
        objc_msgSend(obj, @selector(init));  
        return objc_autoreleaseReturnValue(obj);    
}  
In this case, since obj = objc_autoreleaseReturnValue(obj), and then it is called retainAutoreleasedReturnValue,  
therefore obj will not execute autoRelease method for obj, and obj accepts the hand off of retain count.   
```
What is the difference between wild pointer and dangling pointer?    
```
Dangling pointer:  
The memory space it points to has been recycled.
Wild pointer:   
A pointer which is not initialzed and may point to random address.     
```
What is the default @property qualifier in Objective-c?    
```
@property (atomic,readWrite,strong) UIView *view;  
@property (assign) NSInteger number;
```
What is the universal memory layout? (i.e. For C/C++ program)
```
stack  
heap   
code: store assembly code (code segment includes a rodata section which stores readOnly data   
like string and const var)  
data segment: store already initialized global and static variables  
bss(Block Started by Symbol) segment: store uninitialized global and static variables    
```
How does Dealloc work?  
```
dealloc -> _objc_rootDealloc -> object_dispose -> objc_destructInstance  
void *objc_destructInstance(id obj) 
{  
    if (obj) {  
        // Read all of the flags at once for performance.  
        bool cxx = obj->hasCxxDtor();  
        bool assoc = obj->hasAssociatedObjects();  

        // This order is important.  
        if (cxx) object_cxxDestruct(obj);  
        if (assoc) _object_remove_assocations(obj);  
        obj->clearDeallocating();  
    }  
    return obj;  
}  
1)object_cxxDestruct:   
release the member variables and call super class dealloc  :  
for ( ; cls; cls = cls->superclass) {    
    traverse all the member variables and send objc_storeStrong(&memberVarId, null) message to them.  
    [cls dealloc];    
}      
2)remove_assocations    
release the objects which are dynamic bound with the object;    
3)clearDeallocating:   
sideTable_clearDellocating()。  
weak_clear_no_lock: clear the __weak hash table, in other words, set all __weak id as nil;         
table.refcnts.erase();: clear the reference count of the object from the reference table;   
```
Explicit getters/setters for @properties (MRC)   
```
1)@property(retain) NSString *value  
-(void) setValue:(NSString*)aValue {    
    @synchronized{  
          [aValue ratain];  
          [_value release];  
          _value = aValue;  
    }    
}  
-(void) value {    
    @synchronized{    
          retain [[_value retain] auteRelease]  
    }     
}   
If the property is readOnly, no setter method will be generated.  
2)@property(copy) NSString *value
-(void) setValue:(NSString*)aValue {      
    @synchronized{    
          [_value release];    
          _value = [aValue copy];  
    }    
}   
-(void) value {      
    @synchronized{      
          retain _value;    
    }     
}     
3)@property (assign) NSString *value  
-(void) setValue:(NSString*)aValue {      
    @synchronized{    
          aValue = value;  
    }    
}   
-(void) value {      
    @synchronized{      
          retain _value;    
    }     
}     
```
What is the mechanism of retain and release?  
```
The psedu algorithm :  
hashValue = Hash(id)  
retain: sizeTable(hashValue)++    
release: sizeTable(hashValue)--  
```
What will the autoReleasePool drain?  
```
1)In the run loop of the main thread of UIApplication, two observers are registered.   
2) The first observer will elicit _wrapRunLoopWithAutoreleasePoolHandler() when noticing the upcoming      
entering the loop. This observer has the highest priority which makes sure its handler is executed first.      
3) The second observer monitors the event of BeforeWaiting(sleep) and Exit(exit the loop).   
When noticing BeforeWaiting, it executes _objc_autoreleasePoolPop to pop the old pool and _objc_autoreleasePoolPush to push a new pool.       
When noticing Exit, it executes _objc_autoreleasePoolPop() to pop the pool.    
This observer has the lowest priority which makes sure that its handlers are executed after all other handlers.        
```
What is the difference between @synthesize and @dynamic for properties?  
```
1)  @synthesize will generate setter and getter method for you in compile time.  Usually, when we define @property, the   
system will automatically help us add corresponding @synthesize.    
2)  @dynamic is usually used to avoid warnings about methods missing at compile time. It tells the compiler that the setter and getter method    
is defined somewhere else(Some accessors are created dynamically at runtime or defined by super class).  
```
What will cause BAD_ACCESS?  
```
Visit the memory space which has been recycled. (e.g. using the dangling pointer).    
```
How does retain count works in ARC?  
```
Each Side Table contains following tables:  
weak_size_table : stores the weak id referencing the object 
size_table : stores the reference count
retainCount = size_table[hashValue[id]]    
```
What is the data structure under the AutoReleasePool?
```
AutoReleasePoolPage:  (C++ object: a Doubly-linkes list)  
```
![avatar](https://ririripley.github.io/assets/img/autoReleasepoolPage.png)
```
@autoReleasePool{  
        // do whatever you want  
}  
is actually :  
void * atautoreleasepoolobj = objc_autoreleasePoolPush();    
        // do whatever you want    
objc_autoreleasePoolPop(atautoreleasepoolobj);  

the NEXT pointer points to the next id to be added to autoreleasePool.
If the AutoReleasePoolPage C++ struct is running out of memory, a new AutoReleasePoolPage struct  
will be created whose parent points to the old page. The old page's child pointer points to the new page.  
When objc_autoreleasePoolPush is excuted, atautoreleasepoolobj is added to the top of the stack of the page struct.    
When objc_autoreleasePoolPush is pop, all the id on top of atautoreleasepoolobj will be sent release method, and NEXT pointer  
will point to the top of the last pool.    
```

### **Reference**
https://choujiji.github.io/2019/08/20/%E4%BD%BF%E7%94%A8__weak%E5%8F%98%E9%87%8F%EF%BC%8C%E6%8C%87%E5%90%91%E7%9A%84%E5%AF%B9%E8%B1%A1%E5%B0%B1%E4%BC%9A%E8%A2%AB%E5%8A%A0%E5%88%B0autoreleasepool%E4%B8%AD%EF%BC%9F/  
https://juejin.cn/post/6844903703208525838   
https://stackoverflow.com/questions/21802069/explicit-getters-setters-for-properties-mrc   
https://stackoverflow.com/questions/8382523/setter-and-getter-for-an-atomic-property?lq=1  
https://www.jianshu.com/p/0afda1f23782  
https://juejin.cn/post/7032676024412274718   

