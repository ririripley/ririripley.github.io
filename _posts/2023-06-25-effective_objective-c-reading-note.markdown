---
author: ripley
comments: false
date: 2023-06-25 08:11:08+00:00
layout: pos
slug: iOSDevelopment
title: Reading Note Of Effective Objective-C
wordpress_id: 304
categories:
- Tech
tags:
description: Reading Note Of Effective Objective-C
--- 
## **Objective-C Objects**   
1)
```    
The memory for objects is always allocated in heap space and never on the stack. It is illegal to declare a stack-allocated Objective-C object: NSString stackString;
```
2)
```
Sometimes in Objective-C, you will encounter variables that don’t have a * in the definition and might use stack space. These variables are not holding Objective-C objects. An example is CGRect, from the CoreGraphics framework: CGRect frame;
A CGRect is a C structure, defined like so: 

struct CGRect {
  CGPoint origin;
  CGSize size;
};
typedef struct CGRect CGRect;
```
## **Minimize Importing Headers in Headers**
1)Forward Declaring The Class
Declare classes in a header and import their corresponding headers in an implementation. 
Doing so avoids coupling classes together as much as possible.  Deferring the import to where it is required   
enables you to limit the scope of what a consumer of your class needs to import.
```
// EOCPerson.h File
#import <Foundation/Foundation.h>
//  To compile anything that uses EOCPerson, you don’t need to know the full details about what an EOCEmployer is.   
// All you need to know is that a class called EOCEmployer exists.
@class EOCEmployer;  
  
@interface EOCPerson : NSObject
@property (nonatomic, strong) EOCEmployer *employer; 
@end      

// EOCPerson.m  File File  
#import "EOCPerson.h"  
// The implementation file for EOCPerson would then need to import the header file of EOCEmployer,     
// as it would need to know the full interface details of the class in order to use it.  
#import "EOCEmployer.h"  
@implementation EOCPerson  
// Implementation of methods 
@end  
```   
Using forward declaration also alleviates the problem of both classes referring to each other.
Sometimes, forward declaration is not possible, as when declaring protocol conformance.  
In such cases, consider moving the protocol-conformance declaration to the class-continuation category(extension), 
if possible. Otherwise, import a header that defines only the protocol. 
## **Prefer Literal Syntax over the Equivalent Methods**
1)Literal Arrays
However, you need to be aware of one thing when creating arrays using the literal syntax. 
If any of the objects is nil, an exception is thrown, since literal syntax is really just syntactic sugar around creating an array 
and then adding all the objects within the square brackets.
```         
NSArray *animals = @[@"cat", @"dog", @"mouse", @"badger"];                 
```
Now consider the scenario in which object1 and object3 point to valid Objective-C objects, but object2 is nil. 
The literal array, arrayB, will cause the exception to be thrown. However, arrayA will still be created but will contain 
only object1. The reason is that the arrayWithObjects: method looks through the variadic arguments until it hits nil, 
which is sooner than expected. This subtle difference means that literals are much safer.
```
NSArray *arrayA = [NSArray arrayWithObjects:object1, object2, object3, nil];
NSArray *arrayB = @[object1, object2, object3];
```
(2)Literal Dictionaries  
```
// The objects and keys have to all be Objective-C objects.
NSDictionary *personData =
    @{@"firstName" : @"Matt",
      @"lastName" : @"Galloway",
      @"age" : @28};
```
(3) Limitation Of Literal Syntax
, the class of the created object must be the one from the Foundation framework. 
There’s no way to specify your own custom subclass that should be created instead.
## **Prefer Typed Constants to Preprocessor**
(1) Internal Const Variable
The preprocessor will blindly replace all occurrences of ANIMATION_DURATION, so if that were declared in a header file,
anything else that imported that header would see the replacement done.
```
// NOT RECOMMENDED
#define ANIMATION_DURATION 0.3
```
Define a constant than using a prepro- cessor define, and this definition has type information
The const qualifier means that the compiler will throw an error if you try to alter the value. In this scenario, that’s exactly what is required.
The static qualifier means that the variable is local to the translation unit in which it is defined.
A translation unit is the input the compiler receives to generate one object file. In the case of Objective-C,   
this usually means that there is one translation unit per class: every implementation (.m) file. 
If the variable were not declared static, the compiler would create an external symbol for it. 
If another translation unit also declared a variable with the same name, the linker would throw an error with a message.
```
// RECOMMENDED
static const NSTimeInterval kAnimationDuration = 0.3;
```
A constant that does not need to be exposed to the outside world should be defined in the implementation file where it is used.
e.g.
```
// EOCAnimatedView.h File
#import <UIKit/UIKit.h>
@interface EOCAnimatedView : UIView 
- (void)animate;
@end

// EOCAnimatedView.m File
#import "EOCAnimatedView.h"
static const NSTimeInterval kAnimationDuration = 0.3;
@implementation EOCAnimatedView 
- (void)animate {
    [UIView animateWithDuration:kAnimationDuration
                     animations:^(){ // Perform animations}];
  }
@end
```
The usual convention for constants is to prefix with the letter k for constants that are local to a translation unit (implementation file). 
For constants that are exposed outside of a class, it is usual to prefix with the class name.
(2)Expose A Constant Externally
The constant has to be defined once and only once.
The constant is “declared” in the header file and “defined” in the implementation file.
The "extern" keyword tells the compiler that there will be a symbol for EOCStringConstant in the global symbol table. The compiler simply knows that the constant will exist when the binary is linked
```
// In the header file
extern NSString *const EOCStringConstant;
// In the implementation file
NSString *const EOCStringConstant = @"VALUE";
``` 
Prefixing with the class name that the constant relates to is prudent and will help you avoid potential clashes.
## **Use Enumerations for States, Options, and Status Codes**
(1)Enumerations
``` 
// The compiler gives a unique value to each member of the enumeration, starting at 0 (by default if not specified )and increasing by 1 for each member.
enum EOCConnectionState: NSInteger { 
    EOCConnectionStateDisconnected = 1, 
    EOCConnectionStateConnecting, 
    EOCConnectionStateConnected,
};
typedef enum EOCConnectionState EOCConnectionState;
EOCConnectionState state = EOCConnectionStateDisconnected;
``` 
(2)Bit Representation
``` 
enum UIViewAutoresizing {
    UIViewAutoresizingNone = 0,
    UIViewAutoresizingFlexibleLeftMargin  = 1 << 1,
    UIViewAutoresizingFlexibleWidth UIViewAutoresizingFlexibleRightMargin = 1 << 2, 
    UIViewAutoresizingFlexibleTopMargin = 1 << 3, 
    UIViewAutoresizingFlexibleHeight = 1 << 4, 
    UIViewAutoresizingFlexibleBottomMargin = 1 << 5,
}
``` 
(3) A Safer Way
The NS_OPTIONS macro is defined in different ways if compiling as C++ or not. If it’s not C++, it’s expanded out the same as NS_ENUM. 
However, if it is C++, it’s expanded out slightly differently since The C++ com- piler acts differently when two enumeration values are bitwise OR’ed together.
For this reason, you should always use NS_OPTIONS if you are going to be ORing together the enumeration values. If not, you should use NS_ENUM.
``` 
typedef NS_ENUM(NSUInteger, EOCConnectionState) 
{   EOCConnectionStateDisconnected, 
    EOCConnectionStateConnecting, 
    EOCConnectionStateConnected,
};
typedef NS_OPTIONS(NSUInteger, EOCPermittedDirection) {
    EOCPermittedDirectionUp    = 1 << 0,
    EOCPermittedDirectionDown  = 1 << 1,
    EOCPermittedDirectionLeft  = 1 << 2,
    EOCPermittedDirectionRight = 1 << 3,
};
``` 
The first argument for "NS_ENUM" is the type used to store the new type.
In a 64-bit environment, EOCConnectionState will be bytes long–same as NSUInteger.
Make sure that the specified size can fit all of the defined values, or else an error will be generated.
The second argument is the name of the new type.
Inside the block, the values are defined as usual.
