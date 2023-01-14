---
author: ripley
comments: false
date: 2022-12-29 12:11:08+00:00
layout: post
slug: iOSDevelopment
title: Objective-C Runtime Mechanism
wordpress_id: 304
categories:
- Tech
tags:
description: Objective-C Runtime Mechanism
---
## **iOS - Objective-C Runtime Mechanism**
1) Data Structure Of Objective-C instance and class object          
   ![avatar](https://ririripley.github.io/assets/img/objective-c-object-model.png)  
1.1) objc_object      
``` 
struct objc_object {      
    Class isa  OBJC_ISA_AVAILABILITY;      
};    
```
1.2) Class      
``` 
typedef struct objc_class *Class;            
```
1.3)objc_class     
```
struct objc_class {      
  
// points to MetaClass which stores Class method and properties      
Class _Nonnull isa  OBJC_ISA_AVAILABILITY;       
  
#if !__OBJC2__  
    //pointer to Super Class  
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;    
    //Class name  
    const char * _Nonnull name                               OBJC2_UNAVAILABLE;  
    
    long version                                             OBJC2_UNAVAILABLE;  
    long info                                                OBJC2_UNAVAILABLE;  
    // size of instance object   
    long instance_size                                       OBJC2_UNAVAILABLE;  

    // member variables info(name and type of the variables but not their values(values are stored in objc_object struct))          
    struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;  

    // method list (pointet to pointer)  
    struct objc_method_list * _Nullable * _Nullable methodLists   OBJC2_UNAVAILABLE;  

    // method cache list (methods called will be cached for quick calling next time)    
    struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;  
    
    struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;    
#endif  
} OBJC2_UNAVAILABLE;    
```  
1.4)objc_ivar_list      
```
struct objc_ivar_list {  
// number of variables     
int ivar_count                                   OBJC2_UNAVAILABLE;    
#ifdef __LP64__  
    // space required for this objc_ivar_list    
    int space                                     OBJC2_UNAVAILABLE;  
#endif  
    /* variable length structure */  
    struct objc_ivar ivar_list[1]                            OBJC2_UNAVAILABLE;  
}  
  
What is variable length structure:     
Flexible Array Member used in C++ struct.    
e.g. 
typedef struct mytest{  
	int a;  
		
	//c does not takes space, it represents an address and has to be the last member of the struct.    
	char c[];    
}mt;      
Usage:  
constchar* tmp_buf = "abcdefghijklmnopqrstuvwxyz";    
intntmp_buf_size = strlen(tmp_buf);   
mt* pmt = (mt*)malloc(sizeof(mt) + intntmp_buf_size + 1);      
memset(pmt,0, sizeof(mt) + intntmp_buf_size + 1);  
memcpy((char*)(pmt->c),tmp_buf, ntmp_buf_size);   
In this way, the whole struct occupies a continutous memory space.   
'free(pmt)' is enough to release all the memory.    
```
1.5) objc_ivar  
```
struct objc_ivar {  
// name of the variable  
char * _Nullable ivar_name                               OBJC2_UNAVAILABLE;  
// type of the variable  
char * _Nullable ivar_type                               OBJC2_UNAVAILABLE;    
// the offset of the stored variable  
int ivar_offset                                          OBJC2_UNAVAILABLE;  
#ifdef __LP64__  
    // space needed to store this variable  
    int space                                             OBJC2_UNAVAILABLE;  
#endif  
}  
```
1.6) id    
``` 
typedef struct objc_object *id;             
```
In sum, the value of  member variables in an instance is all store in the instance object(objc_object).    
The instance methods are all stored in **methodLists** of the Class object(pointed by isa in the instance object struct).     
The instance member variables info is all stored in **ivars** of the class object.  
The class methods are all stored in **classMethodLists** of the Meta Class object(pointer by isa in the class object struct).
2) How much memory does an NSObject takes?
```      
class_getInstanceSize(obj)    
// get the size of all the member variables of the object       
```
```      
malloc_size((__bridge const void *)(obj))  
// get the size of the object pointer by obj       
```   
Therefore, **class_getInstanceSize** can be taken as the actual memory space an object requires while malloc_size  
is the space allocated by the system part of which is free.   
Based on the above info, we can infer that system must has done something special to allocate memory space  
for an object. Let's try to find out what happens under the hood.   
``` 
This is the method stack of alloc method:  
alloc -> _objc_rootAlloc -> callAlloc -> class_createInstance -> _class_createInstanceFromZone  
Inside _class_createInstanceFromZone method, memory align is applied:  
The memory bytes allocated should be times of 16 bytes and at least 16 bytes.  
``` 
Also, we should know that there is bytes align in a struct:   
Each variable will take the same memory space consistent with the variable which occupies the largest memory space.  

