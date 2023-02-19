---
author: ripley
comments: false
date: 2023-02-18 08:11:08+00:00
layout: post
slug: iOSDevelopment
title: Build And Launch An App
wordpress_id: 304
categories:
- Tech
tags:
description: Cocopods, Build And Compiling
--- 
## **Cocopods**    
1)How does Cocopods work?      
For example, here is a pod file:    
```        
// Podfile     
pod 'AFNetworking', :git => 'https://ririripley/AFNetworking.git', :tag => '3.2.1'      
```    
1.1)According to **:git => ‘git@gitlab.xxx.net:ios-thirdpartservice/xxxreact.git’**, locate the corresponding repo.      
1.2)According to **:tag => ‘1.0.0’**, locate the correpsonding commit.    
1.3)Find the .podspec file in the corresponding commit:    
```        
Pod::Spec.new do |s|    
  s.name     = 'AFNetworking'    
  s.version  = '3.2.1'    
  s.license  = 'MIT'    
  s.summary  = 'A delightful iOS and OS X networking framework.'    
  s.homepage = 'https://github.com/AFNetworking/AFNetworking'    
  s.social_media_url = 'https://twitter.com/AFNetworking'    
  s.authors  = { 'Mattt Thompson' => 'm@mattt.me' }    
  s.source   = { :git => 'https://github.com/AFNetworking/AFNetworking.git', :tag => s.version, :submodules => true }    
  s.requires_arc = true    
      
  s.public_header_files = 'AFNetworking/AFNetworking.h'    
  s.source_files = 'AFNetworking/AFNetworking.h'    
      
  pch_AF = <<-EOS    
#ifndef TARGET_OS_IOS    
  #define TARGET_OS_IOS TARGET_OS_IPHONE    
#endif    
#ifndef TARGET_OS_WATCH    
  #define TARGET_OS_WATCH 0    
#endif    
#ifndef TARGET_OS_TV    
  #define TARGET_OS_TV 0    
#endif    
EOS    
  s.prefix_header_contents = pch_AF    
      
  s.ios.deployment_target = '7.0'    
  s.osx.deployment_target = '10.9'    
  s.watchos.deployment_target = '2.0'    
  s.tvos.deployment_target = '9.0'    
  s.subspec 'Security' do |ss|    
    ss.source_files = 'AFNetworking/AFSecurityPolicy.{h,m}'    
    ss.public_header_files = 'AFNetworking/AFSecurityPolicy.h'    
    ss.frameworks = 'Security'    
  end    
end    
```    
1.4) Check whether **s.name** is the same as the one in podfile(both **AFNetworking**). If yes, then find the    
files which need to be imported, in this case, **'AFNetworking/AFNetworking.h'**. As can be seen as follows, **AFNetworking.h**       
as well as **AFSecurityPolicy.h** and **AFSecurityPolicy.m** is exposed in **Pod** dir.      
![avatar](https://ririripley.github.io/assets/img/project_pods_tree.png)    
1.5) Finally, your actual project pulls in the code through either pod install then the physical files get downloaded    
to your mac and copied for your project under **Pod** dir.    
2)Podsepc file    
```    
A specification describes a version of Pod library. It includes details about where the source should be fetched from which tag or commit or branch,     
what files to use, the build settings to apply, and other general metadata such as its name, version, and description.    
```    
3)Subspecs    
Subspecs describes the norms of submodule of a repo.    
```    
Pod::Spec.new do |spec|    
spec.name          = 'ShareKit'    
spec.source_files  = 'Classes/ShareKit/{Configuration,Core,Customize UI,UI}/**/*.{h,m,c}'    
# ...    
    
spec.subspec 'Evernote' do |evernote|    
evernote.source_files = 'Classes/ShareKit/Sharers/Services/Evernote/**/*.{h,m}'    
end    
    
spec.subspec 'Facebook' do |facebook|    
facebook.source_files   = 'Classes/ShareKit/Sharers/Services/Facebook/**/*.{h,m}'    
facebook.compiler_flags = '-Wno-incomplete-implementation -Wno-missing-prototypes'    
facebook.dependency 'Facebook-iOS-SDK'    
end    
# ...    
end    
```    
With the above example a Podfile using **pod 'ShareKit'** command results in the inclusion of the whole library,    
while **pod 'ShareKit/Facebook'** command can be used if you are interested only in the Facebook specific parts in which    
the part of subspec in the Podfile of shareKit works.         
4)Podfile.lock & Manifest.lock    
When **pod install** and **pod update** command is exectuted , cocoapods will generate a **podfile.lock** file, which records the version.    
```    
// Podfile.lock    
PODS:    
- AFNetworking (3.2.1):    
    - AFNetworking/NSURLSession (= 3.2.1)    
    - AFNetworking/Reachability (= 3.2.1)    
    - AFNetworking/Security (= 3.2.1)    
    - AFNetworking/Serialization (= 3.2.1)    
    - AFNetworking/UIKit (= 3.2.1)    
- AFNetworking/NSURLSession (3.2.1):    
    - AFNetworking/Reachability    
    - AFNetworking/Security    
    - AFNetworking/Serialization    
- AFNetworking/Reachability (3.2.1)    
- AFNetworking/Security (3.2.1)    
- AFNetworking/Serialization (3.2.1)    
- AFNetworking/UIKit (3.2.1):    
    - AFNetworking/NSURLSession    
    
DEPENDENCIES:    
- AFNetworking (~> 3.2.1)    
    
SPEC REPOS:    
trunk:    
- AFNetworking    
    
SPEC CHECKSUMS:    
AFNetworking: b6f891fdfaed196b46c7a83cf209e09697b94057    
    
PODFILE CHECKSUM: 798ae8c2e51bc1cfec4a132a30b3bf5219febe9e    
    
COCOAPODS: 1.11.3    
```    
When we execute the command **pod install** or **pod update**, cocoapods will generate a podfile.lock file, which record all the    
modules in details. Then here comes the question, since we already have podfile, why do we still need podfile.lock for version control.    
There are two reasons:    
Firstly, we do not always specify exact versions of our pods, like : **pod 'A'**.    
Secondly, even though we do specify the version of A like **pod 'A', '1.0.0'**, it is hard to gurantee the dependency has all specified    
the exact version. One typical example is if the pod A has a dependency on pod A2 — declared in A.podspec as dependency 'A2', '~> 3.0'.    
(FYI, the '~> 3.0' here means a version up to 4.0 (but not including 4.0 and higher))    
In such case, using pod 'A', '1.0.0' in your Podfile will indeed force user1 and user2 to both always use version 1.0.0 of the pod A, but:    
user1 might end up with pod A2 in version 3.4 (because that was A2's latest version at that time)    
while when user2 runs pod install when joining the project later, they might get pod A2 in version 3.5    
(because the maintainer of A2 might have released a new version in the meantime).    
Therefore, we need podfile to guarantee the versions of all moduls in our pods dir. Since Pods dir is usually git ignored.    
```    
Manifest.lock is a copy of podfile.lock which is under the Pods dir. Every time we build the project, the script will check whether podfile.lock      
is the same as Manifest.lock, if not, then you will see some warnings like this:     
"error: The sandbox is not in sync with the Podfile.lock. Run 'pod install' or update your CocoaPods installation."    
This mechanism is to remind the developer that the versions of local pods are not consistent with what the project specify which can help     
to avoid problems causing by version inconsistency.    
```    
5)Difference between pod install & pod update & pod repo update    
pod install    
```    
Every time the pod install command is run — and downloads and install new pods.    
It writes the version it has installed, for each pods, in the Podfile.lock file. This file keeps track of the installed version    
of each pod and locks those versions.     
Conforming priority : podfile -> podlock    
For example, if no exact version is specified for A in podfile, then it will install the specified version in podfile.lock of A.    
If podfile specifies the version of A, then it will install the version even it is different from the one specified in podfile.lock.    
```    
pod update    
```    
When you run pod update PODNAME, CocoaPods will try to find an updated version of the pod PODNAME,     
without taking into account the version listed in Podfile.lock.     
It will update the pod to the latest version possible (as long as it matches the version restrictions in your Podfile).    
If you run pod update with no pod name, CocoaPods will update every pod listed in your Podfile to the latest version possible.    
After that, a brand new podfile.lock file is generated.    
```    
pod repo update    
```    
Updates the local clone of the spec-repo NAME.     
If NAME is omitted this will update all spec-repos located at ~/.cocoapods/repos in your home folder.    
So what is in ~/.cocoapods/repos, try to open it in your local pc, you will find that it copies the repo of     
"https://github.com/CocoaPods/Specs/tree/master/Specs", which records all the podspec file of pod library.     
In this way, cocoapods know how to download the corresponding pod library according to the information in    
~/.cocoapods/repos.      
```    
## **Make Static And Dynamic Library**    
You can do step by step following this link:    
```    
https://blog.csdn.net/fallenink/article/details/53501969    
```    
## **Build Phase & Launching of An APP**    
![avatar](https://ririripley.github.io/assets/img/app-build-and-run.png)    
### **Reference**    
https://www.jianshu.com/p/049dfb9c6274      
https://cloud.tencent.com/developer/article/1458158    
https://struggleblog.com/2021/05/19/podspec/    
http://www.samirchen.com/about-podfile-lock/    
https://guides.cocoapods.org/using/pod-install-vs-update.html    
https://stackoverflow.com/questions/20213751/what-is-the-usage-of-in-cocoapods    
https://blog.csdn.net/fallenink/article/details/53501969    
https://juejin.cn/post/6859926544408969224    
