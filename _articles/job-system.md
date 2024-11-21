---
title: I Bought 12 cores
date: 2024-10-28
draft: true
layout: post
---


# Why ??
I have a scene to load but i don't want the loading to block the whole program. 
How can I do that?

- Callbacks + Polling
- Async
- Multithreading 
- All of the above

First of all let's make sure of one thing: I don't need multithreading for my system to do the job.
Like not at all. 
But should that stop me?


What parallelisation can bring:

## Better IO ! (read from file, write to file, wait for GPU completion)
-> More like a callback system
-> Tasks are long lived but short but has short CPU execution time 
-> May be event driven (io_uring)
-> TODO: try io_uring vs read vs poll vs idk vs mmaped file
    -> Expected result:
        - mmaped file is the same / faster than an async read but there may be latency issues when accessing memory
        + May be inconsistent
        - It would be better to use an io_uring 

See: [DMA](https://en.wikipedia.org/wiki/Direct_memory_access), [Bus Mastering](https://en.wikipedia.org/wiki/Bus_mastering) [DMAengine](https://docs.kernel.org/driver-api/dmaengine/provider.html), [PCI in the Linux Kernel](https://tldp.org/LDP/tlk/dd/pci.html)
## Compute
-> I have 12 processors, (well 6*2 thx to hyperthreading) i should use all of them!
-> Some things are long lived and should not block a frame to be rendered (eg. loading of a file)

## Defered reclamation
In vulkan code, rather than having blocking, a coroutine could wait for a fence / timeline semaphore.
Exemple: deletion of old ressource duting swapchain recreation

## Control Flow
A frame is a graph!
Each node a task.
That could be executed wither on CPU or on GPU

# How
## Throwing some ideas
Coroutine or Fiber ?!

### Fiber aka stackful coroutines:
- Maybe 2 stacks ?  One for env / cooperation (shared, ro or atomic writes/read are needed) and a primary stack
- Scheduler 
  - Dispatch taks to threads
  - Maybe this https://dl.acm.org/doi/10.1145/1837853.1693479
  - 
- Ability to start multiple tasks at once (with id ot maybe per task data (think shader vertex shader inputs))
- Allocate task + Stack with given size (fixed)
- API could looks like draw call dispatch


## Literature survey

Work stealing
In a task based
can be done stackless

Fork Join style => Tasks are stored locally, not globally!
Thus we have 2 part of the code

See Wool paper {% cite faxenEfficientWorkStealing2010a %}. The introduction section is very good.
The book {% cite scottSharedMemorySynchronization2024 %}
Looks like [rayon](https://docs.rs/rayon/latest/rayon/)'s design.


## References

{% bibliography --cited %}