Combined what is mentioned above(memory align and struct bytes align), we can analyze **NSObject** example now:  
![avatar](https://ririripley.github.io/assets/img/NSObject_memory_space.png)  
**Analysis**  
```
Since isa takes 8 bytes, combined with memory aligning, an NSObect is allocated 16 bytes. But it only takes 8  
bytes actually, the rest 8 bytes are all 0.  
``` 
One more example:
``` 
//interface  
@interface Animal: NSObject{  
int weight;  
long num;
}  
@end  
  
int main(int argc, const char * argv[]) {  
@autoreleasepool {  
Animal *animal = [[Animal alloc] init];  
NSLog(@"animal对象实际需要的内存大小: %zd", class_getInstanceSize([animal class]));  
NSLog(@"animal对象实际分配的内存大小: %zd", malloc_size((__bridge const void *)(animal)));  
}  
return 0;  
}  
//Printint Result:   
animal对象实际利用的内存大小: 24
animal对象实际占用的内存大小: 32
```  
**Analysis**  
```  
Int takes 4 bytes, isa takes 8 bytes, long takes 8 bytes, due to bytes aligning in struct, an Animal instance object     
takes 8 + 8 + 8 = 24 bytes, and the system allocates 32 bytes(times of 16 bytes).     
```  
3)What is class_rw_t and class_ro_t?  
```      
struct objc_class : objc_object {    
    Class superclass;   
    cache_t cache;             // formerly cache pointer and vtable    
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags    
  
    class_rw_t *data() {   
        return bits.data();     
    }  
    void setData(class_rw_t *newData) {     
        bits.setData(newData);  
    }  
}     
class_rw_t* data() {  
    return (class_rw_t *)(bits & FAST_DATA_MASK);     
}  
```
As can be seen, **data** method returns the class_rw_t obj.     
**struct class_rw_t**  
```
struct class_rw_t {  
// Be warned that Symbolication knows the layout of this structure.     
uint32_t flags;  
uint32_t version;  
    
    const class_ro_t *ro;      
    
    method_array_t methods; // 方法列表      
    property_array_t properties; // 属性列表    
    protocol_array_t protocols; // 协议列表  
  
    Class firstSubclass;  
    Class nextSiblingClass;  
  
    char *demangledName;  
};  
```
**cache_t**
```
cache_t is used to cache previously called methods.  
1)   
struct cache_t {    
    struct bucket_t *_buckets; // bucket_t array      
    mask_t _mask; // the length of the bucket     
    mask_t _occupied; // the number of cached methods            
};   
2)   
struct bucket_t {   
private:   
    cache_key_t _key; // SEL is the Key      
    IMP _imp; // function pointer         
};       
```
This is how it works:  
![avatar](https://ririripley.github.io/assets/img/cache_t.png)  
```
index= Hash(SEL) ----> bucket_t* bucket = buckets[index]   
----> bucket._key = SEL  
----> bucket._imp = IMP    
```
**class method_array_t**: list_array_tt<method_t, method_list_t>  ->  2D array (list of list)         
**class property_array_t**: list_array_tt<property_t, property_list_t>  ->  2D array (list of list)         
**class protocol_array_t**: list_array_tt<protocol_ref_t, protocol_list_t>  ->  2D array (list of list)           
Take method_array_t as example:  
![avatar](https://ririripley.github.io/assets/img/method_list_t.png)      
**class method_t**  
```
struct method_t {  
    SEL name;  // 函数名  
    const char *types;  // 编码（返回值类型，参数类型）  
    IMP imp; // 指向函数的指针（函数地址）  
};    
```
**SEL**  
```
We can take SEL as char* under the hood.  
Usage:    
We can get SEL through the following 2 methods:  
1) SEL sel1 = @selector(test);     
2) SEL sel2 = sel_registerName("test");    
On the other hand, we can also convert SEL to String through the following 2 methods:  
1) char *string = sel_getName(sel1);  
2) NSString *string2 = NSStringFromSelector(sel2);    
```
Since SEL is identified by the String it represents, methods of the same name in different class has the same SEL which is     
shown as follows:  
```
NSLog(@"%p,%p", sel1,sel2);  
Runtime-test[23738:8888825] 0x1017718a3,0x1017718a3    
// As you can see, the two SEL are the same one stored in address 0x1017718a3.        
```
**types**  
```
You can take types as the encoded string result representing the return value as well as the parameter of the method.    
```
**IMP**  
```
IMP is an address which stores the implementation of the method.(You can take it as the function ptr).   
For example,    
Printing description of data->methods->first.imp:  
(IMP) imp = 0x000000010c66a4a0 (Runtime-test`-[Person testWithAge:Height:] at Person.m:13)  
```
**struct class_ro_t**   
```
struct class_ro_t {    
uint32_t flags;  
uint32_t instanceStart;  
uint32_t instanceSize;  
#ifdef __LP64__  
    uint32_t reserved;  
#endif  

    const uint8_t * ivarLayout;     
      
    const char * name;  
    method_list_t * baseMethodList;     
    protocol_list_t * baseProtocols;  
    const ivar_list_t * ivars;  

    const uint8_t * weakIvarLayout;  
    property_list_t *baseProperties;  
  
    method_list_t *baseMethods() const {     
        return baseMethodList;  
    }  
};  
```
struct **class_ro_t** is read-only.    
4) What is the relationship between the class_rw_t obj and class_ro_t obj inside a Class object?      
In the beginning, the property, member variables info, method_list and so on are all stored in ro(returned by data method),      
when there is category for the class, **realizeClass** method will be executed to copy these info into rw.  
```
static Class realizeClass(Class cls)      
{  
runtimeLock.assertWriting();    

    const class_ro_t *ro;  // read-only   
    class_rw_t *rw;  
    Class supercls;  
    Class metacls;  
    bool isMeta;  

    if (!cls) return nil;    
    if (cls->isRealized()) return cls;    
    assert(cls == remapClass(cls));    

    // In the very beginnning, cls->data refers to ro        
    ro = (const class_ro_t *)cls->data();    

    if (ro->flags & RO_FUTURE) {   
        // If rw is already initialzied and allocated space  
        rw = cls->data();    
        ro = cls->data()->ro;    
        cls->changeInfo(RW_REALIZED|RW_REALIZING, RW_FUTURE);    
    } else {   
        // If rw is not initialzied yet    
        rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1); // Allocate space for rw      
        rw->ro = ro;  // rw->ro refers to the original ro  
        rw->flags = RW_REALIZED|RW_REALIZING;  
        // data method should return rw    
        cls->setData(rw);   
    }  
}  
```
5) How does Category work?  
```
   struct category_t {  
   const char *name;  
   classref_t cls;  
   struct method_list_t *instanceMethods;   // list of method_t          
   struct method_list_t *classMethods;   // list of method_t   
             
   struct protocol_list_t *protocols;   // list of protocol_t  
              
   struct property_list_t *instanceProperties; // properties  
     
   // Fields below this point are not always present on disk.      
   struct property_list_t *_classProperties;     
  
   method_list_t *methodsForMeta(bool isMeta) {    
   if (isMeta) return classMethods;  
   else return instanceMethods;  
   }  
  
   property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);    
   };  
