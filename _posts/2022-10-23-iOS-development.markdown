---
author: ripley
comments: false
date: 2022-10-23 12:11:08+00:00
layout: post
slug: iOSDevelopment
title: iOS Development UIKit part
wordpress_id: 304
categories:
- Tech
tags:
description: iOS Development for UIKit part
---
## **UIKit**
What is the relationship between UIView and CALayer:
```
1) UIView can be regarded as a layer-host view in which CALayer is repsonsible for drawing, displaying & animation, 
while the view is responsible for interaction(e.g. handling touch events).  
2) CALayer is a subclass of NSObject; UIView is a subclass of UIResponder.    
3) UIView has a property LAYER to obtain the underlying CALayer of it. The undelying CALayer's delegate = UIView.  
UIView conforms to CALayerDelegate protocol to respond to layer-related events, by implementing the protocol, UIView is able to  
respond to layer-related events such as providing the layer’s content(METHOD displayLayer:) and handling the layout of sublayers.  
```
What is the difference between bounds and frame:  
```
(1) Bounds is the view's own coodinate system.    
(2) Frame is the view's position and size in the perspective of its superview's coordinate system. The width and height of frame   
is not necessarily equal to which of the bounds. (e.g. a rotated UIView)  
```
What is LOADVIEW method for:  
```
(1) A UIViewController can set its view in two methods: 1. Implementing loadView method programmatically. 2. Create a nib file.  
(2) If method one is not implemented, UIViewController will automatically call the designated method initWithNibName:WithBundle: in  
which the nib name is the controller's name and the bundle is mainBundle by default parameters are nil. If no such nib file exists, a  
generic view will be assigned as the view controller's view.       
(3) If method one is implemented, UIViewController will gain its view through method one.   
``` 
What is the super class of UIButton and UILabel respectively:
```
(1) UIButton -> UIControl -> UIView -> UIResponder    
(2) UILabel -> UIView -> UIResponder         
(3) UIControl provides methods such as addTarget:action:forControlEvents: as well as removeTarget:action:forControlEvents:.  
```
How to implement didReceiveMemoryWarning:  
```
(1) didReceiveMemoryWarning is the message received by UIViewController when the app receives a memory warning.    
(2) In iOS6, when the system sent memory warning messages, it will also automatically recycle those memory occupied by bitmap which  
is a memory-costly object owned by CALayer. In this way, without the recyclization of UIView and CALayer, most memory could be released.  
And the bitmap can be rebuilt easily by drawRect method when needed. 
(3) In didReceiveMemoryWarning, we need to call the super method and then dispose of any resources which aren't absolutely critical. 
Most of the view controllers will be hanging onto data caches, intermediary data, or other bits and pieces, often to save recalculation.    
```
Lifecycle of UIView:  
```
(1) When a view is added as subview, viewWillAppear will be called.  
(2) Before the view is displayed on the screen, viewWillLayoutSubViews and viewDidLayoutSubViews will be called followed by drawRect method, and finally  
viewDidAppear is called.   (layoutSubviews -> drawRect)
(3) When the view is removed from super view, viewWillDisAppear will be called followed by viewDidDisappear.       
```
Lifecycle of UIViewController:
```
(1)-[ViewController loadView]  
(2)-[ViewController viewDidLoad]  
(3)-[ViewController viewWillAppear:]  
(4)-[ViewController viewWillLayoutSubviews]  
(5)-[ViewController viewDidLayoutSubviews]  
(6)-[ViewController viewDidAppear:]  
(7)-[ViewController viewWillDisappear:]  
(8)-[ViewController viewDidDisappear:]  
(9)-[ViewController dealloc]  
(10)-[ViewController didReceiveMemoryWarning]         
```
How to get the current displaying UIViewController on the screen?
```
// header file: UIWindow+DisplayVc.h  
@interface UIWindow   
- (UIViewController *) visibleViewController;    
@end
       
// implementation file: UIWindow+DisplayVc.m  
#import "UIWindow+DisplayVc.h"  

@implementation UIWindow (DisplayVcs)
  
- (UIViewController *)visibleViewController {  
    UIViewController *rootViewController = self.rootViewController;  
    return [UIWindow getVisibleViewControllerFrom:rootViewController];  
}
  
+ (UIViewController *) getVisibleViewControllerFrom:(UIViewController *) vc {  
    if ([vc isKindOfClass:[UINavigationController class]]) {  
        return [UIWindow getVisibleViewControllerFrom:[((UINavigationController *) vc) visibleViewController]];  
    } else if ([vc isKindOfClass:[UITabBarController class]]) {  
        return [UIWindow getVisibleViewControllerFrom:[((UITabBarController *) vc) selectedViewController]];  
    } else {  
        if (vc.presentedViewController) {  
            return [UIWindow getVisibleViewControllerFrom:vc.presentedViewController];  
        } else {  
            return vc;  
        }  
    }  
}  
@end  
```
What is the relationship beween method layoutIfNeed and setNeedsDisplay?   
```
When layoutIfNeeded is called, the current event roop will check the mark symbol needsUpadteConstraints, needsUpadteLayout,  
needsDisplay immediately and execute their corresponding methods: upadteConstraints, layoutSubviews, drawRect. All these will be  
done before the layoutIfNeeded method returns. Using setNeedsDisplay, setNeedsLayout, setNeedsUpdateConstraints will mark corresponding  
symbols but actions will not execute until next update cycle.            
```
How does the gesture recognizer and UI refresh under the hood?    
![avatar](https://ririripley.github.io/assets/img/iOSAppRenderProcess.png)  
```
Here I assume you have a basic understanding about run loop mechanism. The following figures show the calling stack about the gesture         
recognizer and UI refresh:                
```
The is the view hierarchy and settings of the app.   
![avatar](https://ririripley.github.io/assets/img/view_hierarchy.png)     
1) How touch events are intercepted by the gesture recognizer?  
When we try to apply pan gesture within the scope of the white view(BNRDrawView), here is what happens:  
![avatar](https://ririripley.github.io/assets/img/gr_interception.png)    
As can be seen from the log, all three views has received the touch:begin message, but when the gesture recognizer of the  
blue view(superView) has been recognized, all three views has received the touch:cancel message and after that, all touch events are no more   
sent to these views but only handled by the gesture recognizer. And the view of UITouch(orignally BNRDrawView since it is the first responder in the hit-test)          
is also set to nil since the gr has been recognized.    
2) How are gesture recognizers handled?  
![avatar](https://ririripley.github.io/assets/img/source1_hit_test_mechanism_in_run_loop.png)     
![avatar](https://ririripley.github.io/assets/img/source1_hit_test_mechanism_in_run_loop_2.png)  
![avatar](https://ririripley.github.io/assets/img/source1_responder_chain_in_run_loop.png)      
![avatar](https://ririripley.github.io/assets/img/gr_in_run_loop.png)  
When we swiped the screen on the top of BNRDrawView, SpringBoard.app will send the touch events to the app through mach port. At this time, if the app is    
sleeping, it will wake up and deal with the source1 event. The source1 handler will generate source0 event by converting touch events to UIEvent. Then it is     
time for the app to deal with source0 events:    
hit-test to find the first responder -> UIWindow sentEvent to the first responder ->  responder chain  ->  gr recognized and intecepted the responder chain ->    
gr marked as dirty ->  _UIGestureRecognizerUpdateObserver()(BeforeWaiting) in the run loop execute the callback of dirty gr   
3) How does UI refresh?  
![avatar](https://ririripley.github.io/assets/img/UI_refresh_in_run_loop.png)      
BNRDrawView setNeedsDisplay -> BNRDrawView marked as dirty -> _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()(BeforeWaiting) in the run loop          
traverse the dirty View / CALayer and execute their drawRect method.    
### **Reference**  
https://blog.csdn.net/qq_28551705/article/details/83989115    
https://github.com/ririripley/ririripley.github.io/blob/master/_posts/2023-01-08-iOS-runloop-mechanism.markdown    

