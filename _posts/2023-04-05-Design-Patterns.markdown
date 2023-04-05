---
author: ripley
comments: false
date: 2023-04-05 08:11:08+00:00
layout: pos
slug: iOSDevelopment
title: Design Patterns
wordpress_id: 304
categories:
- Tech
tags:
description: Design Patterns
--- 
## **Design Patterns**   
1)Template Method Pattern
![avatar](https://ririripley.github.io/assets/img/TemplateMethod.png)          
```    
Define a skeleton and defer the varying steps to subclasses.        
This pattern keeps a fairly tight coupling between the sub and superclasses.        
```
2)Strategy Pattern        
```
Object composition instead of inheritance.         
```
3)Singleton Design Pattern        
```        
Restricts the instantiation of a class to one object.        
 Sometimes we need to have only one instance of our class for example a single DB connection shared by multiple objects as creating a separate DB connection for every object may be costly.          
```        
## **SOLID Principles**
1)Single Responsibility        
```
T is responsible for two different duties: responsibility P1, responsibility P2. When the class T needs to be modified due to the change in the requirement of the responsibility P1,                 
it may cause the P2 function that was originally functioning normally to malfunction. That is to say, duties P1 and P2 are coupled together.        
Solution:        
Comply with different responsibilities and package different responsibilities into different classes or modules        
```        
2)Open/Closed        
```
A software entity such as classes, modules, and functions should be open to extensions and closed to modifications.        
When software needs to change, try to implement changes by extending the behavior of the software entities,         
rather than modifying the existing code to implement the changes.        
```
3)Liskov Substitution Principle LSP        
```
The derived class can replace the base class, and the function of the software unit is not affected.        
```
4)Interface Segregation
```
The client should not rely on interfaces it does not need; the dependency of one class on another should be based on the smallest interface.        
```
5)Dependency Inversion Principle
```        
High-level modules should not rely on low-level modules, both of which should rely on their abstractions;         
abstractions should not rely on details; details should rely on abstractions.        
The origin of the problem:        
Class A directly depends on class B. If you want to change class A to dependent class C, you must do so by modifying the code of class A.         
In this scenario, class A is generally a high-level module responsible for complex business logic;         
class B and class C are low-level modules that are responsible for basic atomic operations;         
if class A is modified, it will introduce unnecessary risks to the program.        
Solution:        
A ---depends on--- B / C   ----change to-----   A ---depends on --- interface I, B,C ---implements--- interface I        
Modify class A to rely on interface I. Class B and class C each implement interface I.         
Class A indirectly communicates with class B or class C through interface I, which greatly reduces the chance of modifying class A.        
```        
6)Law of Demeter        
```        
An object should have as little knowledge as possible about other objects and not speak to strangers.        
Another way of saying : only communicate with direct friends        
Direct friend:        
Classes in member variables, method parameters, method return values,        
Indirect friends:        
Class in local variables        
```        
## **Common Design Patterns**          
```         
1) builder  : e.g. HyperLinkCodeBuilder : setProperty + build Method(returns hyperlinkcode )         
2) singleTon          
3) Observer: one to many e.g. KVO + NSNotificationCenter          
4) Adapter:             
5) Delegate: e.g. ScrollView Delegate ---> OnScrollUpdate           
6) Strategy : e.g. A (B b, C c) --> Composite        
7) Template Method: inheritance        
```
### **Reference**        
https://medium.com/@mena.meseha/6-principles-of-software-design-3a8478954e1c        
https://cloud.tencent.com/developer/article/1458158        