```
**instanceMethods example**    
![avatar](https://ririripley.github.io/assets/img/method_list_t_example.png)     
**classMethods example**  
![avatar](https://ririripley.github.io/assets/img/class_methods_example.png)       
**protocols example**  
![avatar](https://ririripley.github.io/assets/img/protocol_list_example.png)  
**instanceProperties example**      
![avatar](https://ririripley.github.io/assets/img/property_list_t_example.png)  
**Pseudocode of mechanism of Category Realization**  
```
objc_init{      
  category_ t **catlist = _getObjc2CategoryList(); //  obtain the category list of the class object
  // instance method implementation in category            
  if (cat-›instanceMethods || cat-›protocols || cat-›instanceProperties) {    
         remethodizeClass(cls);    
   }    
   // class method implementation in category   
   if (cat-›classMethods || cat-›protocols || (hasClassProperties && cat-›_classProperties)) {      
      remethodizeClass(cls->ISA());    
   }  
}  

void remethodizeClass(Class cls) {
   category_list  **cats = unattachedCategoriesForClass(cls, false/*not realizing*/);          
   attachCategories(cls, cats, true /*flush caches*/);  
}  
void attachCategories(Class cls, category_list *cats, bool flush_caches) {    
   // Allocate space for method_list, protocol_list and propeties_list    
   method_list_t **mlists = (method_list_t **)malloc(cats-›count * sizeof(*mlists));          
   property_list_ t **proplists = (property_list_t **)malloc(cats-›count * sizeof(*proplists));              
   protocol_list_t **protolists = (protocol_list_t **)malloc(cats-›count * sizeof(*protolists));              
      
   
   int mount = 0;    
   int propcount = 0;    
   int protocount = 0;    
   
   int i = cats-›count;    
   bool fromBundle = NO;    
   // Traverse the category_list      
   // Count backwards through cats to get newest categories first        
   while (i--)  {    
      auto& entry = cats-›list[i];        
      
      // put the method_list_t in mlists       
      method_list_t *mlist = entry.cat-›methodsForMeta(isMeta);      
      if (mlist) {    
         mlists[mcount++] = mlist;      
      }  
      
     // put the property_list_t in proplists           
     property_list_t *proplist = entry.cat-›propertiesForMeta(isMeta, entry.hi);               
     if (proplist) {    
         proplists[propcount++] = proplist;    
     }    
         
     // put the protocol_list_t in protolists      
     protocol_list_t *protolist= entry.cat-›protocols;        
     if (protolist) {    
         protolists[protocount++] = protolist;        
     }    
   }          
   // After the above traversal of catlist, all the added list has been stored in mlist, proplists, protolists respectively.  
  
  // Add all the added list in the original class rw struct  
  auto rw = cls-›data();  
  rw-›methods.attachLists(mlists,mcount);  
  free(mlists);    
  
  rw-›properties.attachLists(proplists,propcount);  
  free(proplists);  
  
  rw-›protocols.attachLists(protolists, protocount);    
  free(protolists);  
}    

// In this way, the original array will expand with addestLists added to the front of the array.    
void attachLists (List* const * addedLists, uint32_t addedCount) {
   if (addedCount == 0) return;  
   if (hasArray( )) {  
      uint32_t oldCount = array()-›count;  
      uint32_t newCount = oldCount + addedCount:  
      setArray( (array_t *)realloc(array(), array_t: :byteSize(newCount)));  
      array()-›count = newCount;  
      
      // void *memmove(void *__dst, const void *__src, size_t __len) -> Move "__len" bytes from address "__src" to address "__dst"     
      memmove(array()-›lists + addedCount, array()-›lists,oldCount * sizeof(array()-›lists(0]));  
      
      // void *memmove(void *__dst, const void *__src, size_t __len) -> Copies "__len" bytes from address "__src" to address "__dst"    
      memcpy(array()-›lists, addedLists,addedCount * sizeof(array()-›lists[0]));   
   }
}
```
Based on the pseudocode, we can summarize that the class object has expanded its method_array_t, protocol_array_t as well as  
properties_array_t. For better comprehension, let us learn about one essential method in objc_msg_send:     
```
//Through this method, the received find the corresponding IMP by SEL.    
getMethodNoSuper_nolock(Class cls, SEL sel)   
{   
    for (auto mlists = cls->data()->methods.beginLists(), end = cls->data()->methods.endLists(); mlists != end;++mlists)      
    {      
        method_t *m = search_method_list(*mlists, sel);      
        if (m) return m;      
    }   
      
    return nil;      
}   
// Since the method_list_t in category is added in the beginning of method_array_t(method_list_t *), its IMP is first found.     
```
5) It is possible to add properties and member variables to class by category?      
```
1) Properties can be added to category but the setter and getter method will not be generated automatically. We can use assocation   
manager to achieve this.  
2) Memeber variabled cannot be added by category since ivars info is fixed in the class object in compiling time. Ivars is also read-only in   
class_rw_t struct.        
```
6) What is the difference between **load** and **initialize** method ?  
**Initialize**:      
Called after main function    
mechanism: obj_send_message          
The runtime sends initialize to each class in a program just before the class,    
or any class that inherits from it, is sent its first message from within the program.     
Superclasses receive this message before their subclasses.    
If initialize method of subclass is not implemented, it will call super class initialize method.          
```   
callInitialize(Class cls)      
{      
   ((void(*) (Class, SEL))objc_msgSend) (cls, SEL_initialize);      
}      
As can be seen callInitialize is through function objc_msgSend which is introduced in the above that utilizes search_method_list method     
to find corresponding IMP.    
// The order of initialize methods: if the category has implemented initialize method, only the most front initialize method in metho_array_t will    
be executed. 
```
What should be paid attention to category is that the order of compiling sources determines the priority of method of    
a Class Object. The source file compiled later will be merged in the Class object later, in other words, the method of   
the last source file will be added to the most front position of method_array_t.
**Example**  
```
// In Parent.m  
+ (void)initialize {    
    NSLog(@"Initialize Parent, caller Class %@", [self class]);      
}    
  
