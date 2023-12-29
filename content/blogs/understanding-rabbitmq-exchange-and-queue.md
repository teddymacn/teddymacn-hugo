---
author: Teddy
title: "Understanding RabbitMQ Exchange and Queue"
date: "2016-02-22"
#description: ""
summary: ""
tags: ["rabbitmq"]
#ShowToc: true
#TocOpen: true
---

In [previous post](/blogs/behind-rabbitmq-exchange-types/), I compared the difference of ActiveMQ (JMS) and RabbitMQ (AMQP)'s design philosophy. RabbitMQ has its beauty of simplicity in its design with 5 exchange types. In this post, I want to clarify the understanding of RabbitMQ's exchange & queue and their relationship - binding.

In brief, exchanges are the only places where messages could be published to; while queues are the only places where messages could be consumed from. And the configurations for how messages should be routed from exchanges to queues are called bindings.

Imagine a message as a mail, then an exchange is like a post office, and a queue is kind of a mailbox in front of someone's house. Different exchange types are like different types of mails which will be delivered in different ways. For instances, if a school wants to send the same notification mail to every student's mailbox (kind of broardcast), it is exchange type - "fanout"; if a mail should only be sent to one specific student's mailbox, it is exchange type - "direct", and the key of the binding between the exchange and the queue should be the address of the student's mailbox which the post office could understand. The mail to be sent to the post office should have the same address written on it, this address is the "routing key" of the message.

In RabbitMQ, it is not possible for a publisher to send a message to a queue directly. Even when you send a message without specifying an exchange, the message is actually sent to the global default exchange, who has bindings to each queue with queue name as binding key.

Messages can only be consumed from queues, consuming from exchanges directly are not allowed. An exchange has no storage. So if you send a message to an exchange while there are no queues bound to it or no bindings match the routing key of the message, the message will be thrown away immediately.

About designing exchanges, there is [an interesting discussion](http://stackoverflow.com/questions/32220312/rabbitmq-amqp-best-practise-queue-topic-design-in-a-microservice-architecture) on StackOverflow for modeling a classic scenario. One exchange, or many? Which one is better? My opinion is similar to derick's. It really depends on your system needs. RabbitMQ's exchange types are simple, but flexible enough to support different modellings for the same scenario. There are always tradeoffs with each one. But no matter one or many exchanges you choose, I agree with Jason Lotito's comments there about routing keys and naming convention of queues: "Routing keys are elements of the message, and the queues shouldn't care about that (he means the naming of queues should not care about value of routing keys of published messages). That's what bindings are for.", "Queue names should be named after what the consumer attached to the queue will do. What is the intent of the operation of this queue.".
