---
layout: single
title: "Using DLX (dead-letter-exchange) to schedule celery tasks"
date: 2022-10-26
read_time: true
comments: true
share: true
categories: [Programming]
tags: [celery, rabbitmq, dlx, scheduling]
excerpt: "How to use RabbitMQ Dead Letter Exchanges (DLX) to schedule Celery tasks, with practical steps and caveats."
---
![code-block]({{ site.url }}/assets/images/celery-tasks-schedule/code-block.png)

> Note: The solutions mentioned in this article will only work if you are using RabbitMQ as the Broker.

This article will show how we can use DLX (dead-letter-exchange) to schedule celery tasks. (For example, How to run a celery task after `X` hours or `X` days)

## Dead Letter Exchanges (DLX)
Messages from a queue can be “dead-lettered”; that is, republished to an exchange when any of the following events occur:

- The message is negatively acknowledged by a consumer
- The message expires due to per-message TTL; or
- The message is dropped because its queue exceeded a length limit

[Click here to know more about DLX](https://www.rabbitmq.com/dlx.html).

## Idea

![Idea]({{ site.url }}/assets/images/celery-tasks-schedule/idea.png)


We can declare a temp queue and add the `x-dead-letter-exchange` and `x-dead-letter-routing-key` argument property, setting it to the name of the destination queue. And we will publish tasks to the temp queue with the TTL (time to live) header. Once the TTL is expired, celery tasks will move from the temp queue to the destination queue so that the celery worker can process them.

## Issues with this approach

- Only when expired messages reach the head of a queue will they be discarded (or dead-lettered). For example, imagine there is a task with 24-hour TTL at the head of the queue, and we have pushed a new task to the queue with a TTL of 1 sec. Until and unless the message that’s on the head (with 24-hour TTL) is expired, we will not be able to dead letter/move the messages with 1-sec TTL. (i.e. messages with 1-sec TTL have to wait for the message with 24-hour TTL to expire)
- When setting per-message TTL, expired messages can queue up behind non-expired ones until the latter is consumed or expired. Hence, resources used by such expired messages will not be freed and counted in queue statistics (e.g. the number of messages in the queue).

## Implementation steps

### 1) Getting queues
First, we will get all the required queues for “Scheduling celery Task”.

```python
# queues.py
from typing import NamedTuple

from kombu import Exchange, Queue


class QueuesForDelayedTaskDelivery(NamedTuple):
    destination_queue: Queue
    temp_queue_for_delayed_delivery: Queue


def get_queue_for_delayed_task_delivery(
    destination_queue_name: str,
) -> QueuesForDelayedTaskDelivery:
    """
    For setting up a queue using which we can delay
    the delivery of a task to the destination-queue
    for a certain time so that subscribers doesn't
    see them immediately.

    We will combine per-message-TTL and DLX (dead-letter-exchange)
    to delay task delivery.By combining these to functions we publish a
    message to a queue which will expire its message after the TTL
    and then reroute it to the destination queue and with the dead-letter
    routing key so that they end up in a queue which we consume from.

    (Per-message-TTL has to be set while invoking the task,
    see: https://stackoverflow.com/questions/26990438/how-to-set-per-message-expiration-ttl-in-celery )

    """
    EXCHANGE_TYPE: str = "direct"

    # * Like celery creates the queue and exchange with the same name,
    # * we will also create exchanges and queues with same name.
    temp_exchange_name = temp_queue_name = f"temp-{destination_queue_name}"
    destination_exchange_name = destination_queue_name

    # defining destination queue and exchange.
    destination_exchange = Exchange(
        destination_exchange_name,
        type=EXCHANGE_TYPE,
    )
    # max-retry-exceeded-queue-for-scan-doc-task-queue
    destination_queue = Queue(
        destination_queue_name,
        exchange=destination_exchange,
        routing_key=destination_queue_name,
    )

    # Steps to create temp_queue_for_delayed_delivery:
    #   1. Add the x-dead-letter-exchange argument property,
    #      and set it to the name of destination queue exchange.
    # 	2. Add the x-dead-letter-routing-key argument property,
    #      and set it to the name of the destination queue.
    #   3. Once messages from temp_queue_for_delayed_delivery, expires due
    #      to per-message TTL,they will be forwarded to the destination
    #      queue,and we will be able to achieve delayed task delivery.
    dead_letter_queue_args_for_temp_queue = {
        "x-dead-letter-exchange": destination_exchange_name,
        "x-dead-letter-routing-key": destination_queue_name,
    }

    temp_exchange_for_delayed_delivery = Exchange(
        temp_exchange_name,
        type=EXCHANGE_TYPE,
    )

    temp_queue_for_delayed_delivery = Queue(
        temp_queue_name,
        exchange=temp_exchange_for_delayed_delivery,
        routing_key=temp_queue_name,
        queue_arguments=dead_letter_queue_args_for_temp_queue,
    )
    return QueuesForDelayedTaskDelivery(
        destination_queue=destination_queue,
        temp_queue_for_delayed_delivery=temp_queue_for_delayed_delivery,
    )
```

### 2) Updating Celery Config
We need to add the `temp_queue` that we created to the celery configuration. Otherwise, moving messages to the same will fail with the `amqp.exceptions.PreconditionFailed` error.

```python
# celery_config.py
import celery

from queues import get_queue_for_delayed_task_delivery


class CeleryConfig:

    # Adding 'temp queue' to the task_queues.
    # otherwise, moving messages to the same will fail with
    # amqp.exceptions.PreconditionFailed error

    queues_for_delayed_task_delivery = get_queue_for_delayed_task_delivery(
        destination_queue_name="add-tasks"
    )
    task_queues = (queues_for_delayed_task_delivery.temp_queue_for_delayed_delivery,)
```

There are several options you can set that’ll change how celery works. Here we are using a configuration class/object for setting the configurations we needed for celery.

### 3) Defining Celery Bootstep to Declare the Queues
We need to define a worker bootstep that declares the queues for “Scheduling celery Task”. (Declaring a queue will cause it to be created if it does not already exist. The declaration will have no effect if the queue does already exist and its attributes are the same as those in the declaration)

```python
# celery_bootsteps.py
from celery import bootsteps

from queues import get_queue_for_delayed_task_delivery


class DeclareQueueAndExchangeForDelayedTaskDelivery(bootsteps.StartStopStep):
    """
    Celery Bootstep to declare the exchange and queues for
    "Scheduling celery Task".
    """

    #  'bootsteps.StartStopStep': Bootsteps is a technique to add functionality
    #                             to the workers.A bootstep is a custom class
    #                             that defines hooks to do custom actions at different
    #                             stages in the worker.

    # The bootstep we have defined, require the Pool bootstep.
    # Pool: The current process/eventlet/gevent/thread pool
    requires = {"celery.worker.components:Pool"}

    def start(self, worker):
        queues_for_delayed_task_delivery = get_queue_for_delayed_task_delivery(
            destination_queue_name="delayed-add-tasks"
        )
        with worker.app.pool.acquire() as conn:
            queues_for_delayed_task_delivery.destination_queue.bind(conn).declare()
            queues_for_delayed_task_delivery.temp_queue_for_delayed_delivery.bind(conn).declare()
```

Celery Bootsteps is a technique to add functionality to the workers. A bootstep is a custom class that defines hooks to do custom actions at different stages in the worker.

### 4) Adding bootsteps to the worker
Now we will add the celery bootsteps that we have defined earlier to the worker so that it can declare the required Queue and Exchange for scheduling the celery task.

```python
# celery_app.py
import celery

from celery_bootsteps import DeclareQueueAndExchangeForDelayedTaskDelivery
from celery_config import CeleryConfig

app = celery.Celery("My_Celery")
app.config_from_object(CeleryConfig)

# adding new bootsteps to the worker that declare
# Queue and Exchange For Delayed-Delivery-Task,
app.steps["worker"].add(DeclareQueueAndExchangeForDelayedTaskDelivery)
```

### 5) Defining the celery task
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

### 6) Sending the celery task to the temp queue
At last, we will send the celery task to the temp queue using the `apply_async` method. Once celery tasks from `temp_queue` expire due to TTL, they will be forwarded to the destination queue.

```python
# __main__.py 
from queues import QueuesForDelayedTaskDelivery, get_queue_for_delayed_task_delivery
from tasks import add

if __name__ == "__main__":

    queues_for_delayed_task_delivery: QueuesForDelayedTaskDelivery = (
        get_queue_for_delayed_task_delivery(destination_queue_name="delayed-add-tasks")
    )
    add.apply_async(  # kwargs: eqivalent to calling add(a=1,b=2)
        kwargs={"a": 1, "b": 2},
        # Moving task to the temp-queue (for delayed-task-delivery),
        # Once tasks from temp_queue_for_delayed_delivery, expires due
        # to TTL,they will be forwarded to the destination queue,
        # and we will be able to achieve delayed task delivery.
        queue=queues_for_delayed_task_delivery.temp_queue_for_delayed_delivery,
        # setting TTL of 300 sec (5 min) for task
        expiration=300,
    )
```

---


I hope you have got something valuable from the article. I have added the GitHub repo that contains a working example of scheduling celery tasks using DLX. Please feel free to log issues in the GitHub repo for suggested improvements or any bugs that I missed. All comments/criticism are appreciated.

[GitHub Link.](https://github.com/anandhu-gopi/DLX-to-schedule-celery-tasks)

Happy learning 🙂.

## References

- https://docs.celeryq.dev/en/stable/
- https://news.ycombinator.com/item?id=23259607
- https://stackoverflow.com/questions/48910214/celery-calling-delay-with-countdown
- https://www.rabbitmq.com/dlx.html
- https://medium.com/@hengfeng/how-to-create-a-dead-letter-queue-in-celery-rabbitmq-401b17c72cd3
- https://stackoverflow.com/a/46128311

---

[Original post from medium](https://medium.com/@anandhu.gopi97/how-to-schedule-celery-tasks-part-1-a1ea7b70b6a3)