// In Child.m    
// Initialized method is commented.    
  
// In main.m    
Child *child = [Child new];    
  
The printing result is :    
2017-04-05 22:25:56.056 load[742:32295] Initialize Parent, caller Class Parent      
2017-04-05 22:25:57.179 load[742:32295] Initialize Parent, caller Class Child       
```
In the above example, 'Child new' sends message to Child class object. Based on Apple documentation, initialize message is sent to     
Class object only once when it receives the first message in the program. Superclasses receive this message before their subclasses.        
Therefore, Parent Class object receive initialize message first, then comes to the Child class. Since Child class does not implement       
its own initialize method, initialize method of Parent Class is called twice.    
**Load**:  
Called before main function.    
mechanism: function_address      
If load method of subclass is not implemented, it will not call super class load method.       
```
void call_class_loads() {  
// As can be seen load_method is called through function address. (Function address of load method of Class cls).        
   for (i = 0; i < used; it+) {            
      Class cls = classes[i].cls;            
      load_method_t load_method = (load_method_t)classes[i].method;            
      (*load_method) (cls, SEL_load);        
   }  
}  
  
void call load methods(void) {    
   // call load method of class            
   call_class_loads();  
       
   // call load methods of all categories        
   more_categories = call_category_loads();        
}         
// The order of load methods: class load methods -> category load methods       
``` 
7) How to add properties by using category?  
This is an example of adding properties by category.       
```     
@interface Person (Fatter)  

@property (nonatomic, copy) NSString *liking;  
  
@end  

const void *kPersonLikingKey; // The key of the associated object  
  
@implementation Person (Fatter)   
  
# pragma mark - getter&setter  

- (NSString *)liking {  
    NSString *str = objc_getAssociatedObject(self, kPersonLikingKey);  
    return str;  
}  
  
- (void)setLiking:(NSString *)liking {  
    objc_setAssociatedObject(self, kPersonLikingKey, liking, OBJC_ASSOCIATION_COPY);  
}  
@end
```
![avatar](https://ririripley.github.io/assets/img/category_add_properties.png)  
**objc_setAssociatedObject**  
**objc_getAssociatedObject**  
objc_setAssociatedObject(self, kPersonLikingKey, liking, OBJC_ASSOCIATION_COPY);    
```
AssociationHashMap:     
{  
   key: id      
   value: ObjectAssociationMap          
}  
ObjectAssociationMap:   
{  
   key: void* (key of the property, i.e. kPersonLikingKey)            
   value: value of the property      
}   
In sum, what objc_setAssociatedObject(self, kPersonLikingKey, liking, OBJC_ASSOCIATION_COPY) does is :     
AssociationHashMap[self][kPersonLikingKey] = liking;      
what objc_getAssociatedObject(self, kPersonLikingKey) does is :  
return AssociationHashMap[self][kPersonLikingKey];   
```
6) When is the category merged into the class object ?  
```
Before main function is executed, _objc_init method will be called. What does _objc_init method do ?  
It will call the 'remethodizeClass' function as we mention before to merge category info into the class object.  Load methods of all  
class objects will also be called inside _objc_init.  
```
7) What is Class extension? What is the difference between extension and category ?   
**Example of Class Extension**  
```
// CutomizedView_extension.h
@interface CutomizedView ()
- (void) test;   
@end 


// UIView.m 
#import 'CutomizedView_extension.h'  
#import 'CutomizedView.h'  
@implementation CutomizedView    
- (void) test {    
   NSLog(@"test");  
}    
@end      
```
OR  
```
// UIView.m   
@interface CutomizedView ()  
- (void) test;     
@end   
  
@implementation CutomizedView      
- (void) test {    
   NSLog(@"test");    
}    
@end      
```
**Diff NO.1**:      
As can be seen, Class Extension requires modification of the .m file of the class, in other words, the source code  
of the Class needs to be open. Therefore, Class Extension cannot be applied in System Class while Class Category can.  
**Diff NO.2**:  
Class Extension is added to Class in compiling time while Class Category is added to class in run time.  
Using the command 'clang -rewrite-objc $ocFile_name -o $cpp_file_name', we can find that extension related code can be found   
in the project while category related code cannot. Actually, for category, the related code is added when the program begins running and 
executing **objc_init** method.  
**Diff NO.2**:  
Extension can add member variables while Category cannot.  
This is because in Class object class_rw_t struct, the ivars is read only which has been determined in compiling time.
8) What is Method Swizzling?    
Method swizzling is the process of changing the implementation of an existing selector.    
Here is an example of Method Swizzling for UIViewController Class. (Hook the system viewDidLoad method of UIViewController)      
```  
@interface UIViewController (swizzle)  
@end  
  
@implementation UIViewController (swizzle)  
// FYI, load method of each Class Object will be called before main function of the program is called.     
+ (void)load {  
    static dispatch_once_t onceToken;  
    dispatch_once(&onceToken, ^{  
        [self swizzle_viewDidLoad];  
    });  
}  
  
