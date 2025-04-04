---
author: ripley
comments: false
date: 2023-01-14 12:11:08+00:00
layout: post
slug: iOSDevelopment
title: iOS Foundation
wordpress_id: 304
categories:
- Tech
tags:
description: Foundation
---
## **Foundation**    
1)What is the difference between nil、NIL、NSNULL？      
```    
typedef struct objc_class *Class;    
    
struct objc_object {    
//（objc_class）    
Class isa  OBJC_ISA_AVAILABILITY;    
};    
    
struct objc_class {    
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;    
    Class _Nullable super_class;        
    const char * _Nonnull name;          
    long version;                 
    long info;                    
    long instance_size;    
    struct objc_ivar_list * _Nullable ivars;    
    struct objc_method_list * _Nullable * _Nullable methodLists;    
    struct objc_cache * _Nonnull cache;    
    struct objc_protocol_list * _Nullable protocols;    
#endif    
    
//A singleton object used to represent null values in collection objects that don’t allow nil values.    
@interface NSNull : NSObject    
+ null //Returns the singleton instance of NSNull.    
```    
1.1) nil    
```    
-> null pointer to Objective-C object (struct objc_object*)    
NSString *someString = nil;    
id someObject = nil;    
```    
1.2) NIL    
```    
-> null pointer to Objective-C class (struct objc_class*)    
Class someClass = Nil;    
Class anotherClass = [NSString class];    
```    
1.3)NULL    
```    
-> null pointer to primitive type or absence of data    
int *pointerToInt = NULL;    
char *pointerToChar = NULL;    
struct TreeNode *rootNode = NULL;    
```    
1.4)NSNull    
```    
NSNull is a class for objects that represent null.    
NSNull is often used in Foundation collections since they cannot store nil values.    
NSMutableDictionary *dict = [NSMutableDictionary dictionary];    
    
// the following throws an exception because dictionaries cannot store nil values:    
[dict setObject:nil forKey:@"someKey"];     
    
// valid code    
[dict setObject:[NSNull null] forKey:@"someKey"];    
```    
2)NSCache & NSMutableDictionary & NSMutableArray    
```     
NSMutableDictionary and NSMutableArray are not thread-safe. They are not designed to be simultaneously read or written          
by multiple threads. So you have to use serial threads if multiple threads needs to visit the data.       
NSCache is thread-safe.  It makes use of some policies so as to delete some data if the application is short of memory.    
```     
For example, we need to apply lock mechanism when an NSMutableArray maybe modified in mutiple threads.    
```    
NSLock *arrayLock = [[NSLock alloc] init];    
    
...    
    
[arrayLock lock]; // NSMutableArray isn't thread-safe    
[myMutableArray addObject:@"something"];    
[myMutableArray removeObjectAtIndex:5];    
[arrayLock unlock];    
```    
3)What is the difference between MAC, UUID, UDID, IDFV and IDFA?    
3.1)  UDID (Unique Device Identifier)    
```    
The unique identifier of the Apple iOS device. It is banned by Apple.    
```    
3.2)  UUID (Universally Unique Identifier)    
```    
UUID is based on the application layer. Apple recommends using a UUID to generate a unique identification string for your app.     
Developers can call once when the app is first launched, and then store the string instead of Udid.     
However, if the user deletes the app and then installs again, a new string is generated, so it is not guaranteed to uniquely identify the device.     
With UUID, consider the processing of the application being removed and then reinstalled.     
One solution: The UUID is usually generated only once, stored in the iOS system, if the app is deleted,     
after reloading the app its UUID is the same, unless the system resets.     
However, there is no guarantee that the system will be available after the upgrade (if the system saves the information).    
```    
3.3）MAC Address    
```    
MAC address is used on the network to differentiate the uniqueness of the device. It is banned by Apple.    
```    
3.4) OPEN UDID    
```    
The openudid for each iOS device is generated by the first app with the Openudid SDK package,     
and if you completely remove all apps with the Openudid SDK package (such as a recovery system, etc.),     
Openudid will regenerate and will be different from the previous values. equivalent to new equipment;    
```    
3.5) IDFA (Identifier For Advertising)    
```    
IDFA is another new method in iOS 6, provides a method advertising identifier, by calling the method will return a Nsuuid instance, finally can get a uuid, stored by the system.     
However, even though this is stored by the system, there are several cases where the ad identifier is regenerated.     
This ad identifier is regenerated if the user completely resets the system (restore location and privacy, general-purpose, Setup, and so on).     
In addition, if the user explicitly restores the ads (set-up, general-purpose, and so on, to the ads, and restore the ad identifiers),     
then the ad identifiers will be regenerated.     
With respect to the restoration of the ad identifier, it is important to note that if the program is running in the background,     
the user "restores the ad identifier" and then goes back to the program, and then gets the ad identifier and does not immediately get the restored identifier.     
You must terminate the program before restarting the program to obtain the restored AD identifier.    
All the apps in the same device get the same IDFA.     
Users can reset IDFA or choose to protect IDFA from beging read.     
```    
3.6) IDFV (Identifier For Vendor)    
```    
IDFV is for vendor to identify the user, each device in the same vender application, have the same value.     
The vender refers to the application provider, but to be exact, it is matched by the first two parts of the Bundleid DNS reversal, if the same is the same vender,     
for example, for Com.somecompany.appone, Com.somecompany.apptwo These two bundleid, they belong to the same vender, sharing the value of the same IDFV.     
Unlike IDFA, the value of IDFV can be taken, so it is well suited to identify the user as the primary ID for internal user behavior analysis, instead of Openudid.    
Note: If the user uninstalls all apps that belong to this vender, the IDFV value will be reset, that is,     
the APP,IDFV value of this vender will be re-installed before the values are different.    
```    
4) What's the difference between the atomic and nonatomic attributes?    
   Atomic guarantees thread safety for reading and writing.       
   4.1) What's actually happening for nonatomic attributes?    
