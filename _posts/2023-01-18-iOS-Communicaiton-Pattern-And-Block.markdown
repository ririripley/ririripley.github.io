---    
author: ripley    
comments: false    
date: 2023-01-18 12:11:08+00:00    
layout: post    
slug: iOSDevelopment    
title: Communication Patterns & Block    
wordpress_id: 304    
categories:    
- Tech    
tags:    
description: Common Communication Patterns And Block    
---    
## **NSNotification**    
1)NSNotificationCenter    
Each running app has a defaultCenter notification center, and you can create new notification centers to      
organize communications in particular contexts.      
![avatar](https://ririripley.github.io/assets/img/NSNotificationCenter.png)    
1.1)Add Observer    
```    
typedef NSString *NSNotificationName;    
    
- (void)addObserver:(id)observer // An object to register as an observer, the observer here is weak reference.    
           selector:(SEL)aSelector  // A selector that specifies the message the receiver sends observer to alert it to the notification posting.    
               name:(NSNotificationName)aName  // The name of the notification to register for delivery to the observer.    
             object:(id)anObject; // The object that sends notifications to the observer.    
// If you add the same KVO for n times, then SEL messages will be sent n times to the observer. Therefore, make sure that     
addObserver and remove observer in the same number of times.                                  
```    
Inside NSNotificationCenter, there are three data structs: Named Table, Nameless Table and Wildcard.    
Named Table    
![avatar](https://ririripley.github.io/assets/img/Named-table.png)    
```    
NamedTable{    
    Key: NSNotificationName    
    Value: Inner Table {    
                    Key: anObject      
                    Value: observer linked list    
                }    
}       
e.g.     
    
- (void)testNotification{    
    [[NSNotificationCenter defaultCenter]addObserver:self selector:@selector(notificationHandler:) name:@"Eezy" object:@"my object"];    
    [[NSNotificationCenter defaultCenter]postNotificationName:@"Eezy" object:@"my object" userInfo:@{@"Status": @"Success"}];    
}    
     
- (void)notificationHandler:(NSNotification *) notification{    
     NSLog(@"%@",notification.object);    
}    
```    
    
Attention, when the parameter object is nil, the system will automatically generate a key and store the observers      
with nil object in the table. The observers in this table will receive all the notifications of the name NSNotificationName.    
For example,    
```    
self.name = @"Tom";    
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(test1) name:@"lala" object:_name];    
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(test2) name:@"lala" object:nil];    
    
[[NSNotificationCenter defaultCenter] postNotificationName:@"lala" object:nil]; // Only test2 is called.    
    
[[NSNotificationCenter defaultCenter] postNotificationName:@"lala" object:_name]; // Both test1 and test2 is called.     
    
_name = @"lala";    
[[NSNotificationCenter defaultCenter] postNotifcationName:"lala" object:_name]; // Since name changed, only test2 is called.     
```    
Nameless Table    
![avatar](https://ririripley.github.io/assets/img/Nameless-table.png)    
```    
When the parameter name is nil, use nameless table.     
NamelessTable{    
    Key: anObject    
    Value: observer linked list    
}    
```    
Wildcard    
```    
A linked list. When the parameter object and name is nil, observers will be added to this linked list.      
All the observers in wildcard can receive all NSNotifications.      
```    
To summarize, the observer will be stored based on the parameters.    
When name is not nil, use named table.    
When name is nil, if object is not nil, use nameless table using object as key, otherwise, use wildcard.    
1.2)Notify Observer    
```    
- (void)postNotificationName:(NSNotificationName)aName // The name of the notification.    
  object:(id)anObject //The object posting the notification.    
  userInfo:(NSDictionary *)aUserInfo; // A user info dictionary with optional information about the notification.    
```    
The process of notifying observers:    
```    
1) Initialize an array which is used to store observers.     
2) Traversal wildcard linked list and add all the observers to the observer array.     
3) If object is not nil, get the observers in namelessTable[object] and add them to the obeserver array.    
4) If name is not nil, get both the observers in namedTable[name][object] and namedTable[name][nilObjectCorrepsondingKey] and add     
all of them to the observer array.    
5) for (int i = 0; i < obeserverArrayLen; i++) {    
    auto observerInfo = obeserverArray[i];    
    [observerInfo->observer performSelector: observerInfo->selector withObject: notification];    
}    
```    
As can be seen for the above process, NSNotification add and notify observer in the same thread and this mechanism works      
in a synchronized way. Then comes the question: how to post notification asynchronously? The answer is use **NSNotificationQueue**.    
2)NSNotificationQueue    
A notification center buffer. A notification queue maintains notifications in first in, first out (FIFO) order.    
When a notification moves to the front of the queue, the queue posts it to the notification center, which in turn dispatches    
the notification to all objects registered as observers.    
Every thread has a default notification queue, which is associated with the default notification center for the process.    
    