+ (void)swizzle_viewDidLoad {  
    [self exchangeMethodWithSelecter:@selector(viewDidLoad) toSelecter:@selector(swiz_viewDidLoad)];  
}  
  
- (void)swiz_viewDidLoad {  
    // Self-defined code you wish to execute when viewVidLoad is called.     
    NSLog(@"My own code");  
    // After executing the self-defined code, execute the function corresponding to SEL swiz_viewDidLoad.    
    // Since method_exchangeImplementations exchanges the implementations of SEL 'viewDidLoad' and SEL 'swiz_viewDidLoad'    
    // The SEL - IMP mapping is :     
    // viewDidLoad - > swiz_viewDidLoad      
    // swiz_viewDidLoad -> viewDidLoad    
        
    [self swiz_viewDidLoad];  // Executing the original viewDidLoad method.    
}  
  
/**  
 @param origin (The original method to be replaced)    
 @param destination (Method to replace the original one)    
 */  
+ (void)exchangeMethodWithSelecter:(SEL)origin toSelecter:(SEL)destination {  
    Method originMethod = class_getInstanceMethod(self, origin);  
    Method destinationMethod = class_getInstanceMethod(self, destination);  
    method_exchangeImplementations(originMethod, destinationMethod);  
}    
  
@end         
```
8) How to add properties and methods to class object at runtime ?      
8.1) Adding properties at runtime         
Add properties by using category. (objc_getAssociatedObject and objc_setAssociatedObject)    
8.2) Adding methods at runtime     
Let me introduce the message sending mechanism of object-c first.  
Here is an example of sending messages to an instance object.  
```
Person *p = [Person new];
[p sayHello];  
```   
1. get isa of obj_object struct   
2. isa points to the Person Class object.     
3. Look up SEL 'sayHello' in Person Class object's cached_methods, if failed, then look it up in method_array_t.     
4. If Step 3 succeeds, execute the IMP corresponding to SEL 'sayHello'.      
Otherwise, get superClass of Person Class object which points to the super Class object of Person Class and repeat Step 3.              
Step 4 will be recursively executed until finding the super Class which has the corresponding IMP.     
Here is one thing to pay attention to, the trace of class method is different from intance method.     
instance method: class -> super class of the class -> super class of the class' super class -> ...... -> NSObject     
class method: meta class -> super class of meta class -> super class of the meta class' super class -> ...... -> NSObject       
5. The upmost super class is NSObject, if the search all the way up to NSObject Class failed, there comes the **dynamically  
resolving** methods step -> **Fast Forwarding** -> **Normal Forwarding**.  
8.3) **Dynamically Resolving Methods**  
```
// Source Code of Dynamically Resolving Methods        
void _class_resolveMethod(Class cls, SEL sel, id inst)        
{        
    if (! cls->isMetaClass()) {            
         // Instance Method Resolving     
        _class_resolveInstanceMethod(cls, sel, inst);        
    }         
    else {        
        // Class Method Resolving         
        _class_resolveClassMethod(cls, sel, inst);        
        if (!lookUpImpOrNil(cls, sel, inst,         
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/))         
        {        
            _class_resolveInstanceMethod(cls, sel, inst);        
        }        
    }        
}          
```
At runtime, when the corresponding IMP of the selector cannot be found, resolveInstanceMethod or resolveClassMethod       
will be called as the second opportunity for developers to add methods. It can be notified that class method resolving involves     
both **class_resolveClassMethod** and **class_resolveInstanceMethod**. It is easy to understand the function of class_resolveClassMethod     
here, it adds method to the meta class.   
But what is **class_resolveInstanceMethod** for?   
If **class_resolveClassMethod** failed, the system will check whether the meta class has implemented **class_resolveInstanceMethod**,    
if it has, it means that methods maybe add to the meta class itself. Therefore, it is necessary to check the implementation  
of **class_resolveInstanceMethod** in case of missing the IMP. Usually, the struct of meta class is not attainable for developers,   
but there is one exception: NSObject class. NSObject class is the super class of NSObject meta class. In other words,  
NSObject class is the root class of the meta class. Therefore, if NSObject class implemented **class_resolveInstanceMethod**,      
the class resolving process will search all the way up to NSObject class finally and add corresponding IMP.    
Here is an example for **instance method**:      
```    
#import <objc/runtime.h>    
    
void globalMethod() {    
    NSLog(@"call global method");        
}    

@interface Person : NSObject    
- (void) instanceMethod;    
@end    

@implementation Person    
+(BOOL)resolveInstanceMethod:(SEL)sel {        
    NSLog(@"call resolveInstanceMethod");        
    if (sel == @selector(instanceMethod)) {    
        class_addMethod(self, sel, (IMP)globalMethod, "@v:");            
    }    
    return true;    
}    
    
@end    
    
int main(int argc, char * argv[])    
{    
    @autoreleasepool {        
        Person *person = [[Person alloc] init];            
        [person instanceMethod];        
        return 0;    
    }    
}    
The printing result is:  
call resolveInstanceMethod  
call global method      
```
FYI, class_addMethod: Adds a new method to a class with a given name and implementation:      
```
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);      
cls : The class to which to add a method.      
name: A selector that specifies the name of the method being added.    
imp: A function which is the implementation of the new method. The function must take at least two arguments—self and _cmd.    
```
As can be seen, the system will search whether Person has implemented **resolveInstanceMethod** static method, if not, then the search will  
further apply to Person Class's super class， which is NSObject class here. If searching all the way up to NSObject Class failed, it means the       
dynamically resolving methods failed, then the system will turn to *forwardingTargetForSelector* method, which will be introduced later.  
If the system found the implementation of **resolveInstanceMethod**, then the corresponding IMP towards the selector will be added.   
This time, the method searching will be successful.    
   
As for dynamically resolving methods for **class method**, it is kind of different.    
Here is an example for **class method**:  
```
@interface Person : NSObject    
+ (void) classMethod;  
@end  
  