```    
//@property(nonatomic, retain) UITextField *userName;    
//Generates roughly    
    
- (UITextField *) userName {    
    return userName;    
}    
    
- (void) setUserName:(UITextField *)userName_ {    
    [userName_ retain];    
    [userName release];    
    userName = userName_;    
}    
```    
4.2) What's actually happening for atomic attributes?    
```    
//@property(retain) UITextField *userName;    
//Generates roughly    
- (UITextField *) userName {    
      UITextField *retval = nil;    
      @synchronized(self) {    
        retval = [[userName retain] autorelease];    
      }    
      return retval;    
  }    
    
- (void) setUserName:(UITextField *)userName_ {    
  @synchronized(self) {    
      [userName_ retain];    
      [userName release];    
      userName = userName_;    
  }         
}         
```         
5) Hash & IsEqual Method         
   5.1)isEqual    
```         
Check if two objects have the same value.         
```         
Example:    
```         
@interface Person : NSObject         
@property (nonatomic, copy) NSString *name;         
@property (nonatomic, strong) NSDate *birthday;         
@end         
         
@implementaion Person         
- (BOOL)isEqual:(id)object {         
    if (self == object) {         
        return YES;         
    }         
             
    if (![object isKindOfClass:[Person class]]) {         
        return NO;         
    }         
             
    return [self isEqualToPerson:(Person *)object];         
}         
         
- (BOOL)isEqualToPerson:(Person *)person {         
    if (!person) {         
        return NO;         
    }         
             
    BOOL haveEqualNames = (!self.name && !person.name) || [self.name isEqualToString:person.name];         
    BOOL haveEqualBirthdays = (!self.birthday && !person.birthday) || [self.birthday isEqualToDate:person.birthday];         
             
    return haveEqualNames && haveEqualBirthdays;         
}         
@end         
```         
5.2) == (Attention, Objective-C does not supper operator overriding)    
```         
For OC object type: Check if two objects are the same object(share the same space address).         
For OC primitive type: Check if two objects have the same value.          
```         
5.3) Hash method    
```         
In order to improve the efficiency, NSSet and NSDictionary will execute following steps while checking if objects are equal:           
Step 1:          
    if hash(member) == hash(target) {         
        go to step 2         
    } else {         
        return false         
    }         
Step 2: if (member isEqualTo tagrget) {         
            return true         
        } else {         
            return false         
        }            
                 
Object hash method will be called when it is added to NSSet or set as key of NSDictionary.          
For example:         
erson *person1 = [Person personWithName:kName1 birthday:self.date1];         
Person *person2 = [Person personWithName:kName2 birthday:self.date2];         
NSMutableSet *set1 = [NSMutableSet set];         
[set1 addObject:person1]; // hash method of person1 will be called          
NSMutableDictionary *dictionaryKey2 = [NSMutableDictionary dictionary];          
[dictionaryKey2 setObject:kValue2 forKey:person2]; // hash method of person2 will be called          
```         
5.3.1) How to implement Hash Method    
```         
[super hash] method returns the space address of the object. We should not directly use [super hash] as hash method for     
class objects. The following example illustrates the problem:            
         
Person *person1 = [Person personWithName:kName1 birthday:self.date1];         
Person *person2 = [Person personWithName:kName1 birthday:self.date1];         
NSLog(@"[person1 isEqual:person2] = %@", [person1 isEqual:person2] ? @"YES" : @"NO");         
         
NSMutableSet *set = [NSMutableSet set];         
[set addObject:person1];         
[set addObject:person2];         
NSLog(@"set count = %ld", set.count);  // print 2, but actually we expect one since the two objects are equal         
```               
Hash Method Valid Implementation:    
```           
@interface Person : NSObject         
@property (nonatomic, copy) NSString *name;              
@property (nonatomic, strong) NSDate *birthday;         
@end         
         
@implementaion Person         
         
- (NSUInteger)hash {         
  return [self.name hash] ^ [self.birthday hash];         
}         
@end          
```         
6) What is the difference between id and instancetype ?         
   Both id and instancetype can be used as pointer refering to any object.         
   id: a reference to some random Objective-C object of unknown class         
   instancetype: a reference to instance of the class to which the method belongs    
    