NSNotificationQueue works depending on run loop:    
Whereas a notification center distributes notifications when posted,    
notifications placed into the queue can be delayed until the end of the current pass through the run loop or until the run loop is idle.    
    
Duplicate notifications can be coalesced so that only one notification is sent although multiple notifications are posted.    
```    
Adds a notification to the notification queue with a specified posting style, criteria for coalescing, and run loop mode.    
- (void)enqueueNotification:(NSNotification *)notification  // The notification to add to the queue.    
               postingStyle:(NSPostingStyle)postingStyle //The constants that specify when notifications are posted.     
               coalesceMask:(NSNotificationCoalescing)coalesceMask //The constants that specify how notifications are coalesced.     
                   forModes:(NSArray<NSRunLoopMode> *)modes; //The list of modes the notification may be posted in.     
```    
Example:    
```    
#import "NotificationViewController.h"    
#define NOTIFICATIONNAME @"NSNotificationCenter__"    
@interface NotificationViewController ()    
    
@end    
    
@implementation NotificationViewController    
    
- (void)viewDidLoad {    
  [super viewDidLoad];    
  UIButton *button1 = [UIButton buttonWithType:UIButtonTypeSystem];    
  button1.frame = CGRectMake(200, 0, 100, 100);    
  button1.backgroundColor = [UIColor redColor];    
  [button1 addTarget:self action:@selector(buttonClick1) forControlEvents:UIControlEventTouchUpInside];    
  [self.view addSubview:button1];    
    
  [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(addObserver) name:NOTIFICATIONNAME object:nil];    
  }    
    
- (void)addObserver    
  {    
    NSLog(@"接收到消息==%@",[[NSThread currentThread]description]);    
    NSLog(@"睡眠5秒");    
    sleep(5);    
  }    
    
    
- (void)buttonClick1    
  {    
    NSLog(@"开始postNotificationName==%@",[[NSThread currentThread]description]);    
    NSNotification *noti = [NSNotification notificationWithName:NOTIFICATIONNAME object:nil];    
    [[NSNotificationQueue defaultQueue] enqueueNotification:noti postingStyle:NSPostASAP];    
    NSLog(@"结束postNotificationName==%@",[[NSThread currentThread]description]);    
  }    
  @end    
// The printing result is :     
开始postNotificationName==<NSThread: 0x600002b84680>{number = 1, name = main}    
结束postNotificationName==<NSThread: 0x600002b84680>{number = 1, name = main}    
接收到消息==<NSThread: 0x600002b84680>{number = 1, name = main}    
睡眠5秒      
```    
## **Delegate**    
1)Delegate is usually describes with weak to avoid reference cycle.    
2)What is the difference between **Delegate** and **Observer** Mode    
```    
Delegate mode: better for one-to-one reltionship    
Observer mode: better for one-to-many reltionship    
    
Delegate mode has higher couping while observer mode has lower one.     
FYI, coupling refers to how related or dependent two classes/modules are toward each other.    
```    
## **KVO**    
1)addObserver:forKeyPath:options:context:    
// This method does not maintain strong references to the observing object, the observed objects, or the context.    
// Registering an observer multiple times causes receiving notifications multiple times.    
```    
//Registers the observer object to receive KVO notifications for the key path relative to the object receiving this message.    
(void)addObserver:(NSObject *)observer   // The object to register for KVO notifications.    
       forKeyPath:(NSString *)keyPath    // The key path, relative to the array, of the property to observe.     
          options:(NSKeyValueObservingOptions)options // A combination of the NSKeyValueObservingOptions values that specifies what is included in observation notifications.     
          context:(void *)context; // Arbitrary data that is passed to observer in observeValueForKeyPath:ofObject:change:context:.    
```    
2)Example:    
```    
#import <Foundation/Foundation.h>    
@interface Person : NSObject    
@property (nonatomic, assign) int age;    
@end    
    
    
- (void)viewDidLoad {    
  [super viewDidLoad];    
  Person *p1 = [[Person alloc] init];    
  Person *p2 = [[Person alloc] init];    
  p1.age = 1;    
  p1.age = 2;    
  p2.age = 2;    
  // self 监听 p1的 age属性    
  NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;    
    
  [p1 addObserver:self forKeyPath:@"age" options:options context:nil];    
  p1.age = 10;    
  [p1 removeObserver:self forKeyPath:@"age"];    
}    
    
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context    
{    
  NSLog(@"监听到%@的%@改变了%@", object, keyPath,change);    
}    
// The printing result:     
监听到<Person: 0x604000205460>的age改变了{    
    kind = 1;    
    new = 10;    
    old = 2;    
}    
```    
As can be seen, person1 property change is notified.      
3)How does it work?    
When p1 receives the message **addObserver:forKeyPath:options:context:**, what is happening under the hood?    
```    
1)The isa pointer of p1 changes from Person Class object to NSKVONotifyin_Person class object.     
2)The pseudo code of NSKVONotifyin_Person class.     
    
///NSKVONotifying_Person.m     
#import "NSKVONotifying_Person.h"    
    
@implementation NSKVONotifying_Person    
    
- (void)setAge:(int)age{    
  _NSSetIntValueAndNotify();     
}    
    
void _NSSetIntValueAndNotify(){    
  [self willChangeValueForKey:@"age"];    
  [super setAge:age];    
  [self didChangeValueForKey:@"age"];    
}    
    
- (void)didChangeValueForKey:(NSString *)key{    
  [observe observeValueForKeyPath:key ofObject:self change:nil context:nil];    
}     
@end    
Therefore, when excecuting [p1.age = 10], what actually happens is     
objc_send_message(p1->isa, setAge, 10), which further elicit [observeValueForKeyPath:ofObject:change:context:] method.    
```    
In summarize, when call ****addObserver:forKeyPath:options:context:**** method, a corresponding subclass will be generated    
and get the instance object's isa pointer pointed to the generated subclass.  When the property of the instance object changes,      
the observer will be sent **observeValueForKeyPath:ofObject:change:context:** message.    
We can notice that, **willChangeValueForKey:** and **didChangeValueForKey:** can also be manually called. And also,    
if we directly modify the member variables (i.e. _age = 10), KVO does not work since it does not elicit set method.    
4) Pitfalls About KVO    
   4.1) In cases of inheritance, how to decide which class object to handle the change?    