@implementation Person  
  
+(BOOL)resolveClassMethod:(SEL)sel {    
    NSLog(@"call resolveClassMethod");  
    if (sel == @selector(classMethod)) {    
        class_addMethod(objc_getMetaClass(NSStringFromClass(self).UTF8String), sel, (IMP)globalMethod, "@v:");      
    }  
    return true;    
}  
@end  
  
int main(int argc, char * argv[])  
{   
    @autoreleasepool {  
        Person *person = [[Person alloc] init];       
        [Person classMethod];       
            return 0;     
    }       
}       
The printing result is:       
call resolveClassMethod       
call global method     
```  
The process is the same as **resolveInstanceMethod**. The system will search whether Person has implemented **resolveClassMethod** static method.    
The searching will go all the way up to NSObject Class.  
One more example for **class method** using **resolveInstanceMethod**:    
```  
#import <objc/runtime.h>    
    
void globalMethod() {    
    NSLog(@"call global method");    
}    

@implementation NSObject (instanceMethod)        
    
+ (BOOL)resolveInstanceMethod:(SEL)sel {    
      NSLog(@"call resolveInstanceMethod");    
    if (sel == @selector(classMethod)) {    
        class_addMethod(objc_getMetaClass(NSStringFromClass(self).UTF8String), sel, (IMP)globalMethod, "@v:");                
    }        
    return true;            
}    
@end        

@interface Person : NSObject        
+ (void) classMethod;        
- (void) instanceMethod;        
@end        
        
@implementation Person        
@end    
    
int main(int argc, char * argv[])    
{    
    @autoreleasepool {    
        [Person classMethod];            
        return 0;        
    }    
}    
The printing result is:       
call resolveInstanceMethod         
call global method       
```  
8.3) **Fast Forwarding**
Fast Forwarding can forward the message to the other object.  
Example:  
```
@interface Student : NSObject    
+(void)classMethodTestFastForwarding:(NSString *)strValue;    
-(void)instanceMethodTestFastForwarding:(NSString *)strValue;    
@end    
@implementation Student    
    
-(void)instanceMethodTestFastForwarding:(NSString *)strValue {    
    NSLog(@"Student instanceMethodTestFastForwarding value=%@",strValue);    
}    
    
+(void)classMethodTestFastlForwarding:(NSString *)strValue {    
    NSLog(@"Student classMethodTestFastForwarding value=%@",strValue);    
}    
@end    
    
@interface Person : NSObject    
+(void)classMethodTestFastForwarding:(NSString *)strValue;    
-(void)instanceMethodTestFastForwarding:(NSString *)strValue;    
@end    
    
@implementation Person    
-(id)forwardingTargetForSelector:(SEL)aSelector {    
    if ([NSStringFromSelector(aSelector) isEqualToString:@"instanceMethodTestFastForwarding:"]) {    
         Student*  student = [[Student alloc]init];    
        if ([student respondsToSelector:aSelector]) {    
            return student;    
        }    
    }    
    return [super forwardingTargetForSelector:aSelector];    
}    
    
+(id)forwardingTargetForSelector:(SEL)aSelector {    
    if ([NSStringFromSelector(aSelector) isEqualToString:@"classMethodTestFastForwarding:"]) {    
        if ([Student respondsToSelector:aSelector]) {    
            return [Student class];    
        }    
    }    
    return [super forwardingTargetForSelector:aSelector];    
}     
@end     
int main(int argc, char * argv[])        
{        
    @autoreleasepool {    
        Person* person = [[Person alloc] init];    
        [person instanceMethodTestFastForwarding: @"instanceMethodTestFastForwarding"];        
        [Person classMethodTestFastForwarding:  @"classMethodTestFastForwarding"];                
        return 0;            
    }        
}        
The printint result is :      
Student instanceMethodTestFastForwarding value=instanceMethodTestFastForwarding          
Student classMethodTestFastForwarding value=classMethodTestFastForwarding        
```
8.4) **Normal Forwarding**  
Normal Forwarding can forward the message to more than one object.    
Example:  
```
@interface Teacher : NSObject  
+(void)classMethodTestNormalForwarding:(NSString *)strValue;  
-(void)instanceMethodTestNormalForwarding:(NSString *)strValue;  
@end  
  
@implementation Teacher  
-(void)instanceMethodTestNormalForwarding:(NSString *)strValue {  
    NSLog(@"Teacher instanceMethodTestNormalForwarding value=%@",strValue);  
}  

+(void)classMethodTestNormalForwarding:(NSString *)strValue {  
    NSLog(@"Teacher classMethodTestNormalForwarding value=%@",strValue);  
}  
@end  
  
@interface Student : NSObject  
+(void)classMethodTestNormalForwarding:(NSString *)strValue;  
-(void)instanceMethodTestNormalForwarding:(NSString *)strValue;  
@end  
@implementation Student  
  
-(void)instanceMethodTestNormalForwarding:(NSString *)strValue {  
    NSLog(@"Student instanceMethodTestNormalForwarding value=%@",strValue);  
}  
  
+(void)classMethodTestNormalForwarding:(NSString *)strValue {  
    NSLog(@"Student classMethodTestNormalForwarding value=%@",strValue);  
}  
@end  
  
@interface Person : NSObject  
+(void)classMethodTestNormalForwarding:(NSString *)strValue;  
-(void)instanceMethodTestNormalForwarding:(NSString *)strValue;  
@end  
  
@implementation Person  
  
