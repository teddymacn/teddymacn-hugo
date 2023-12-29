---
author: Teddy
title: "Behind RabbitMQ Exchange Types"
date: "2016-02-18"
#description: ""
summary: "The underlying philosophy behind RabbitMQ 'exchange types'"
tags: ["rabbitmq"]
#ShowToc: true
#TocOpen: true
---

Before touching RabbitMQ, I used MSMQ & ZeroMQ much in real projects before. I also played  Apache ActiveMQ together with Camel a little bit (but not in real projects) when evaluating different ESB solutions before.

Like everyone, my first glance of RabbitMQ is the [Get Started](https://www.rabbitmq.com/getstarted.html) tutorials, and the feeling of the RabbitMQ design philosophy is, it is quite like the design philosophy of ZeroMQ - simple & consistent interface and super flexible configuration. Among all of the configuration options, the most interesting part is the "exchange types". You might want to say that the concept of "exchange types" is from the [AMQP](https://www.rabbitmq.com/amqp-0-9-1-reference.html) protocol, not invented by RabbitMQ. Yes, you are right. But no doubt that RabbitMQ is one of the most popular AMQP implementations.

The 6 examples in the [Get Started](https://www.rabbitmq.com/getstarted.html) tutorials show the flexibility of RabbitMQ with the consistent and simple Publish & Receive interface. And I believe you could already imagine many more usages of them to match many other common scenarios.

But, what's the underlying philosophy behind "exchange types"? In a word, it is all about implementing integration patterns in a manner of simple, stupid.

If you ever played Apache ActiveMQ and Camel, you must be familar with the [Enterprise Integration Patterns](http://www.enterpriseintegrationpatterns.com). The examples of Apache Camel even exactly match all of the integration patterns. The design philosophy of ActiveMQ is, it provides the basic queue and pub/sub ability, and, together with Camel, to give ultimate flexibility for implementing all the integration patterns.

The design philosophy of ZeroMQ and RabbitMQ is quite different. Ultimate flexibility is cool, but performance and simplicity are also important. ZeroMQ chooses ultimate performance and simplicity rather than more integration features, while RabbitMQ is kind of in between of ActiveMQ and ZeroMQ. Its interface is as simple as ZeroMQ; the performance is not as super as ZeroMQ, but good enough; and it provides minimal but elegant configuration options to support most of the common integration patterns. One of the essences of its design is just the "exchange types", which abstracts most common message routing & consuming scenarios with only 5 easy-to-understand types: default (no routing), direct, fanout (broadcast), topic and header.

Modeling & programming is the art of abstracting complexity. A real elegant design must be simple, stupid! That's what behind the idea of "exchange types".