```       
// Solution : Check whether the observing object is the one we want, otherwise, throw it to super class.     
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context{    
  if (object == _myObject && [keyPath isEqualToString:@"contentOffset"]) {    
      [self doSomethingWhenContentOffsetChanges];}    
  else {    
      [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];}    
  }    
```    
4.2) How to avoid repeated observer removal or adding?    
To avoid repeated removal of observers:    
4.2.1)Use try catch    
4.2.2)Use method swizzling    
Check whether observationInfo has the observer or not before adding or removing the observer. Use method swizzling to replace    
the original implementaion of **addObserver** and **removeObserver** method.    
NSObject {    
// Returns a pointer that identifies information about all of the observers that are registered with the observed object.    
@property void *observationInfo;    
}    
```     
// NSObject+SafeKVO.h    
@interface NSObject (SafeKVO)    
- (void)removeAllObserverdKeyPath;    
@end    
    
// NSObject+SafeKVO.m    
#import "NSObject+SafeKVO.h"    
#import <objc/runtime.h>    
@implementation NSObject (SafeKVO)    
     
+ (void)load {    
        
    Method originAddM = class_getInstanceMethod([self class], @selector(addObserver:forKeyPath:options:context:));    
    Method swizzAddM = class_getInstanceMethod([self class], @selector(swizz_addObserver:forKeyPath:options:context:));    
        
    Method originRemoveM = class_getInstanceMethod([self class], @selector(removeObserver:forKeyPath:context:));    
    Method swizzRemoveM = class_getInstanceMethod([self class], @selector(swizz_removeObserver:forKeyPath:context:));    
        
    IMP originAddMIMP = class_getMethodImplementation([self class], @selector(addObserver:forKeyPath:options:context:));    
    IMP originRemoveIMP = class_getMethodImplementation([self class], @selector(removeObserver:name:object:));    
        
    BOOL hasAddM = class_addMethod([self class], @selector(addObserver:forKeyPath:options:context:), originAddMIMP, method_getTypeEncoding(originAddM));    
        
        
    BOOL hasRemoveM = class_addMethod([self class], @selector(removeObserver:forKeyPath:context:), originRemoveIMP, method_getTypeEncoding(originRemoveM));    
        
    // excute once     
    static dispatch_once_t onceToken;    
    dispatch_once(&onceToken, ^{    
        if (!hasAddM) {    
            method_exchangeImplementations(originAddM, swizzAddM);    
        }    
        if (!hasRemoveM) {    
            method_exchangeImplementations(originRemoveM, swizzRemoveM);    
        }    
    });    
}    
     
- (void)swizz_addObserver:(nonnull NSObject *)observer forKeyPath:(nonnull NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(nullable void *)context {    
    if (![self hasKey:keyPath]) {    
        // 调用系统的添加observer 方法    
        [self swizz_addObserver:observer forKeyPath:keyPath options:options context:context];    
    }    
}    
- (void)swizz_removeObserver:(nonnull NSObject *)observer forKeyPath:(nonnull NSString *)keyPath context:(nullable void *)context {    
    if ([self hasKey:keyPath]) {    
        [self swizz_removeObserver:observer forKeyPath:keyPath context:context];    
    }    
}    
     
- (void)removeAllObserverdKeyPath {    
    id info = self.observationInfo;    
    NSArray *arr = [info valueForKeyPath:@"_observances._property._keyPath"];    
    for (NSString *keyPath in arr) {    
        // TODO context 需要考虑值不为nil的时候    
        [self removeObserver:self forKeyPath:keyPath context:nil];    
    }    
}    
     
- (BOOL)hasKey:(NSString *)kvoKey {    
    BOOL hasKey = NO;    
    id info = self.observationInfo;    
    NSArray *arr = [info valueForKeyPath:@"_observances._property._keyPath"];    
    for (id keypath in arr) {    
        // 存在kvo 的key    
        if ([keypath isEqualToString:kvoKey]) {    
            hasKey = YES;    
            break;    
        }    
    }    
    return hasKey;    
}    
     
@end    
```    
5)Registering Dependent Keys    
If the value of one property depends on that of one or more other attributes in another object or the value    
of one attribute changes, then the value of the derived property should also be flagged for change, we can    
ensure that key-value observing notifications are posted by registering dependent keys.    
// Returns a set of key paths for properties whose values affect the value of the specified key.    
+ (NSSet<NSString *> *)keyPathsForValuesAffectingValueForKey:(NSString *)key;    
  Here is an example:    