-(void)forwardInvocation:(NSInvocation *)anInvocation {  
    SEL selector = anInvocation.selector;  
    BOOL found = FALSE;  
    if ([NSStringFromSelector(selector) isEqualToString:@"instanceMethodTestNormalForwarding:"]) {  
        Teacher* teacher = [[Teacher alloc] init];  
        Student* student = [[Student alloc] init];  
        if ([teacher respondsToSelector:selector]) {  
            [anInvocation invokeWithTarget:teacher];  
            found = YES;  
        }  
        if ([student respondsToSelector:selector]){  
            [anInvocation invokeWithTarget:student];  
            found = YES;  
        }    
          
        // optional  
        if (!found) {  
            [self doesNotRecognizeSelector:selector];  
        }   
    }  
}  
  
-(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {  
    NSMethodSignature * sign = [super methodSignatureForSelector:aSelector];  
    if (!sign) {  
        sign = [NSMethodSignature signatureWithObjCTypes:"v@:@"];  
    }  
    return sign;  
}  
  
+(void)forwardInvocation:(NSInvocation *)anInvocation {  
    SEL selector = anInvocation.selector;  
    BOOL found = FALSE;  
    if ([NSStringFromSelector(selector) isEqualToString:@"classMethodTestNormalForwarding:"]) {  
        if ([Teacher respondsToSelector:selector]) {  
            [anInvocation invokeWithTarget:[Teacher class]];  
            found = YES;  
        }  
        if ([Student respondsToSelector:selector]){  
            [anInvocation invokeWithTarget:[Student class]];  
            found = YES;  
        }  
          
        // optional  
        if (!found) {  
            [self doesNotRecognizeSelector:selector];  
        }   
    }  
}  
  
+(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {  
    NSMethodSignature * sign = [super methodSignatureForSelector:aSelector];  
    if (!sign) {  
        sign = [NSMethodSignature signatureWithObjCTypes:"v@:@"];  
    }  
    return sign;  
}  
@end  
  
int main(int argc, char * argv[])  
{  
      
    @autoreleasepool {  
        Person *person = [[Person alloc] init];  
        [person performSelector:@selector(instanceMethodTestNormalForwarding:) withObject:@"instanceMethodTestNormalForwarding"];  
        [Person performSelector:@selector(classMethodTestNormalForwarding:) withObject:@"classMethodTestNormalForwarding"];  
         return 0;  
    }  
}    
The printint result :  
Teacher instanceMethodTestNormalForwarding value=instanceMethodTestNormalForwarding  
Student instanceMethodTestNormalForwarding value=instanceMethodTestNormalForwarding  
Teacher classMethodTestNormalForwarding value=classMethodTestNormalForwarding  
Student classMethodTestNormalForwarding value=classMethodTestNormalForwarding  
```
8.5) What is **Tagged Pointer**?  
In 64-bit system, a pointer occupies 8 bytes while the value it stores does not require so much space. Thus, the 8 bytes memory  
space is divided by half. One half stores the value while the other half stores the address.  The optimization can also apply to  
isa pointer.  Half of the isa bytes can store the reference number while the other half stores the address of the class object.  
Such an isa pointer is called **NON_POINTER_ISA**.  
9)Convert Dictionary to Object Model  
```
@{  
@"name" : @"Xiaoming",    
@"age" : @18,    
@"sex" : @"男"     
}  
```
Get the ivars info of the class object, fill the value of each member variables of the class object.    
Ivar  _Nonnull * class_copyIvarList(Class cls, unsigned int *outCount);    
- (void)setValue:(id)value forKeyPath:(NSString *)keyPath;  
```
#import "NSObject+Model.h"  
#import <objc/runtime.h>  

@implementation NSObject (Model)  

+ (instancetype)ModelWithDict:(NSDictionary *)dict {  
    NSObject * obj = [[self alloc] init];    
    [obj transformDict:dict];  
    return obj;  
}  

- (void)transformDict:(NSDictionary *)dict {  
    Class cla = self.class;  
    // count: number of ivars   
    unsigned int outCount = 0;    
    Ivar *ivars = class_copyIvarList(cla, &outCount);     
       
    // traverse all the ivars and fill the value        
    for (int i = 0; i < outCount; i++) {      
        Ivar ivar = ivars[i];  
        // get the name of ivar   
        NSString *key = [NSString stringWithUTF8String:ivar_getName(ivar)];     
           
        // get the property name corresponding to the ivar     
        key = [key substringFromIndex:1];   
             
        id value = dict[key];     
             
        if (value == nil) continue;     
        // set the value of the property    
        [self setValue:value forKeyPath:key];    
    }  
    free(ivars);   
}   
```
10)Objects Archiving And Unarchiving  
Classes whose instances need to be archived and unarchived must conform to the NSCoding protocol and implement its two required methods,     
encodeWithCoder: and initWithCoder:.  
@protocol NSCoding    
- (void)encodeWithCoder:(NSCoder *)aCoder;      
- (instancetype)initWithCoder:(NSCoder *)aDecoder;      
@end    
```
#import "Items.h"  
@interface Items ()<NSCoding>  
@property (nonatomic, copy) NSString *name;  
@property (nonatomic, copy) NSString *age;  
@end  

@implementation Items  
- (void)encodeWithCoder:(NSCoder *)aCoder{  
    [aCoder encodeObject:_name forKey:@"name"];  
    [aCoder encodeObject:_age forKey:@"age"];  
}  
  
- (instancetype)initWithCoder:(NSCoder *)coder  
{  
    self = [super init];  
    if (self) {  
        _name = [coder decodeObjectForKey:@"name"];  
        _age = [coder decodeObjectForKey:@"age"];  
    }  
    return self;  
}  
@end  

@interface Person  
@property (nonatomic) NSMutableArray *privateItems;  
- (BOOL)saveChanges;  
@end   

@implementation Person  

- (NSString *)itemArchivePath  
{  
    // Make sure that the first argument is NSDocumentDirectory  
    // and not NSDocumentationDirectory  
    NSArray *documentDirectories = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);  
  
    // Get the one document directory from that list  
    NSString *documentDirectory = [documentDirectories firstObject];  
  
    return [documentDirectory stringByAppendingPathComponent:@"items.archive"];  
}  

- (void) createItem {  
   Items *item = [[Items alloc] init];  
   [self.privateItems addObject:item];  
}  

// Archive all the items in the file   
- (BOOL)saveChanges    
{  
    NSString *path = [self itemArchivePath];  
    // Returns YES on success, archiveRootObject will send encodeWithCoder messsage to each item in the privateItems array     
    return [NSKeyedArchiver archiveRootObject:self.privateItems toFile:path];   
}   

// Unarchive all the items from the file
- (instancetype)initPrivate   
{   
    self = [super init];   
    if (self) {   
        _privateItems = [[NSMutableArray alloc] init];   
        NSString *path = [self itemArchivePath];   
        // unarchiveObjectWithFile will send initWithCoder messsage to form the _privateItems array    
        _privateItems = [NSKeyedUnarchiver unarchiveObjectWithFile:path];   
        // If the array hadn't been saved previously, create a new empty one     
        if (!_privateItems) {   
            _privateItems = [[NSMutableArray alloc] init];     
        }   
}   
       return self;     
}   
```
Instead of performing encode and decode task for each property of the object, we can utilize the runtime characteristic to  
optimize the code. Encode and decode can be written as follows:
```
(void)encodeWithCoder:(NSCoder *)aCoder{  
  
    unsigned int count = 0;  
    Ivar * ivars = class_copyIvarList([Person class], &count);  
    for (int i = 0; i < count; i++) {  
        Ivar ivar = ivars[i];  
        const char * name = ivar_getName(ivar);  
        NSString * key = [NSString stringWithUTF8String:name];  
        [aCoder encodeObject:[self valueForKey:key] forKey:key];  
    }  
}  
  
- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder{  
  self = [super init];  
  if (self) {  
        unsigned int count = 0;  
        Ivar * ivars = class_copyIvarList([Person class], &count);  
        for (int i = 0; i < count; i++) {  
            Ivar ivar = ivars[i];  
            const char * name = ivar_getName(ivar);  
            NSString * key = [NSString stringWithUTF8String:name];  
            id value = [aDecoder decodeObjectForKey:key];  
            [self setValue:value forKey:key];  
        }  
  }  
  return self;  
}  
```
11)What is the difference between 'calling a method' and 'sending a message' ?   
Calling a method:  
```
Function calls are really just jumping to a certain spot in memory and executing code. There's no dynamic behavior involved.  
```
Sending a message:    
```
Objects can choose to not respond to messages, or forward messages on to different objects, or whatever. More details are entailed  
in  objc_msgSend() function.  
```
12)Runtime Methods Cache  
To speed the messaging process, the runtime system caches the selectors and addresses of methods as they are used.    
The mechanism is entailed in the **cache_t** struct introduced in section **(1.6) id**.    
13) Type Encoding  
The Objective-C compiler generates type encodings for all the types.  
```  
For example,   
-(void)hello:(NSString*)value; ----->  "v@:@"     
v    :   return value is void  
@    ：  receiver (type: id)  
:    :   SEL  
@    ：  parameter (type: id)    
```
14)Multiple-inheritance    
Objective-C doesn't support multiple inheritance. Use composition or protocols.    

15)Mechanism Of Perform Selector
```
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2 {    
     IMP msg;  
     if (aSelector == 0) {  
         [NSException raise: NSInvalidArgumentException format: @"%@ null selector given", NSStringFromSelector(_cmd)];    
     }  
     // find the coreresponding IMP of selector 
     msg = objc_msg_lookup(self, aSelector);  
     if (!msg) {  
         [NSException raise: NSGenericException format: @"invalid selector '%s' passed to %s", sel_getName(aSelector), sel_getName(_cmd)];  
         return nil;  
     }  
     return (*msg)(self, aSelector, object1, object2);  
  }  
```
16)Runtime Get Class Related Method              
Class object_getClass(id obj);          
```     
The class object of which object is an instance, or Nil if object is nil.          
Class object_getClass(id obj)     
{     
   if (obj)      
      return obj->getIsa();          
   else      
      return Nil;          
}                
```     
id objc_getClass(const char *name);     
```     
return the Class object for the named class, or nil if the class is not registered with the Objective-C runtime.       
```     
+ (Class)class;     
```     
//Returns the class object.     
+ (Class)class {     
  return self;     
}     
```     
- (Class)class;     
```     
Returns the class object for the receiver’s class.     
- (Class)class {     
    return object_getClass(self);     
}     
```     
id objc_getMetaClass(const char *name);     
```     
Returns the metaclass definition of a specified class.     
```     
### **Reference**  
https://cloud.tencent.com/developer/article/1156752    
https://www.jianshu.com/p/fa66c8be42a2    
https://www.jianshu.com/p/51e4c117d596    
https://anyeler.top/2018/07/29/Category%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/    
https://cloud.tencent.com/developer/article/1391529    
https://www.jianshu.com/p/ab5d7c81b810  
https://www.incredibuild.com/integrations/clang  
https://juejin.cn/post/6890350559128190983#heading-5  
https://cloud.tencent.com/developer/article/1799493    
https://juejin.cn/post/7029343711112724517    
https://blog.csdn.net/u010105969/article/details/62233752    
https://davedelong.tumblr.com/post/58428190187/an-observation-on-objective-c    
https://www.jianshu.com/p/6fb4641e6ec5    
https://stackoverflow.com/questions/4192203/objective-c-multiple-inheritance
https://blog.chenyalun.com/2018/09/30/PerformSelector%E5%8E%9F%E7%90%86/






