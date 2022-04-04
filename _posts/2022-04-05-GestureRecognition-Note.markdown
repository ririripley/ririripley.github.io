---
author: ripley
comments: false
date: 2022-04-03 12:11:08+00:00
layout: post
slug: GestureRecognition
title: Analysis of Gesture Recognition
wordpress_id: 304
categories:
- Tech
tags:
description: Mechanics of Gesture Binding
---
## **Reading Notes About Gesture Recognition**   
The PDF as follows illustrates the mechanics of Gesture Recognition. Flutter providers differents kinds of gesture recognizer which handles corresponding    
pointer event, DoubleTap Recognizer dealing with double tap, i.e..  When one pointer event takes place, it will be delocated a pointer ID and a gesture arena.  
The gesture arena is owned by a global mangaer called Gesture Manager. Gesture Manager is in charge of a global routers which handles the reponsibility of dispatching pointer route event   
added by Gesture Recognizers.  Every time a pointer event happens, the gesture manager will first dispatch the added pointer route and event and then choose to close or sweep the arena based   
on its state.  A more specific example of double tap is illustrated in the final part of this PDF in details.      
    
<object data="https://ririripley.github.io/assets/img/Gesture.pdf" type="application/pdf" width="1000px" height="1400px">
    <embed src="https://ririripley.github.io/assets/img/Gesture.pdf">
        <p>This browser does not support PDFs. Please download the PDF to view it: <a href="https://ririripley.github.io/assets/img/Gesture.pdf">Download PDF</a>.</p>
    </embed>
</object>  
### **参考**     
https://www.epubit.com/bookDetails?id=UB7812bd93e7e58

