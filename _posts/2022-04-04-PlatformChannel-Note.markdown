---
author: ripley
comments: false
date: 2022-04-01 12:11:08+00:00
layout: post
slug: PlatformChannel
title: Platform Channel
wordpress_id: 304
categories:
- Tech
tags:
description: Introduction To Platform Channel
---
## **Reading Notes About Platform Channel**   
The PDF as follows illustrates the implementation of platform channel including message channel, method channel and event channel.   
Literally, message channel is used to send messages from dart to native side. As for method channel, it is used to call native method from dart side.   
Event channel is mostly used for frequent calls of native method from dart side, keyboard event notice, i.e.  :  
Platform View is provided by flutter for developers to apply native UI components in flutter project and it is achieved by the application  
of platform channel to share texture information: texture ID. The core of platform view is the rendering native texture in flutter.  

    
<object data="https://ririripley.github.io/assets/img/platformChannel.pdf" type="application/pdf" width="1000px" height="1400px">
    <embed src="https://ririripley.github.io/assets/img/platformChannel.pdf">
        <p>This browser does not support PDFs. Please download the PDF to view it: <a href="https://ririripley.github.io/assets/img/platformChannel.pdf">Download PDF</a>.</p>
    </embed>
</object>  
### **参考**     
https://www.epubit.com/bookDetails?id=UB7812bd93e7e58