```    
@interface Person : NSObject    
    
@property(nonatomic, copy) NSString *name;    
@property(nonatomic, assign) NSInteger age;    
    
@end    
    
@interface Student : NSObject    
    
@property(nonatomic, copy) NSString *information;    
@property(nonatomic, strong) Person *person;    
    
@end      
```    
The relationship graph can be described as follows:    
```    
Student.information = f(person.name, person.age)    
```    
The dependent keys are person.name and person.age, and we want to observe the information change of a Student instance object.    
Here is the implementation:    
```    
@implementation Student    
- (instancetype)init {    
  if (self = [super init]) {    
    self.infomation = @"";    
    self.person = [[Person alloc]init];    
    self.person.name = @"";    
    self.person.age = 0;    
  }    
  return self;    
}    
    
- (NSString *)infomation {    
    return [NSString stringWithFormat:@"student_name = %@ | student_age = %ld",self.person.name,self.person.age];    
  }    
    
- (void)setInfomation:(NSString *)infomation {    
    NSArray *array = [infomation componentsSeparatedByString:@"#"];    
    if (array && array.count == 2) {    
    self.person.name = array[0];    
    self.person.age = ((NSNumber *)array[1]).integerValue;    
    }    
  }    
    
+ (NSSet<NSString *> *)keyPathsForValuesAffectingValueForKey:(NSString *)key {    
  if ([key isEqualToString:@"infomation"]) {    
    NSSet *set = [NSSet setWithObjects:@"person.name", @"person.age",nil];    
    return set;    
  }    
  return nil;    
}    
@end    
    
// viewController.m    
- (void)viewDidLoad {    
    [super viewDidLoad];    
    self.student = [[Student alloc]init];    
    [self.student setInfomation:@"Charlie#30"];    
    [self.student addObserver:self forKeyPath:@"infomation" options:NSKeyValueObservingOptionOld|NSKeyValueObservingOptionNew context:nil];    
}    
    
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {    
    if ([keyPath isEqualToString:@"infomation"]) {    
        NSLog(@"OBSERVE : old infomation = %@ | new infomation = %@", change[NSKeyValueChangeOldKey], change[NSKeyValueChangeNewKey]);    
    }    
}    
    
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {    
    self.student.person.name = @"Vivian";    
    self.student.person.age = 40;    
  }    
}     
// Once touch happens, the printing result will be :    
OBSERVE : old infomation = student_name = Charlie | student_age = 30 | new infomation = student_name = Vivian | student_age = 30    
 OBSERVE : old infomation = student_name = Vivian | student_age = 30 | new infomation = student_name = Vivian | student_age = 40        
```    
## **KVC**    
1)NSKeyValueCoding    
A mechanism by which you can access the properties of an object indirectly by name or key.    
2)Get Value    
- valueForKey:    
  Returns the value for the property identified by a given key.    
