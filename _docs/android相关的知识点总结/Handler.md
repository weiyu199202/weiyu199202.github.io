---
title: Handler的源码分析
category: android重要的知识点
order: 1
---



### **第一部分：基础定义**

1. **作用：子线程与主线程通过Handler来进行通信。子线程可以通过Handler来通知主线程进行UI更新。（准确来讲指的是线程间的通信）**

2. **过程：handler通过 sendMessage 方法发送信息到消息队列 MessageQueue，Looper不断轮询MessageQueue里面的消息Message，一旦有未延时的消息，就回调用handler的dispatchMessage方法然后返回给handleMessage方法使用。**

   原理图如下:

   ![](\images\handler原理图.png)

   







