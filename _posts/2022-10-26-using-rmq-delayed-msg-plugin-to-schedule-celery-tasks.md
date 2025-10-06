---
layout: single
title: "Using RabbitMQ Delayed Message Plugin to schedule celery tasks"
date: 2022-10-26
read_time: true
comments: true
share: true
tags: [celery, rabbitmq, dlx, scheduling]
excerpt: "Learn how to use RabbitMQ Delayed Message Plugin to schedule celery tasks."
---



![Idea]({{ site.url }}/assets/images/celery-tasks-schedule-2/code-block.png)

> Note: The solutions mentioned in this article will only work if you are using Rabbitmq as the Broker.

This article will show how we can use RabbitMQ Delayed Message Plugin to *schedule celery tasks*. *(For example, how to run a celery task after ‘X’ hours or ‘X’ days)*

**RabbitMQ Delayed Message Plugin:**

The RabbitMQ delayed exchange plugin is used to implement a wait time between when a message reaches the exchange and when it is delivered to the queue. Every time a message is published, an offset in milliseconds can be specified.

[Click here](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange) to know more about the plugin.

**Idea:**

![Idea]({{ site.url }}/assets/images/celery-tasks-schedule-2/idea.png)


We can declare an exchange with the type ‘x-delayed-message’ and then publish tasks with the custom header x-delay expressing a delay time for the task in milliseconds. The message will be delivered to the respective queues after x-delay milliseconds.

