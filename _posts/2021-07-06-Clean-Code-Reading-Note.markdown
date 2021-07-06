---
author: ripley
comments: false
date: 2021-07-06 12:26:08+00:00
layout: post
slug: CodeFormatting
title: Reading Note of Clean Code
wordpress_id: 304
categories:
- Tech
tags:
description: CodeCC
---
## **Clean Code**  
Here are some excerpts from *Clean Code* useful for me.   
#### **Comments**  
```
(1)Comments do not make up for bad code, instead, you should explain yourself the code.    
```
![avatar](https://ririripley.github.io/assets/img/CLEANCODEcomments.png)  
![avatar](https://ririripley.github.io/assets/img/CLEANCODEcommentsmodel.png)    
```
(2)Point out warning of consequences.  
(3)Write TO-DO comments.  
```  
```
(4)No Commented-Out-Code  
```
![avatar](https://ririripley.github.io/assets/img/CLEANCODECommentedOutCode.png)
```
(5)Some Necessary Explanation
i.e. the rationale behind the logic, usually seed in algorithm design     
```        
#### **Classes**  
```
(1)Structure Of Class  
public static var  
private static var    
private instance var  
public functions  
private functions  
(2)Classes Should Be Small  
SRP(single responsibility principle)  
(3)Classes should be open for extension but closed for modification  
Using OO programming instead of procedural programming.      
i.e. override, extends, implements    
(4)Classes should depend upon abstractions, not on concrete details.  
i.e. Just ask for value from the interface.      
```    
#### **Objects & Data Structure**
```    
(1) Data Transfer Obkects 
Private variables better manipulated by getters and setters.       
```
![avatar](https://ririripley.github.io/assets/img/CLEANCODEobjectsmodel.png)  
```    
(2) Law of Demeter
A method f of Class C should only call the methods of these:  
[1]C  
[2]An object created by f  
[3]An object passed as an argument to f  
[4]An object held in an instance variable of C   
```      
#### **Meaningful Names**  
```
(1)Use Pronounceable Names  
```
![avatar](https://ririripley.github.io/assets/img/CLEANCODEusePronouceableNames.png)
```  
(2)Classes and objects should have noun or noun phase names like Customer, Manager.  
(3)Methods should have verb or verb phase names like getNum, save.  
```
```  
(4)When constructors are overloaded, use static factory methods with name that describe the arguments.    
```
![avatar](https://ririripley.github.io/assets/img/CLEANCODEfactoryConstructorModel.png)  
```  
(5)Don't use similar names for different concepts or different names for the same concept.  
bad example:  productInfo productData       
```
