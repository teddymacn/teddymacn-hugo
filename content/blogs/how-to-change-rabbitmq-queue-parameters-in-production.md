---
author: Teddy
title: "How to Change RabbitMQ Queue Parameters in Production?"
date: "2016-01-19"
#description: ""
summary: "Tips to change RabbitMQ Queue Parameters in Production"
tags: ["rabbitmq"]
#ShowToc: true
#TocOpen: true
---

[RabbitMQ](http://www.rabbitmq.com/) does not allow re-declaring a queue with different values of parameters such as durability, auto delete, etc. Some parameters could be configured both by queue parameter and server-side policies, but if both are set, queue parameters win. So as long as queue parameters are used, it is the same problem.

So, what if, in production environment, we do want to do the change?

**Firstly**, if your system could accept temporary downtime, then the easiest way is apparently to stop all your publisher and subscriber apps, delete the queue, and create the new one with new parameters.

**Secondly**, if possible, instead of applying incompatible parameters on existing queue, it is always recommended to add a version number to your queue (meaning creating a new queue with the same name prefix as the old one, but to add/increase the version number as part of the queue name). So that after released the config or code and restarted all the publisher and consumer apps, you only need to move all the pending messages from the old queue (if any) to the new queue, and then delete the old queue.

If you really want to change some incompatible parameters of an existing queue in production environment in a safe way (no message lost and system hang), some tricky manual steps have to be executed. RabbitMQ is so flexible, you could have many different ways to reach the same goal, so here I just try to give some examples, from which, you could gain some clues to benefit your real scenarios.

**Example 1**, if the dead letter exchange and the message TTL option of the old queue is not configured with queue parameters, meaning we could configure them with server-side policies, and also all the publisher apps only work with exchanges forwarding messages to the old queue, not publishing messages to the old queue directly:

1. We could temporarily create a new queue and a new exchange bound to it, set the temporary exchange as the dead letter exchange of the old queue and set the message TTL of the old queue to 0 with server-side policies. From now on, all the new messages published to the old queue will automatically be forwarded to the new queue.
2. As long as the old queue have no pending messages any more, you could change the upstream exchanges of the old queue to bind to the new queue instead.
3. Now you could recreate the old queue and repeat step 1 on the new queue (dead letter exchange and TTL 0) to change back to use the recreated queue.
4. Delete all temporary exchanges and queues.

**Example 2**, similar to example 1, the difference is we could not configure dead letter exchange or message TTL on old queue, and if our subscribers could tolerate receiving duplicated messages:

1. We could create a temporary queue and bind to all the upstream exchanges of the old queue, so that duplicated messages are forwarded to both the old queue and the new queue now.
2. Then we delete & recreate the old queue with the same bindings.
3. After restarted all the apps (they are still talking to the old queue), let's move all the pending messages in the new queue to the old queue with the shovel plugin (please realize that there might be some duplicated messages received by subscribers).
4. Delete the temporary queue.
