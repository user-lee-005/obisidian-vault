- A thread is represented by an object of the Thread class.
- Creating threads are achieved in 2 different ways:
	1. Implementing java.lang.Runnable interface
	2. Extending the java.lang.Thread class
---
# Thread Class in Java

- We need to extend this for creating threads
- We cannot create a thread using new Thread() --> because the run method would be empty
```
  The run method implemented in the thread class
	@Override  
	public void run() {  
	    if (target != null) {  
	        target.run();  
	    }  
	}
```
- Here the target is a Runnable object, when we create with a default no args constructor then the target is set to null, there for the run method doesnot run anything and just return empty output
- For making it run we need to pass a Runnable method in the constructor
```
Runnable task = () -> { System.out.println("Hello")}; // Runnable is an functional interface
Thread t1 = new Thread(task);
```
- By this method we can see output if we call t1.run();
- The invocation order, t.start() --> start0() # An inbuilt native method for creating and running OS Thread --> t.run() # start0() is not static so it has the information of the instance and tied to it so that thread's runnable is made to run
---
## Extending Thread class

- 