The difference lies in that under ARC environment, **instancetype** will perform type checking while **id** will not.         
**instancetype** can only be used in constructor return type.  **id** can be used as parameter type of return type.          
Here is an example as follows showing the difference:    
```         
@interface MyObject : NSObject         
+ (instancetype) factoryMethod1;          
+ (id) factoryMethod2;         
@end         
         
Line 1: [[MyObject factoryMethod1] count]; // this will throw error since MyObject class does not have [count] method         
Line 2: [[MyObject factoryMethod2] count]; // Will not throw any error since id will not perform type checking in compilation period         
```         
7) Covariant & Contravariant in generics type         
   7.1)Generics Example    
```         
@interface Person<T> :NSObject         
@property (nonatomic, strong) T language;         
@end          
```         
7.2) Covariant & Contravariant keyword         
Keyword Covariant and Contravariant are used to describe the generics type about its conversion.           
Covariant Example:    
```         
@interface Language : NSObject         
@end         
         
@interface iOS : Language         
@end         
         
@interface Person<__covariant T> : NSObject         
@property (nonatomic, strong) T language;         
@end         
- (void)covariant {         
    iOS *ios = [[iOSalloc]init];         
    Language *language = [[Languagealloc]init];         
         
    Person<iOS *> *p = [[Personalloc] init];         
    p.language = ios;         
             
    Person<Language *> *p1 = [[Personalloc] init];         
             
    // without __covariant keyword, an error will throw here         
    // cast subclass(iOS) to superclass(Language)           
    p1 = p;          
}         
```         
Contravariant Example:    
```         
@interface Person<__contravariant T> : NSObject         
@property (nonatomic, strong) ObjectType language;         
@end         
         
- (void)contravariant {         
    iOS *ios = [[iOSalloc]init];         
    Language *language = [[Language alloc] init];         
         
    Person<Language *> *p = [[Person alloc] init];         
    p.language = language;         
         
    Person<iOS *> *p1 = [[Personalloc]init];         
    // without __contravariant keyword, an error will throw here         
    // cast subclass(Language) to superclass(iOS)         
    p1 = p;           
}         
```         
8) Conversion Method Related To NSString         
   8.1)Conversion between SEL and NSString    
