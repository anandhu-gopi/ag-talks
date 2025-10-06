---
layout: single
title: "Is WASM the New Docker? 🤔"
date: 2024-11-30
author_profile: true
read_time: true
comments: true
share: true
categories: [Programming]
tags: [wasm, wasi, docker, webassembly, containers, cross-platform]
excerpt: "Exploring WebAssembly (WASM) and WASI to see if they could replace Docker for cross-platform application deployment."
---

![WASM vs Docker]({{ site.baseurl }}/assets/images/wasm-new-docker/wasm-new-docker.jpeg)

A few days ago, I came across an article that used WASM to build and run apps. So, I decided to try it out and see how we can build apps with WASM and test whether it's the new Docker or not.

Before going into details of what I did, I will be explaining:
1. WASM
2. WASI 
3. And WASM runtime

## 1️⃣ WASM

Historically, browser engines have supported JavaScript, meaning you couldn't write a program in Go and have it run inside a browser. But why run Go, Rust, or C# in a browser when JavaScript is fine? The answer comes with running heavy applications, such as games/photo-editor., in your browser. This is where WASM shines, by performing comparably to running native apps ( builds targeting specific os/platforms, like the games you download and run on your system).

You can write code in a language like C, C++, Go, or Rust, and then compile it to WASM (WebAssembly). The browser engine can run WASM code, so it's no longer limited to just JavaScript.

To anyone interested: a list of WASM Games that you can run in your browser: [https://lnkd.in/gypZnRzZ](https://lnkd.in/gypZnRzZ)

## 2️⃣ WASI: The API That Sets the Standards

As long as the browser engine can run WASM, you can write your code, compile it to WASM, and run it without worrying whether the browser is running on Linux, Windows, or macOS, or whether it's on an x86 or ARM architecture.

This opens up the possibility of compiling your code to WASM, and since browser engines can run it, you can replace the browser engine with a generic tool that can run WASM, making your build cross-platform (build once and run anywhere).

But to ensure cross-platform compatibility, there needs to be a standard interface, which is where WASI (WebAssembly System Interface) comes in.

WASI is a set of APIs that allows WASM to interact with system resources across various platforms. Think of it like a set of contracts that ensure consistency when running WASM code on different systems.

## 3️⃣ WASM Runtime: The Guy Who Takes Things Beyond the Browser

Now, while browser engines can run WASM, to run WASM outside the browser, you need a runtime. A WASM runtime is what runs your WASM code.

Multiple runtimes are available, and the one I used was Wasmtime.

## To Sum it up: 🤠 

You write code in Go, Rust, etc., compile it to WebAssembly, and with a WASM runtime, you can run it on any platform. It will be able to access the host's file system, network, and more, thanks to WASI, which provides a standard set of APIs for cross-platform execution.

## WASM in Action 🎬: 

I wrote a simple Go program that prints a statement to the console. Then, I compiled it to WASM. Now, I built the .wasm file on my Mac, copied it to a Linux machine, installed Wasmtime there, and ran the program — and it worked perfectly. It's as easy as that.

As for whether WASM and WASI will replace Docker, we'll have to wait and see, Perhaps they will even work together: [https://lnkd.in/g_64s-xU](https://lnkd.in/g_64s-xU)

## References 

1. [https://lnkd.in/gEzVesQA](https://lnkd.in/gEzVesQA)


[Original post from Linkedin](https://www.linkedin.com/posts/anandhu-gopi-691b35144_is-wasm-the-new-docker-a-few-days-ago-activity-7268649431535611904-MgnC)
