---
layout: single
title: "Let's Make it Async; Making asynchronous HTTP requests with \"aiohttp\" in Python"
date: 2022-05-15
read_time: true
comments: true
share: true
tags: [python, async, aiohttp, programming]
categories: [Programming]
excerpt: "Learn how to make asynchronous HTTP requests using aiohttp in Python and see the performance improvements."
---
![Fast](../assets/images/async-python/image_1.png)


## What is async and how it's going to help us?

Let's imagine that you went to a restaurant for lunch, and in front of you there is a long queue of 20–30 people waiting to get their lunch and, unfortunately, there is only one waiter in that restaurant to take orders, but there are many chefs in the kitchen who are very good at working parallelly. And there are two ways you can get your lunch (for the sake of simplicity, let's keep it two)

### Synchronous way:
Each person will approach the waiter one by one, i.e. the first person in the line will go to the waiter he will give his order then he will wait for the waiter to bring food, once he got his food he will go and sit someplace, and after that second person will give his order then wait for the waiter to bring food then the third person will go and so on.No one can order while a person is waiting for his food. That is essentially, a person is blocking the waiter from taking orders from others until he gets what he ordered, And it will take an awful lot of time for you to get your lunch.

### Async way or Asynchronous way:

In an async way each person will go to the waiter he will give his order and sit someplace, after that the second person will go and give his order, then the third person will go and so on. That is, as soon as a person gives the order, he will move from the queue and wait somewhere else, he will not block the waiter from taking orders from others, And once the food is ready, the waiter can give it to the person who has ordered it.

i.e. as soon as the waiter got an order, he will inform the kitchen and come back fast to take orders from the next person. If we follow this approach, we can complete the entire process very fast as there are many chefs in the kitchen, the chefs can cook food in the background, and the waiter can respond to the people faster.

