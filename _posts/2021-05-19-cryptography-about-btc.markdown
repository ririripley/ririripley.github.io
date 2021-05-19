---
author: ripley
comments: false
date: 2021-05-12 16:02:08+00:00
layout: post
slug: about_network_security
title: About Network Security
wordpress_id: 304
categories:
- Tech
tags:
description: Cryptography About Bitcoin
---
## **Elliptic Curve Cryptography (ECC)**
#### **Explanation**
(1)LOG Problem:  
```  
given a, a<sup>x</sup>,  
 we can get x = log<sub>a</sub>(a<sup>x</sup>)     
```
(2)Elliptive Curve  
y<sup>2</sup> = x<sup>3</sup> + ax + b   
![avatar](https://ririripley.github.io/assets/img/ellipticCurve.png)  
define:    
```    
D = A + B     
```  
![avatar](https://ririripley.github.io/assets/img/ellipticCurve_tangency.png)  
define:      
```  
C = A + A     
```  
Let's further define:  
```  
C = A * 2
2 * C = C + C 
3 * C = 2 * C + C  
...
k * C = (k - 1) * C + C  
```
for any given k and j, it satisfies:  
k * (j * C) = (kj) * C= (jk) * P = j * ( k * P )  

How to calculate k * C ?
```
9C (e.g.)  
  
9C = 8C  + C

8C = 4C + 4C 
4C = 2C + 2C    
2C = C + C
it takes 4 mathemetical operations.  
Time Complexity:  
O(logk)  
```
Question:    
Back to (1)LOG problem, given kC, C, can we get k ?  
Answer:     
Tough task. You have to try k one by one.  
#### **Application - ECDH**
![avatar](https://ririripley.github.io/assets/img/open_baidu.png)  
Elliptic Curve Diffie-Hellman Ephemeral （ECDHE)  
```
基于ECC的秘钥交换
```  
服务端确定了密钥协商算法为“EC Diffie-Hellman”，发送给客户端。现在两端都知道了使用的是哪个曲线参数（椭圆曲线E、阶N、基点G）。   

premaster secret 计算公式: 用于生成master secret
客户端:  
客户端随机生成一个整数c, 计算pubkey_c = c * G         
> PreMasterSecret：Q = pubkey_s * c = c(s * G)  
    
服务端:   
服务端随机生成一个整数s，计算pubkey_s = s * G       
>  PreMasterSecret：Q = pubkey_c * s = s(c * G)   
    
#### **Bitcoin**  
    

  
