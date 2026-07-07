## JAVA Virtual Threads

Example demonstrating Java Platform Threads vs Virtual Threads

```java
public class ThreadDemo {
    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            System.out.println(
                "Hello from " + (Thread.currentThread().isVirtual() ? "Virtual Thread" : "Platform Thread") 
                + " - " + Thread.currentThread().getName()
            );
            try {
                Thread.sleep(100); // Simulate work
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        };

        // Platform Thread Example
        Thread platformThread = new Thread(task, "PlatformThread-1");
        platformThread.start();
        platformThread.join();

        // Virtual Thread Example (Java 19+)
        Thread virtualThread = Thread.ofVirtual().name("VirtualThread-1").start(task);
        virtualThread.join();
    }
}
// Output (sample):
// Hello from Platform Thread - PlatformThread-1
// Hello from Virtual Thread - VirtualThread-1\
```

Key Differences:

- Platform Threads: Traditional Java threads mapped directly to OS threads, managed by the OS.
- Virtual Threads: Lightweight threads introduced in Java 19 (as a preview), managed by the JVM, allowing creation of many more threads for concurrent tasks.
- Virtual threads make concurrent programming easier and more scalable as they use far less memory and resource per thread.
- Most APIs (like sleep()) work seamlessly with both types of threads.
*/
