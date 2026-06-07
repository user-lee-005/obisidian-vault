# Multitasking
- Allowing several activities to occur concurrently on the computer
- There are 2 types of multitasking
	1. Process based multitasking
	2. Thread based multitasking
- Process based multitasking - Running multiple programs simultaneously
- Thread based multitasking - Running different tasks in the same program
---
# Thread vs Process
- A ***process*** is an independent program in execution with its own memory space, while a ***[[Threads]]*** are lightweight unit of execution within a process that shares memory and resources with other threads
- 2 threads share the same address space
- Because of this switching between 2 threads is less expensive than switching between 2 process
- The cost of communications between the threads is relatively low
---
## Why Multithreading

When the CPU is waiting for the user input, the cpu can use the idle time to do another task

In a single threaded env only one task can be performed, CPU cycles are wasted
