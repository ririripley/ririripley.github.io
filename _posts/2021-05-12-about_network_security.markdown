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
---
### **End-point Communication**
网络上的End-point communication需要保证以下几个方面:
```
(1）Confidentiality: secret   
(2) Authetication  
(3) Integrity: 消息确实来自某人以及not tampered    
```
而网络攻击者可能会进行以下攻击:  
```
(1) 读message  
(2) 修改message  
```

### **Confidentiality:  Cryptography**

Cryptography就是：
```  
plaintext ----Encryption Algorithm + key---> ciphertext  
(nothing more than mathematical operations)
```
K<sub>B</sub>(K<sub>A</sub>(m)) = m   
K<sub>A</sub>(m): ciphertext encrypted using K<sub>A</sub>  
  
#### **1. symmetric key systems**
Ripley's and Zijun's keys are identical and secret.  
##### **Example**
DES  
AES  

#### **2. public key systems**
A pair of keys is used. The public key is known to both Ripley and Zijun (Actually, known to the whole world.).  
The private key is only known to either Zijun or Ripley.  
##### **Example**
RSA  
包含两个部分：  
```
(1) choice of public key and private key,  
(2) encryption and dectryption algorithm
```
##### **concerns about public-key cryptography**
```
(1)任何人可以宣称自己是Ripley然后向Zijun发送message.    
----> 怎么办？ Solution: Digital Signature (绑定sender和message)  
(2) time-consuming (DES is at least 100 times faster than RSA in hardware.)    
----> 怎么办？ Solution: Combination with symmetric key cryptography  
```
#### **keys**
```
(1)session key  
(2)premaster key  
(3) 
```   

### **Integrity:  Cryptography**

#### **Cryptographic Hash Functions(goal : Ensure Message Not to be tampered)**

```   
A crytographic hasn funciton should satisfy:  
If x == y : H(x) = H(y)  
x != y : H(x) != H(y)  
    
```
同时， x (arbitraty length) ---Hash Function---> H(x) (fixed length)      
##### **Example**
SHA  
##### **Issue**
仍无法确定发送方是否为发送方本身  
Solution:   MAC  
  
#### **Message Authentication Code (MAC) : Ensure the origin of message**
```
双方拥有a shared secret : authentication key  
MAC = H(message + authentication key)    
发送方发送messgae + MAC
接收方接收， then  check H(message + authentication key) ?= MAC    
```
![avatar](https://ririripley.github.io/assets/img/figure8_9_Message_authentication_code.png)
                            图片参考文献[1]  

##### **Issue**
如何distribute the shared authentication key        
Solution:  + 加密然后发送   


#### **Digital Signature**
证明某人签署了材料  
思路：  
```
必须附有something unique to the signer
Q: 可否采用MAC作为signature?
A: The answer is no. In this case, the key will be no more unique to the signer.
```
可采用public-key cryptography:  
K<sub>+</sub>(K<sub>-</sub>(m)) = m    
既保证了消息的origin, 也保证了消息not to be tampered. 可见 public-key cryptography满足了message integrity 的要求。  
signature =  K<sub>-</sub>(m)  
##### **Issue**
Expensive Computation  
Solution: 引入hash functions    
signature =  K<sub>-</sub>(H(message)) (H(m) is of fixed-length, leading to less computation.)  
发送方发送: message + signature  
接收方接收， then check H(message) ?=  K<sub>+</sub>signature.  
  
  




sudo chown -R $USER /usr/local
//use for local user
```    

或者用另一种更适合于-global安装的修改方式。

```
sudo chown -R `whoami` ~/.npm
//better for -global install


这样之后你再试试跑nodeschool上面的教程试试看，应该已经可行了。


### **参考**
[1^] James F. Kurose and Keith W. Ross. 2012. Computer Networking: A Top-Down Approach (6th Edition) (6th. ed.). Pearson.