```         
FOUNDATION_EXPORT NSString *NSStringFromSelector(SEL aSelector);         
FOUNDATION_EXPORT SEL NSSelectorFromString(NSString *aSelectorName);         
```         
8.2)Conversion between Class and NSString    
```         
FOUNDATION_EXPORT NSString *NSStringFromClass(Class aClass);         
FOUNDATION_EXPORT Class __nullable NSClassFromString(NSString *aClassName);         
```         
8.3)Conversion between Protocol and NSString    
```         
FOUNDATION_EXPORT NSString *NSStringFromProtocol(Protocol *proto) NS_AVAILABLE(10_5, 2_0);         
FOUNDATION_EXPORT Protocol * __nullable NSProtocolFromString(NSString *namestr) NS_AVAILABLE(10_5, 2_0);         
```         
9)Application Sandbox         
Every iOS application you create is placed in its own sandbox. A sandbox is a directory on the iOS file system.          
A sandbox holds your application’s executable file, which includes all embedded resources.    
```         
1)Caches	         
Persist between runs of the application.         
Does not get backed up when the device is synchronized with iTunes.         
2)Preferences	         
This directory is where any preferences are stored and where the Settings application looks for application preferences.          
Preferences is handled automatically by the class NSUserDefaults and is backed up when the device is synchronized with iTunes.         
3)Documents	         
Persist between runs of the application.          
Data stored in this folder is backed up when the device is synchronized with iTunes.          
4)tmp	         
Files placed in the tmp folder are temporary and should be deleted when an application terminates.          
If the application does not clean the folder, iOS might delete them depending on when space is needed on your device.          
It does not get backed up when the device is synchronized with iTunes.         
```         
10)iOS Header Files Import    
```         
1) #include:         
Used in C/C++         
Simple copy the code from header .h file to current file.         
2) #import:         
It is the same as #include. What is more, it can avoid repeated header file reference. For example,          
when #import "A.h" and #import "B.h" and both "A.h" and "B.h" include the same "C.h", it means "C.h" is repeated included.          
#import can figure out the case and include only once "C.h".            
e.g.          
What is the difference between #import "***" and #import <***>          
#import "***" : import local header file, which means the header file is editable for the developers          
#import <***> : import system header file, which means the header file is read-only since it belongs to the system         
3) What is the difference between #import and @class         
@class is used when you need to know the name of a class in a particular file, but you don't need to know any details about the class (its methods, for example).          
#import is used when you actually need to use the class (i.e., send it a message).         
For example,         
          
//Since you're not using myIvar yet, you don't need to know anything about it except that the type MyOtherClass exists.         
@class MyOtherClass;         
@interface MyClass : NSObject         
{         
    MyOtherClass *myIvar;         
}         
@end         
         
// You're sending the doSomethingElse message to myIvar; the compiler needs to know that instances of MyOtherClass define this method,         
// so you have to import the header file or the compiler will complain.         
#import "MyOtherClass.h"         
- (void)doSomething         
{         
    [myIvar doSomethingElse];         
}         
When it comes to compilation, sometimes using @class can help to improve efficiency.           
When you #import file A into file B, file B becomes dependent upon file A -- that is, if file A changes,          
you'll have to recompile file B.          
If you use @class in file B, file B is not dependent on file A, and thus doesn't need to be recompiled when file A changes --          
so if you're just declaring a type and not actually dependent upon the implementation of file A, you can save yourself compilation time by not #importing file A.         
```    
### **Reference**    
https://www.jianshu.com/p/3447f3230b29    
https://stackoverflow.com/questions/5908936/difference-between-nil-nil-and-null-in-objective-c    
https://www.jianshu.com/p/145d5b3c67dd    
https://topic.alibabacloud.com/a/how-does-ios-get-a-unique-identifier-for-a-device-what-are-the-meanings-of-idfa-idfv-and-font-classtopic-s-color00c1deudidfont-respectively_1_12_30733336.html    
https://stackoverflow.com/questions/588866/whats-the-difference-between-the-atomic-and-nonatomic-attributes    
https://www.jianshu.com/p/915356e280fc    
https://www.jianshu.com/p/b00c116eae61    
    
    
