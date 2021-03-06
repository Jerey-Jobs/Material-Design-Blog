---
title: 闲散整理，带你走进Android短信源码 
subtitle:   "事实上，市场上的大多短信应用都是基于Google原生改的"
grammar_cjkRuby: true
header-img: "img/post-bg-os-metro.jpg"
catalog: true
layout:  post
tags:
    - Android
    - Google源码
categories: Android
date: 2017-02-02
---

过年在家无聊，发现以前写的短信App分析文章，以及ppt，想着就整理出来。好了，废话不多说，开使正文

----------

## 简介

这是Google的Mms源码，不过在15年中旬就停止维护了，后期的维护交给了各大开发商。<br>

[这里是我fork的工程地址 ps:加了注释的](https://github.com/Jerey-Jobs/Mms_google/)

[这里是谷歌工程地址](https://github.com/android/platform_packages_apps_mms)

其实Mms很复杂的，谷歌官方的是简单功能的短信，不过就基础功能版本的短信也够撸了，比如一个彩信附件的显示，其继承关系图就足够复杂，而且彩信还是是smil语言，还有什么pdb，够人好好看一番。

我们可以通过这个工程，看到谷歌人员是怎么写代码的，当然由于这个app的版本迭代很多了，没有重构，没有采用Google推荐的MVP模式。

更别提Rx那些了，这个工程就是4大组件开干。

## 初识架构

![Mms模块构成图](/img/mms/mmsbasemodel.png)

由于懒，简答说，整个短信主要是由2个主要Activity，加2个远程Service，2个本地Service，4个Provider以及4个Receiver组成。其他的一些小功能的就不说了。

- 所谓2个主要Activity，一为短信联系人界面，即一个listView，外加一些增删功能，另一个为短信编辑界面
- 所谓2个远程Service，即为AIDL且进程号不同，跑在系统中的负责短信发送接收，彩信发送接收的服务
- 所谓2个本地Service，即为本应用后台负责发送短信与彩信的服务，实为调用系统服务发送接收，不过本地有部分逻辑需要处理。
- 4个Provider，为：SuggestionsProvider，用于搜索建议，与Search配合，<br>
                  SmsProvider 管理短信数据表<br>
                  MmsProvider 管理彩信数据库<br>
                  MmsSmsProvider 管理短彩信共同数据库<br>
- 4个Receiver，为：SmsReceiver 短信到来
                  PushReceiver 推送，Class0
                  SmsSystemEventReceiver  系统启动
                  PrivilegedSmsReceiver 彩信接收
                  

## 基础知识

**我们来梳理一下基础概念，一些重要的对象。**

Thread对话---是指用户与某个联系人或某几个联系人之间的一系列信息交互。在Mms中，用Thread Id来标识和管理对话，Thread Id也即在数据库表threads中的_id。可能用Conversation是更易于理解。但是Thread本身就有对话的意思，某些论坛中的一个帖子在英语里就叫Thread。Thread的词典释义是：”因特网上关于一个题目一连串的信息 (计算机用语)”，所以这里用Thread，也是比较恰当的。

Conversation--是用来管理Thread对话的，Conversation是一个Thread对话的抽象出来数据结构，它能够，从数据 库中查询，删除一个对话中的消息等，每一个Conversation有一个唯一的Thread Id。但是它也负责一些所有对话的管理，比如查询所有的对话，删除所有的对话等（这个应该是设计上面的问题）。
事实上，Conversation更多时候是充当前前对话的角色，比如在新新建信息时，编辑信息时，或是查看某个对话时，都会有一个 Conversation对象存在在，以代表当前信息所处的对话。它是一个近似单键，都是通过Conversation的静态方法来获得 Conversation对象，有一些其他的方法也是静态的。

ConversationList--负责显示和编辑所有的对话，以列表形式显示所有的Thread，每一项代表一个Thread，通常也会显示这个Thread的状态，如有无草稿，信息发送/接收是否成功等。

Message--消息，泛指短信SMS和彩信MMS。因为不再区分短信和彩信，在对话列表，草稿管理和信息列表中它们都是一样的，都是信息。 Message的数据结构是MessageItem，它是一个纯数据结构，里面存储着关于一个信息的所有数据，还有MessageListitem，它是 一个View，专门用于在消息列表中显示一个信息，里面的数据都是从MessageItem获取。它们统一都被 ComposeMessageActivity，MessageListAdapter和MessageListView来管理。

WorkingMessage--当前消息，它是专门用于代表当前正在创建和编辑的信息的数据结构无论是短信还是彩信，在创建和编辑的时候都放在一个WorkingMessage对象里面。这个对象也负责信息的发送，存储和存储为草稿。

Slideshow--在Mms应用里面，彩信是以Slideshow幻灯片的形式来展示的。一个彩信可以有多张幻灯片，每张幻灯片上面可以有图 片，文字，音频和视频，可以设置每张幻灯片的浏览时长，布局等，这里的幻灯片与Office中的PowerPoint有几分类似。幻灯片的数目限制是以彩 信允许的附件大小为上限，这个也与每张幻灯片上面的媒体大小有关。可以这样讲MMS就是以幻灯片形式存在的，创建的时候是一张幻灯片一张幻灯片的编辑，收 到的彩信或编辑完后，就可以一张张的放映浏览幻灯片。
需要注意的是以幻灯片方式显示彩信仅是应用程序层的处理方式，不同的信息应用程序会以不同的方式处理彩信，实际的彩信的数据是以标准的Pdu方式进行发送和接收，是应用程序在发送前把幻灯片转化成为Pdu，并在接收后把Pdu转化成为自己可识别的幻灯片。

Recipient接受人，这里是指信息的接收者，要么是一个陌生的电话号码，要么是一个陌生的电子邮件地址（彩信时），要么就是手机联系人数据库 中的联系人。彩信和短信对接收人的数量都有限制，这个也是在Mms的Settings时面可以更改的。每一条信息要想发送成功，必须保证接收人是一个合法 的联系人，合法也是不同的手机有不同的定义，但通常来讲，要么与联系人数据库中的某个联系人匹配，要么是一个电话号码，要么是一个电子邮件地址，其他情况 则视为不合法，对于有不合法接收人的信息，不会进行发送。管理联系人的数据结构是Contact和ContactList，其中ContactList是 一个以Contact为元素的ArrayList。Contact不但存储有联系人的一些信息，如名字，电话号码等，它还能与联系人数据库进行同步，也就 是它能保证它是一个合法的联络人，并在数据库中存在。在信息发送前会先进行一次联系人同步，以保证已有的联系人是正确的。

---

### UI初窥

![短信AppUI](/img/mms/mms_view.jpg)

---

### 以下分析大致分为：

 - UI分析
 - 短信发送、接收
 - 彩信
 - 数据库


###  UI主要组成

- 我们看会话列表数据刷新图

![会话列表数据刷新](/img/mms/conversation_refersh.png)

- 编辑界面的类图

![编辑界面类图](/img/mms/write_view_leitu.png)
   
### 短信发送、接收 

- 短信发送流程图
- 
![短信发送流程图](/img/mms/sms_send_liucheng.png)

- 短信发送类图

![短信发送类图](/img/mms/sms_send_class.png)

- 短信接收流程图

![短信接收流程图](/img/mms/sms_receive_l.png)

### 彩信 

MMS为Multimedia Messaging Service的缩写，中文译为多媒体短信服务，通过网络来传输数据

MMS发送和接收: 手机终端合成多媒体消息后，可以向网内的所有合法用户发送多媒体消息。不直接投递给接收方。先发送给一个中间服务器 (MMSC)。彩信网关暂时保存彩信，然后通过Push 协议给接收方发送一个Push 通知，接收方收到通知后再从MMSC 上获取彩信内容。

与MMSC交互的数据为PDU数据

Google内置包里为我们提供了一系列操作PDU的类（com.google.android.mms）

- PduPersister  用于管理PDU存储
- PduParser	用于解析PDU
- PduComposer	用于生成PDU

![彩信发送流程图](/img/mms/mms_send_google.png)

![彩信接收流程图](/img/mms/mms_receive_l.png)

![彩信类图](/img/mms/mms_class.png)


**彩信不自动下载的情况**

> 在会话界面的MessageListItem中会显示一个download按钮，当用户点击该按钮<br>
> -->mContext.startService(intent);<br>
> -->TransactionService中，处理Action为 RETRIEVE_TRANSACTION的请求<br>
> --> transaction = new RetrieveTransaction();<br>
> -->transaction.process();<br>
> -->byte[] resp = getPdu(mContentLocation);<br>
> -->PduParser解析数据成pdu<br>
> -->isDuplicateMessage 判断是否是重复的短信<br>
> -->PduPersister 再将其存储<br>
> -->sendAcknowledgeInd(retrieveConf); 再发送Ack给MMSC<br>
> -->notifyObservers();   通知service ，状态改变<br>



 #### Mms数据库，在短信应用程序中占有很重要的地位

> 1.负责数据的存储 	短信，彩信，对话列表都存储在数据库中<br>
> 
> 2.负责大量的通信 	先待发短信存储到数据库中，发送服务将待短信从数据库取出<br>
> 
> 3.通过ContentProvider，间接的肩负起通知界面数据刷新工作  	getContext().getContentResolver().notifyChange() 	通知观察者去刷新数据<br>


> 
> threads表：在ConversationList.Java中显示的当前短信 <br>
> sms表：短信内容 <br>
> pdu表： 彩信内容<br>
> part表：（存储彩信内容（文本、音乐、图象）文件名 <br>
> pending_msgs：存储待发送的短信与彩信 <br>
> drm：用于彩信权限管理<br>
> words：用于存储关键字，搜索时用<br>
> SmsProvider用于短信相关数据的存取 MmsProvider用于彩信相关数据的存取<br>
> MmsSmsProvider则用于短彩信通用数据的存取，如会话信息、接收者、草稿（公共属性）等



 ----------
### 谢谢大家阅读，如有帮助，来个喜欢或者关注吧！

 ----------
 本文作者：Anderson/Jerey_Jobs 

 博客地址   ： [夏敏的博客/Anderson大码渣/Jerey_Jobs][1] <br>
 简书地址   :  [Anderson大码渣][2] <br>
 CSDN地址   :  [Jerey_Jobs的专栏][3] <br>
 github地址 :  [Jerey_Jobs][4]
 


  [1]: http://jerey.cn/
  [2]: http://www.jianshu.com/users/016a5ba708a0/latest_articles
  [3]: http://blog.csdn.net/jerey_jobs
  [4]: https://github.com/Jerey-Jobs