- valueForKeyPath:    
  Returns the value for the derived property identified by a given key path.    
- valueForUndefinedKey:    
  Invoked by valueForKey: when it finds no property corresponding to a given key.    
  3)Set Value    
```      
1.Try to call the set method corresponding to the key.     
1.1)If the key corresponds to a NSObject pointer, execute assign the value directly.    
1.2)If the key corresponds to primitive type, the value parameter will be converted to the corresponding primitive data type and then assign the value.    
1.3)If the value is nil and the key corresponds to non-NSObject pointer type, setNilValueForKey: method will be called with default implementation throwing     
NSInvalidArgumentException error, which developers can override.     
2.If set method is not implemented, and the corresponding class has +accessInstanceVariablesDirectly method returning YES,      
then go looking for member variable with name _key, _isKey, key, isKey. For example, there is an member variable called _isAge, the execute STEP1.     
3.If both STEP 2 and STEP 3 fail, setValue:forUndefinedKey: will be called which by default throws NSUndefinedKeyException error.      
```        
- setValue:forKey:    
  Sets the property of the receiver specified by a given key to a given value.    
- setValue:forUndefinedKey:    
  Invoked by setValue:forKey: when it finds no property for a given key.    
- setValue:forKeyPath:    
  Sets the value for the property identified by a given key path to a given value.    
