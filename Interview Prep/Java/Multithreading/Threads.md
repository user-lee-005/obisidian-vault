- A thread is an independent sequential path of execution within a program.
- Many threads can run concurrently within a program
- At runtime threads in a program exist in common memory space and therefore can, share both data and code and also the share the process running the program
---

## 3 Important Concepts related to Multithreading in Java

1. [[Creating threads]] and providing the code that gets executed by a thread
2. [[Accessing common data and code thru synchronization]]
3. [[Transitioning between thread states]]

---
# The Main thread

- Whenever we are running a single threaded program, the main thread (a user thread) is automatically created to execute the main() method.
- If no other user threads are spawned, the program terminates when the main method finishes executing
- All other threads, called child threads are spawned from the main thread
- The main method can finish, but the program will keep running until all the user threads have completed the task
- The runtime environment distinguishes between user threads and daemon threads

---

# User Thread vs Daemon Thread

- The threads are divided into 2 types based on their role in the program
- User thread - runs in the foreground performing the primary tasks - example main thread, worker thread
- Daemon thread - runs in the background providing the support services to the user thread - example gc, bg services
- Mark a thread daemon if the process not imp that much
- If there is no user thread running irrespective of the daemon threads, then the program would stop
- Calling the setDaemon(boolean) method in the Thread class marks the status of the thread as either daemon or user, but this must be done before the thread is started 
- As long as user thread is alive, the JVM does not terminate
- A daemon thread is at the mercy of the runtime system: It is stopped if there are no more user threads running, thus terminating the program