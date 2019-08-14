# AbstractQueuedSynchronizer原理

## Intro

AbstractQueuedSynchronizer提供了一个FIFO队列，可以看做是一个可以用来实现锁以及其他需要同步功能的框架。这里简称该类为AQS。AQS的使用依靠继承来完成，子类通过继承自AQS并实现所需的方法来管理同步状态。例如ReentrantLock，CountDownLatch等。

下面基于JDK1.8来分析AQS实现原理

![aqs](/Users/huangdachao/workspace/java-skills-book/201905/img/aqs.png)

## CLH队列锁

CLH锁即Craig, Landin, and Hagersten (CLH) locks。CLH锁是一个基于链表实现的自旋锁，能确保无饥饿性，提供先来先服务的公平性。CLH锁具有可扩展、高性能、公平等特性。申请线程仅仅在本地变量上自旋，它不断轮询前驱的状态，假设发现前驱释放了锁就结束自旋。



## 相关文档

1. [A hierarchical CLH Queue Lock](http://www.cs.tau.ac.il/~shanir/nir-pubs-web/Papers/CLH.pdf)

