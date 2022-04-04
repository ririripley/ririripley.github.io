---
author: ripley
comments: false
date: 2022-04-05 09:11:08+00:00
layout: post
slug: Navigator
title: Flutter App Router - Navigator
wordpress_id: 304
categories:
- Tech
tags:
description: Mechanics of Navigator
---
## **Reading Notes About Navigator**   
The PDF as follows illustrates the mechanics of page router for Flutter App: Navigator. In sum, navigator uses a list to store pushed routes.  
When Navigator is being built, routes of the list will be handled one by one based on their properties(whether visible or not) and build method.   
    
<object data="https://ririripley.github.io/assets/img/Navigation.pdf" type="application/pdf" width="1200px" height="1400px">
    <embed src="https://ririripley.github.io/assets/img/Navigation.pdf">
        <p>This browser does not support PDFs. Please download the PDF to view it: <a href="https://ririripley.github.io/assets/img/Navigation.pdf">Download PDF</a>.</p>
    </embed>
</object>  
### **参考**     
https://www.epubit.com/bookDetails?id=UB7812bd93e7e58

