---
layout: post
title: Kafka协议详解之Consumer Group
categories: kafka
tags: kafka cousumer group
---

在之前我们已经了解了Kafka协议的格式，并且自己动手实现了几个协议的封包和解包操作。现在，我们应该是处于这样一个状态：手有利器，感觉只要能搞懂kafka的每个协议的作用，那么对kafka系统的使用就会得心应手。

今天，我们主要来研究一下cousumer相关的协议。

## 简单的pull模式

Kafka集群作为消息队列，提供的消费逻辑非常简单：

集群将所有的msg分成若干个部分，每个部分称为1个partition。每个partition内部的消息都是保证有序存放的，并



> Written with [StackEdit](https://stackedit.io/).