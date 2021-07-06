---
author: ripley
comments: false
date: 2021-07-06 11:11:08+00:00
layout: post
slug: Networking
title: A Day in the Life of a Web Page Request
wordpress_id: 304
categories:
- Tech
tags:
description: A Day in the Life of a Web Page Request
---
## **The Journey of A HTTP Request**  
#### **Example**  
Let's say that one student, Bob, wants to downloads a web page(www.google.com). Then there is a lot going on:  
```
(1) Connecting to local Ethernet: DHCP, UDP, IP, Ethernet
(2) Send HTTP Request: DNS, ARP
(3) Web Client-Server Interaction: TCP and HTTP  
```
To summarize, in order to get the webpage content from Internet,  
(1)  Bob's laptop first needs to connect to local Ethernet, during which DHCP is required for initializing the laptop's components     
including default gateway router, default DNS server,  network mask, the allocated IP address to the laptop.      
(2)  After that, Bob types the URL and UDP is used to retrieve the IP address of the domain(www.google.com), but in order to send the    
UDP packet,  an ARP request is sent so as to obtain the MAC address of the default gateway router. The UDP packet wrapped in IP datagram    
will be first send to the gateway router, then to the DNS server. In this way, Bob can obtain the IP address as a response from the    
DNS server.    
(3) With the IP address, a HTTP request asking for the content of webpage will reach the IP address navigated by the routers distributed    
in network.   
The PDF as follows illustrates what is going on with a HTTP request in details:  
    
<object data="https://ririripley.github.io/assets/img/LifeOfWebPageRequest.pdf" type="application/pdf" width="1000px" height="1400px">
    <embed src="https://ririripley.github.io/assets/img/LifeOfWebPageRequest.pdf">
        <p>This browser does not support PDFs. Please download the PDF to view it: <a href="https://ririripley.github.io/assets/img/LifeOfWebPageRequest.pdf">Download PDF</a>.</p>
    </embed>
</object>  