- setNilValueForKey:    
  Invoked by setValue:forKey: when it’s given a nil value for a scalar value (such as an int or float).    
4)Example    
```          
@interface Student : NSObject    
@property(nonatomic,copy)NSString *name;    
@property(nonatomic,assign)NSInteger age;    
@end    
    
@interface Student(){    
  NSString *_name;    
  NSInteger _age;    
}    
@end    
    
@implementation Student {    
NSString* myname;    
}    
@synthesize name = _name;    
@synthesize age = _age;    
    
-(void)setName:(NSString *)name{    
  _name = name;    
  NSLog(@"name -- %@",_name);    
}    
- (void)setAge:(NSInteger)age{    
  _age = age;    
  NSLog(@"age -- %ld",(long)age);    
}    
    
- (void)setNilValueForKey:(NSString *)key {    
  if ([key isEqual: @"age"]) {    
    NSLog(@"setNilValueForKey age -- %ld",(long)age);    
    _age = 0;    
  }    
}    
@end    
    
int main(int argc, char * argv[])    
{    
  @autoreleasepool {    
    Student * stu = [Student new];    
    [stu setValue:nil forKey:@"age"];    
    [stu setValue:@"myname" forKey:@"myname"];    
    NSLog(@"age: %ld", stu.age);    
    NSLog(@"myname: %@", [stu valueForKey:@"myname"]);    
  }    
}    
// The printing result is :    
setNilValueForKey age -- 0    
age: 0    
myname: myname     
```    
### **Block**    
1)Variable Capture    
Example:    
```    
int global_i = 1;    
static int static_global_j = 2;    
    
struct __main_block_impl_0 {    
  struct __block_impl impl;    
  struct __main_block_desc_0* Desc;    
  int *static_k;    
  int val;    
      
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_static_k, int _val, int flags=0) : static_k(_static_k), val(_val) {    
        impl.isa = &_NSConcreteStackBlock;    
        impl.Flags = flags;    
        impl.FuncPtr = fp;    
        Desc = desc;    
    }    
};    
    
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {    
    int *static_k = __cself->static_k; // bound by copy    
    int val = __cself->val; // bound by copy    
        
            global_i ++;    
            static_global_j ++;    
            (*static_k) ++;    
            NSLog((NSString *)&__NSConstantStringImpl__var_folders_45_k1d9q7c52vz50wz1683_hk9r0000gn_T_main_6fe658_mi_0,global_i,static_global_j,(*static_k),val);    
}    
    
static struct __main_block_desc_0 {    
  size_t reserved;    
  size_t Block_size;    
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};    
    
    
int main(int argc, const char * argv[]) {    
    
    static int static_k = 3;    
    int val = 4;    
    
    void (*myBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, &static_k, val));    
    
    global_i ++;    
    static_global_j ++;    
    static_k ++;    
    val ++;    
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_45_k1d9q7c52vz50wz1683_hk9r0000gn_T_main_6fe658_mi_1,global_i,static_global_j,static_k,val);    
    
    ((void (*)(__block_impl *))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock);    
    
    return 0;    
}    
```    
As shown in the example, there are 4 types of var:    
global var:  global_i (stored in data section)    
static global var: static_global_j  (stored in data section)    
static local var: static_k (stored in data section)    
local var: val (stored in stack)    
```    
1) static global variable, global variable are stored in data section, which can be accessed any where     
2）static local variable, the block store its address (int *static_k = &static_k)    
3) local variable, the block just copy the value to an internal read-only variable (int val = _val) // shallow copy      
```    
2)Block Type    
_NSConcreteStackBlock    
```    
Block which only uses local var, no strong reference (once stack is pop, life ends)    
```    
_NSConcreteMallocBlock    
```    
Block which uses strong reference  (livelihood :Depend on developers, once released, life ends)    
```    
_NSConcreteGlobalBlock    
```    
Block which only uses global or static var, no strong reference, (livelihood :from Created to Termination of the program)    
```    
3)Variable Storage in Block    
3.1)Under ARC environment, in most of the cases, we don't need to manually copy the block from stack to heap. The compiler automatically detects and does that for us.    
So, in what situation can't the compiler detect it?    
```    
The answer is when a Block is passed as an argument for methods or functions.     
But if the method or the function copies the argument inside, the caller doesn’t need to copy it manually in following cases:      
1)Cocoa Framework methods, the name of which includes “usingBlock”    
2)Grand Central Dispatch API    
```    
3.2)Non-id Type Var Storage    
```    
1)Under MRC environment, unless we manually call **copy**, the block will stay in stack, and the __block variable it    
capture will also have __forwarding referring to itself in the stack.    
2)Under ARC environment, when the __block variable is copied from the stack to the heap, a _Block_object_assign function is called     
so that the Block takes ownership of the __block variable. When the __block variable on the heap is disposed of,     
the _Block_object_dispose function is called to release the object in the __block variable.    
```    
3.2)Id Type Var Storage    
```    
_Block_object_assign    
```    
The function assigns the object to the member variable and calls a function, which is equivalent to the retain method.    
```    
_Block_object_dispose    
```    
The function calls a function, which is equivalent to the instance method “release”, on the object in the target member variable of the struct.    
```    
```    
3.2.1)Under ARC environment, when an automatic variable of id or object type is captured for a Block and the Block is copied from the stack to the heap,    
the _Block_object_assign function is called so that the Block takes ownership of the captured object. It is just like an object,    
which is assigned to the object type automatic variable with a __strong qualifier, is used inside a Block.    
When the block on the heap is disposed of, the _Block_object_dispose function is called to release the captured object.    
When an automatic variable of id or object type with a __strong qualifier has a __block specifier, the same thing happens.    
```    
struct __Block_byref_obj_0 {     
  void *__isa;    
  __Block_byref_obj_0 *__forwarding;    
  int __flags;    
  int __size;    
  void (*__Block_byref_id_object_copy)(void*, void*);    
  void (*__Block_byref_id_object_dispose)(void*);     
  __strong id obj;    
};    
    
__block id obj = [[NSObject alloc] init];    
 IS EQAUL TO FOLLOWING SOURCE CODE:    
/* __block variable declaration */    
__Block_byref_obj_0 obj = {     
    0,    
    &obj,    
    0x2000000,    
    sizeof(__Block_byref_obj_0),    
    __Block_byref_id_object_copy_131,     
    __Block_byref_id_object_dispose_131,    
    [[NSObject alloc] init] // The object, which is assigned to the __block variable, exists as well and ownership of it is managed properly.    
};    
```    
3.2.2)Under MRC environment,  the Block take ownership of the captured object, instead, it just copies the value of the id.    
4)dispatch_block_t    
//The prototype of blocks submitted to dispatch queues, which take no arguments and have no return value.    
typedef void (^dispatch_block_t)(void);    
### **Reference**    
https://eezytutorials.com/ios/nsnotificationcenter-by-example.php#.Y8TYouxByrM    
https://www.jianshu.com/p/4a44b9a15fe9    
https://imlifengfeng.github.io/article/509/    
https://www.jianshu.com/p/733c7c25ac3e    
https://juejin.cn/post/6844903593925935117    
https://juejin.cn/post/6844903747680731150    
https://www.jianshu.com/p/ffe2455e34b4    
https://www.jianshu.com/p/ffe2455e34b4    
https://www.jianshu.com/p/6ae89c3d4b1c      
https://blog.csdn.net/weixin_34004750/article/details/91633037    
https://www.cnblogs.com/funny11/p/10063278.html    
http://zxfcumtcs.github.io/2017/05/07/Multithreading-Traps/    
https://juejin.cn/post/6844903759445753864    
https://halfrost.com/ios_block/    