*Confused about rabbitMq queues and exchanges, [Click here](https://www.quora.com/Why-does-RabbitMQ-have-both-exchanges-and-queues-Is-there-any-purpose-to-exchanges-beyond-the-multi-core-performance-improvement-of-having-separate-Erlang-processes-for-each-one/answer/Alvaro-Videla?ch=15&oid=3619117&share=d3416188&srid=hgBpk&target_type=answer) for an amazing explanation on the same (from one of the core developers of rabbitMq)*

**Issues with this approach:**

1. Celery doesn’t support the ‘RabbitMQ Delayed Message Plugin’ out of the box. So, we need to install the plugin manually in rabbit-MQ.

1. The most recent release of this plugin targets RabbitMQ 3.10.x. Series earlier than 3.9.x are out of support.

1. It will have a capacity issue if the total count of delayed messages exceeds a certain number ([https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/issues/72](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/issues/72))

1. For more, refer to [https://github.com/rabbitmq/rabbitmq-delayed-message-exchange#limitations](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange#limitations)

### Implementation steps

**0. Install the RabbitMQ Delayed Message Plugin:**

[Click here](https://stackoverflow.com/a/52819989) to know how to add the plugin to a rabbitMQ docker image and [click here](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange#installation) for the installation steps from the official GitHub page.

**1.Getting entities for delayed task delivery:**

First, we need to get the AMQP entities (*Queues, exchanges and bindings*) using which we can schedule celery tasks. For celery, the default exchange type is direct, but in our case, we need to use an exchange with the type x-delayed-message.

[x-delayed-message exchange](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange#rabbitmq-delayed-message-plugin)

A user can declare an exchange with the type `x-delayed-message` and then publish messages with the custom header `x-delay` expressing a delay time for the message in milliseconds. The message will be delivered to the respective queues after `x-delay` milliseconds.

```python
# queues.py
"""Queues module provides the queues and exchanges for the celery tasks."""

from typing import NamedTuple

from kombu import Exchange, Queue


class DelayedTaskDeliveryKit(NamedTuple):
    destination_queue: Queue
    destination_exchange: Exchange
    routing_key: str


def get_delayed_task_delivery_kit(
    destination_queue_name: str,
) -> DelayedTaskDeliveryKit:
    """
    For getting the utils using which we can schedule celery tasks.

    Publish tasks with the custom header x-delay expressing a delay time
    for the task in milliseconds.The message will be delivered to the
    respective queues after x-delay milliseconds.


    (For setting 'x-delay' header,
    see: https://stackoverflow.com/questions/35449234/how-could-i-send-a-delayed-message-in-rabbitmq-using-the-rabbitmq-delayed-messag )

    """
    # To use the delayed-messaging feature,
    # declare an exchange with the type x-delayed-message
    destination_exchange = Exchange(
        destination_queue_name,
        type="x-delayed-message",
        # x-delayed-type arguments that can be passed during exchange.declare.
        # here we have used "direct" as exchange type. That means the plugin
        # will have the same routing behavior shown by the direct exchange.
        arguments={"x-delayed-type": "direct"},
    )
    destination_queue = Queue(
        destination_queue_name,
        exchange=destination_exchange,
        routing_key=destination_queue_name,
    )
    return DelayedTaskDeliveryKit(
        destination_queue=destination_queue,
        destination_exchange=destination_exchange,
        routing_key=destination_queue_name,
    )
```

**2.Updating Celery Config:**

We need to add the ‘queue’ that we created to the celery configuration. Because for celery, the default exchange type is direct, and for the delayed task delivery using the “rabbitmq-delayed-message-exchange” plugin, we need to use the “x-delayed-message” exchange type (and hence we need to declare the same)
```python
# celery_app.py
import celery

from config import config
from queues import get_delayed_task_delivery_kit


class CeleryConfig:

    # Adding the destination queue to the task_queues
    # config.For celery, the default exchange type is
    # direct, and for delayed task delivery using the
    # "/rabbitmq-delayed-message-exchange" plugin,we
    # need to use the "x-delayed-message" exchange type
    queues_for_delayed_task_delivery = get_delayed_task_delivery_kit(
        destination_queue_name="add-tasks"
    )
    task_queues = (queues_for_delayed_task_delivery.destination_queue,)
```
**3.Defining the celery task:**

Here we are creating a sample task that adds two numbers.
```python
# tasks.py
from celery_app import app


@app.task(
    name="add",
    ignore_result=True,
    acks_late=True,
)
def add(a: int, b: int):
    print(f"{a} + {b} is {a+b}")
```

**4. Publish the task to the exchange:**

At last, we will send the celery task to the temp queue using the “[apply_async](https://docs.celeryq.dev/en/stable/reference/celery.app.task.html#celery.app.task.Task.apply_async)” method and will add the “x-delay” header specifying the delay value in milliseconds. And the task will be delivered to the respective queues after x-delay milliseconds.

```python
# __main__.py
from queues import DelayedTaskDeliveryKit, get_delayed_task_delivery_kit
from tasks import add

if __name__ == "__main__":

    delayed_task_delivery_kit: DelayedTaskDeliveryKit = get_delayed_task_delivery_kit(
        destination_queue_name="add-tasks"
    )
    add.apply_async(
        # kwargs: eqivalent to calling add(a=1,b=2)
        kwargs={"a": 1, "b": 3},
        exchange=delayed_task_delivery_kit.destination_exchange,
        routing_key=delayed_task_delivery_kit.routing_key,
        # setting 10 second (10000 millisecond) delay
        # refer: https://stackoverflow.com/a/35449483
        headers={"x-delay": 10000},
    )

```
…

I hope you have got something valuable from the article. I have added the GitHub repo that contains a working example of scheduling celery tasks using the “RabbitMQ Delayed Message Plugin”. Please feel free to log issues in the GitHub repo for suggested improvements or any bugs that I missed. All comments/criticism are appreciated.

[Github Link.](https://github.com/anandhu-gopi/RabbitMQ-Delayed-Message-Plugin-to-schedule-celery-tasks)

Happy learning 🙂.

***References*:**

* [https://docs.celeryq.dev/en/stable/](https://docs.celeryq.dev/en/stable/)

* [https://news.ycombinator.com/item?id=23259607](https://news.ycombinator.com/item?id=23259607)

* [https://stackoverflow.com/questions/48910214/celery-calling-delay-with-countdown](https://stackoverflow.com/questions/48910214/celery-calling-delay-with-countdown)

* [https://www.rabbitmq.com/dlx.html](https://www.rabbitmq.com/dlx.html)

* [https://medium.com/@hengfeng/how-to-create-a-dead-letter-queue-in-celery-rabbitmq-401b17c72cd3](https://medium.com/@hengfeng/how-to-create-a-dead-letter-queue-in-celery-rabbitmq-401b17c72cd3)

* [https://stackoverflow.com/a/46128311](https://stackoverflow.com/a/46128311)

* [https://stackoverflow.com/a/35449483](https://stackoverflow.com/a/35449483)

* [https://github.com/rabbitmq/rabbitmq-delayed-message-exchange](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange)

---

[Original post from medium](https://blog.devops.dev/how-to-schedule-celery-tasks-part-2-320b40d39945)