For a more detailed explanation about Async IO [click here](https://realpython.com/async-io-python/).

## Lunch example conclusion:

In the scenario of taking lunch synchronously, there is a lot of waiting. And in an async way, things are getting completed fast as there is no blocking or waiting. Now I will try to connect this to another example below (so that it will be clear how this async will help us, bear with me 🤓).

And before going to the example, we need to know some terminologies.

### Event Loop:

Think of the event loop as a supervisor who monitors some workers. And he will pick one of the workers and will say, "Hey! you go and do whatever you want to do, but do let me know if you are sitting idle. ". And the moment a worker says that he is sitting idle event loop will give chance to some other worker who is ready to work and will wait to hear back from that worker, and it will repeat the same process again and again till all the workers have completed their jobs.

### Coroutine:
Think of Coroutine as a worker, and it needs to do some work and while working if's waiting for something it can notify the supervisor that it is idle and then the supervisor can give control to some other worker.

### Coroutine + Event Loop:

Think of coroutines as python functions that can pause their execution while waiting for something, and they can also notify the event loop that they are sitting idle. And think of an event loop as a while loop that is constantly taking feedback from these coroutines to see if anyone is sitting idle; it can also notify an idle coroutine if whatever that coroutine is waiting on becomes available.

i.e. coroutines are specialized functions that can pause their execution, and they are being continuously monitored by the event loop so that none of them will sit idle and block some other coroutine.

The simplest way to define a coroutine in python using the async/await:

```python
# example: defining a coroutine
async def my_coroutine():
    return "Hello, World!"
```

The simplest way to get the event loop, run tasks until they are marked as complete, and then close the event loop using asyncio.run() :

```python
# example: getting the event loop and running coroutine
import asyncio

async def main():
    result = await my_coroutine()
    print(result)

asyncio.run(main())
```

**coroutine + event loop:**

```python
# example: defining coroutine and running it with the event loop.
import asyncio

async def my_coroutine():
    return "Hello, World!"

async def main():
    result = await my_coroutine()
    print(result)

asyncio.run(main())
```

## Async with: The Asynchronous context manager

An asynchronous context manager can suspend execution in its enter and exit methods ( `__aenter__` and `__aexit__` ). For using them, we can use the "async with" statement.

```python
# example: Using asynchronous context manager
async with aiohttp.ClientSession() as session:
    async with session.get('https://api.example.com') as response:
        data = await response.json()
```

For those who don't know about context managers, they are just an easy and pythonic way to manage resources( For allocating and releasing resources e.g: opening a file and closing it after use ). To know more about context manager [click here](https://realpython.com/python-with-statement/).

For a more detailed explanation about defining coroutines in python [click here](https://realpython.com/async-io-python/) and to know more about event loops in python [click here](https://realpython.com/async-io-python/).

To know about the rules of asyncio [click here](https://docs.python.org/3/library/asyncio.html).

## An example of how to make Async HTTP requests using aiohttp.

For this example, I will pick up a simple use case.

**use case:**
We have given a bunch of names as input, and from that, we need to take each name, make a call to the genderize API (A simple API to predict the gender of a person given their name) and determine the gender

**Input:**
```python
names = ["Alice", "Bob", "Charlie", "Diana", "Eve"]
```

**output:**
```python
[("Alice", "female"), ("Bob", "male"), ("Charlie", "male"), ("Diana", "female"), ("Eve", "female")]
```

### Synchronous version:

```python
import requests
import time

def predict_gender(name):
    url = f"https://api.genderize.io?name={name}"
    response = requests.get(url)
    data = response.json()
    return name, data['gender']

def main():
    names = ["Alice", "Bob", "Charlie", "Diana", "Eve"]
    start_time = time.time()
    
    results = []
    for name in names:
        result = predict_gender(name)
        results.append(result)
    
    end_time = time.time()
    print(f"Time taken: {end_time - start_time:.2f} seconds")
    print(results)

if __name__ == "__main__":
    main()
```

The synchronous version of the code is pretty straightforward. For each name from the input, we will call the predict_gender function, and predict_gender will call the genderize API and will return a tuple containing both name and corresponding gender once the genderize API respond.

### Asynchronous version:

```python
import aiohttp
import asyncio
import time

async def predict_gender(name, session):
    url = f"https://api.genderize.io?name={name}"
    async with session.get(url) as response:
        data = await response.json()
        return name, data['gender']

async def main():
    names = ["Alice", "Bob", "Charlie", "Diana", "Eve"]
    start_time = time.time()
    
    async with aiohttp.ClientSession(raise_for_status=True) as session:
        tasks = [predict_gender(name, session) for name in names]
        results = []
        
        for coro in asyncio.as_completed(tasks):
            result = await coro
            results.append(result)
    
    end_time = time.time()
    print(f"Time taken: {end_time - start_time:.2f} seconds")
    print(results)

if __name__ == "__main__":
    asyncio.run(main())
```

Compared to the synchronous version looks like there is a lot of stuff happening here, so let's dive into it.

**"async def" vs "def"?:**

Usually, for defining a function, we will use "def", but here we are using "async def" as we are dealing with coroutines (think of them like special python functions that can be stopped and resumed while being run), and for defining them, we need to use "async def".

**"await"?**

The word 'await' means wait for (an event). It's a way to notify the event loop that we are waiting for something, and you can suspend the execution until whatever I am waiting for is ready. To call a coroutine function, we must await it to get its results. And the benefit of doing so is that whoever is calling await can pause and pass the control to another coroutine function that's eagerly waiting to do something.

**"async with" instead of "with":**

We will use "with" for context managers and "async with" for asynchronous context managers. So it's kind of like you add the "async" prefix if you are dealing with asynchronous stuff, like defining a coroutine or using an asynchronous context manage.

**async def predict_gender coroutine:**

"predict_gender" is the main coroutine that our program uses. It takes the name and session object as input. And using the session object, the method will make a get call to the genderize API. It makes the request and awaits the response. Using "session.get", we will get the response object, and to get the response body as JSON "resp.json()" method needs to be awaited. (JSON() method of aiohttp.ClientResponse object is a coroutine, and, as I mentioned earlier, we need to use await to call a coroutine and get its result.).

Check out this [stack overflow answer](https://stackoverflow.com/questions/44049719/how-does-asyncio-as-completed-work) on why we need to await the resp.json method after the session.get call

**async def main:**

The above coroutine serves as the main entry point into the script. It uses a single session, and for each name, a task will get created for gender prediction. And, you can see we are calling the "predict_gender" coroutine directly without await, and by doing so, we will get an awaitable coroutine object, and we are appending these objects to a list and calling "asyncio.as_completed" with this list of tasks ( awaitable coroutine objects), this function returns an iterator that yields tasks as they finish.

To put it simply, we will create a bunch of tasks for gender prediction and give it to " asyncio.as_completed", and it will return the tasks ( that you have to await ) as they are completed, in the order of completion.

And while creating the session object, we are setting "raise_for_status" as "True" so that it will raise "aiohttp.ClientResponseError" if the response status is 400 or higher (Basically, throw an error if the response status is 400 or higher), By doing this we can do stuff like catching response errors if the server responds with a non-successful status code and retrying the failed task (it's always better to have some a retry mechanism to handle transient errors)

[Click here](https://docs.aiohttp.org/en/stable/client_quickstart.html) to know more about handling transient errors.

Also, check out this [post about retrying asynchronous functions](https://quentin.pradet.me/blog/how-do-you-rate-limit-calls-with-aiohttp.html).

**asyncio.run:**

Now the last part, "asyncio.run". It will get the event loop, run tasks until they are marked as complete, and then close the event loop. (Think of the event loop as a while loop that is constantly taking feedback from coroutines/tasks to see if anyone is sitting idle; it can also notify an idle coroutine if whatever that coroutine is waiting on becomes available.)

## Performance improvement with "async":


The synchronous version took ≈ 120 sec for 100 names (for predicting the gender of 100 names). And for the same 100 names, the asynchronous version took ≈ 2 sec. Approximately 98% reduction in time.

![Performance chart]({{ site.url }}/assets/images/async-python/image_2.png)

**Note:** The performance of these scripts depends on your network also (My network is not that great, so maybe you will see a variation in speed if you are testing these scripts in your local ;) )

Now, more than this what else do we need to opt for asyncio and aiohttp. Let's make things async whenever we can and save time. Let's make it async.

Happy learning 🙂.

I hope you find this article helpful. Please let me know your suggestions and feedback in the comment section.

## Sources:

- [https://realpython.com/async-io-python/](https://realpython.com/async-io-python/)
- [https://docs.aiohttp.org/en/stable/client_quickstart.html](https://docs.aiohttp.org/en/stable/client_quickstart.html)
- [https://quentin.pradet.me/blog/how-do-you-rate-limit-calls-with-aiohttp.html](https://quentin.pradet.me/blog/how-do-you-rate-limit-calls-with-aiohttp.html) (Very nice blog on rate-limiting HTTP calls using the token-bucket algorithm.)
- [https://stackoverflow.com/questions/44049719/how-does-asyncio-as-completed-work](https://stackoverflow.com/questions/44049719/how-does-asyncio-as-completed-work)
- [https://stackoverflow.com/questions/42231161/asyncio-gather-vs-asyncio-wait](https://stackoverflow.com/questions/42231161/asyncio-gather-vs-asyncio-wait)
- Many more StackOverflow answers have helped me learn the concepts of async programming in python. ( I was not able to keep track of all those, Sorry (-_-) )

---


[Original post from medium](https://medium.com/@anandhu.gopi97/lets-make-it-async-making-asynchronous-http-requests-with-aiohttp-in-python-106eb5e6d048)
