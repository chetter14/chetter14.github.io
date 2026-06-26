---
layout: post
title: Order book. Part 1.
---

I had a desire to write a highload project in C++ with some application in high-frequency trading, finance, and related fields. My goal is not exactly about diving into the domain of a problem, but rather carry out a millions of operations per second and make a *solid* C++ project (with tests, scripts, benchmarks).

I came to the idea of writing an **order book** - a structure that keeps track of all the buy and sell orders (*bids* and *asks*, respectively). It can be found on stock exchanges on UI. As for example, buy orders on the left (marked green) and sell orders on the right (marked red).

An image of order book I found on the Internet:
![](../images/order-book.jpg)

I began thinking about the implementation part of order book. The first assumption I've made is that **prices are not floating-point numbers**, they are just **integers**. This way I can avoid problems with math and special handling of numbers:
```
using Price = unsigned int;
```

How am I going to store bids and asks efficiently, so that it doesn't consume much memory, doesn't degrade the performance, and still presents a solution for the problem? I came to the idea of using an array, where **each index of this array is a price**. 

A bunch of advantages from having an array here:  
1) as I understand, most operations are done on the intersection of buy and sell prices. In array, these cells are going to be near each other. Such a placement in memory is *cache-friendly*.
2) *O(1)* access complexity.

Even though the array seems good here in a *performance* sense, what about memory? Will it be reallocated every time a new higher price arrives? I thought it would be a problem, so I made the second assumption here - **price is going to be limited at the top with some value**. I think that it's a huge oversimplification from me, but again, I'm not trying to make a real-world-like order book that can be used in real projects, at least for now. Maybe in future I'm going to implement this part "correct".
```
constexpr std::size_t MAX_PRICE_VALUE = 9999U, MIN_PRICE_VALUE = 1U;
```

So, the array is going to be *preallocated* and *stay the same size* during the whole execution of program. But what does this array store exactly? 