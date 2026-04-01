---
layout: post
title: "Remembering Concurrency - Lesson 1: Processes and Threads"
date: 2026-03-29 10:00:00 +0000
categories: [Concurrency]
tags: [Concurrency]
description: >
  Refreshing my knowledge of concurrency from first principles in a series of lessons. 
  Lesson 1 - Processes and Threads.
---
[Source code on GitHub → mcawalsh/concurrency-and-parallelism](https://github.com/mcawalsh/concurrency-and-parallelism/tree/main/Lesson1.Threads)


## Processes

So what are processes? Every time you run a program, the OS creates a process. A process is an isolated container that holds...

- The program's code
- A block of memory exclusive to the program
- Its own file handles, network connections, resources
- At least one thread.

Every application you run is a process. When you create a new .Net application and run it, that's a process.

Processes are isolated by design. This is so that if it crashes it doesn't take down other processes with it. They can't accidentally read each other's memory.

## Threads

Threads are what execute code. Threads run on a CPU core. A process must have at least one thread (the main thread). A process can have many threads but without a thread no code can run.

### Heap
A process can have many threads. They all share the same `heap` - where objects created with `new` are stored. A thread can read or write any object on the heap.

Since threads share the heap, two threads modifying the same object simultaneously can corrupt it. This is the root cause of most concurrency bugs.

### Stack
Each thread also has it's own memory space which would be the threads stack memory which is independent of other threads and cannot be modified outside the thread.

## CPU Cores & OS Schedules
How the OS schedules threads on cores

A CPU core can execute one thread at a time. If you have 4 cores you can run 4 threads simultaneously. This is parallelism.

Modern machines run hundreds of threads. The OS handles this with a scheduler that rapidly switches which thread runs on each core. Each thread gets a tiny slice of time before being paused and then another thread gets a turn.

This is how we run tasks concurrently. We create multiple threads and the OS interleaves their execution so rapidly it appears simultaneous but on a single core only one is ever truly at at any one time.

Switching happens so fast it feels simultaneous but its not. The switching has  acost - `context switching`.

When the OS swaps threads it has to
1. save the current thread's state (registers, stack pointer, program counter)
2. restore the next thread's state

This takes time and cache memory.

## Threading Costs

Threads are expensive. They take up memory and time.

- Memory - each thread gets its own stack (~ 1MB by default in .Net)  
- Time - creating an OS thread takes microseconds  
- Context switching - save the current threads state, then load the new threads state - This time adds up.

The more threads you have the more context switching needs to happen and the bigger build up there is. Consider you have 100 threads running on a single core. The context switching scheduling all threads grows quickly and the latency grows.

## Demo

- Link: [https://github.com/mcawalsh/concurrency-and-parallelism/tree/main/Lesson1.Threads](https://github.com/mcawalsh/concurrency-and-parallelism/tree/main/Lesson1.Threads)

```csharp
[DllImport("kernel32.dll")]
static extern int GetCurrentProcessorNumber();

var threads = new List<Thread>();

for (int i = 0; i < 5; i++)
{
    var threadNumber = i + 1;
    var thread = new Thread(() =>
    {
       var id = Thread.CurrentThread.ManagedThreadId;
       Console.WriteLine($"Thread {threadNumber} starting - OS thread ID: {id} - Processor Number: {GetCurrentProcessorNumber()}");
       Thread.Sleep(Random.Shared.Next(500, 2000));
       Console.WriteLine($"Thread {threadNumber} finishing - OS thread ID: {id} - Processor Number: {GetCurrentProcessorNumber()}"); 
    });

    threads.Add(thread);
}

Console.WriteLine("Starting all threads...");
Console.WriteLine();

foreach (var thread in threads)
{
    thread.Start();
}

foreach(var thread in threads)
{
    thread.Join(); // wait for each to finish
}
```

1. This demo creates 5 threads.
2. Each .Net Thread object represents an OS thread.
3. When we start the threads they start actively working and writing to the cammand line.
4. The output is the `threadNumber` that we assigned, the `id` of the OS thread, and the processor number that the current thread is running on.
5. `thread.Join()` causes the application to wait for all threads to complete before writing the final WriteLine statement.

You can see in the output the threads start, sleep, and finish. What we also see is that the threads can start and finish on different cores.

```bash
Process ID : 28580
Process name : Lesson1.Threads
Main thread : 2
CPU cores : 12
Starting all threads...

Thread 1 starting - OS thread ID: 4 - Processor Number: 9
Thread 5 starting - OS thread ID: 8 - Processor Number: 2
Thread 4 starting - OS thread ID: 7 - Processor Number: 2
Thread 2 starting - OS thread ID: 5 - Processor Number: 11
Thread 3 starting - OS thread ID: 6 - Processor Number: 8
Thread 1 finishing - OS thread ID: 4 - Processor Number: 10
Thread 2 finishing - OS thread ID: 5 - Processor Number: 4
Thread 4 finishing - OS thread ID: 7 - Processor Number: 4
Thread 3 finishing - OS thread ID: 6 - Processor Number: 0
Thread 5 finishing - OS thread ID: 8 - Processor Number: 6
All threads finished.
```

In Lesson 2 we look at how the `ThreadPool` solves the cost problem by reusing threads rather than creating new ones.