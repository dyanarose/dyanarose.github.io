---
layout: post
title: "graceful shutdown of java apps under docker"
date: 2017-08-26 13:39:00 +0100
comments: false
categories: 
published: true
---
tl;dr [application-interrupts](https://github.com/dyanarose/application-interrupts)

I ran across a problem recently when working on a Java application that needed to be allowed to finish processing its current batch before shutting down after the receipt of a shut down signal.

This app has a main loop that receives messages off an AWS SQS queue, processes those messages, takes action if required (making a put request to a third party), and then deletes the messages off the queue. 

Each action must only ever send a unique record to the api. Because the third party doesn't expose any unique identifier for a record, though it does provide an endpoint to get a list of all existing records, the application itself must handle the idea of uniqueness. 

So, in the case of a double whammy of my app being shut down after taking action, but before deleting the message from the queue, and the third party being slow to update the list of previous requests, I could end up sending duplicate records when the new app starts up.

## No problem, we've got SIGTERM
When `docker stop` is called on a container, SIGTERM is sent to PID 1, and in Java a [shutdown hook](https://docs.oracle.com/javase/8/docs/technotes/guides/lang/hook-design.html) is used to catch the SIGTERM and clean up any resources before finally stopping. 

In this case though, it's not resources that need cleaning up. This is a continuously running application that needs to finish any work currently in process before returning out of the main loop and stopping. 

As it happens, Java has a good way of letting a Thread know that the application would like it to shut down before it starts its next itteration of work.

## Interrupts
A Thread in Java has an `interrupted` method which returns true if the Thread has been interrupted since the last time `Thread.interrupted()` was called.
(more on [Interrupts](https://docs.oracle.com/javase/tutorial/essential/concurrency/interrupt.html))

### Why are interrupts useful 

In the main loop a condition of `while(!Thread.interrupted())` will allow the while block to run to completion, but also prevent the next execution if an interrupt occured during the previous run, which is exactly what needs to happen to allow messages to complete processing before the app shuts down.

#### How to interrupt a thread

The short answer is 'by invoking `Thread.interrupt`' in the shutdown hook.

```java
private static void threadInterruptedOnShutdown(long timeout) {
    // Setting interrupt doesn't cause the application to wait for the thread to exit.
    // The statements inside the interrupt block in the loop may or may not be executed.
    String name = "threadInterruptedOnShutdown wait " + timeout;
    Thread t = new Thread(new RunnableLoop(name, timeout));
    t.start();
    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
        System.out.println(name + " thread: setting interrupt");
        t.interrupt();
        System.out.println(name + " thread: shutting down");
    }));
}
```

Just calling interrupt on a Thread doesn't give the control needed to ensure the current work is completed. It doesn't necessarily wait on the Thread to exit before letting the application shut down. There's no reason you couldn't write the code to handle this of course, but the [ExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html) already provides that for us.

When submitting a Runnable (or Callable) to the ExecutorService, a handle for a Future is returned. Working together, the Future and the ExecutorService give control over interrupting Threads, waiting for them to exit, and if anything fails to exit, providing an opportunity to do any damage control.

```java
private static void futureFromExecutorService(long timeout) {
    // the executor service submit method allows us to get a handle on the thread
    // via a future and set the interrupt in the shutdown hook
    String name = "futureFromExecutorService wait " + timeout;
    ExecutorService service = Executors.newSingleThreadExecutor();

    Future<?> app = service.submit(new RunnableLoop(name, timeout));

    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
        System.out.println(name + " thread: setting interrupt");
        app.cancel(true);
        service.shutdown();

        try {
            // give the thread time to shutdown. This needs to be comfortably less than the
            // time the docker stop command will wait for a container to terminate on its own
            // before forcibly killing it.
            if (!service.awaitTermination(7, TimeUnit.SECONDS)) {
                System.out.println(name + " thread: did not shutdown in time, forcing service shutdown");
                service.shutdownNow();
            } else {
                System.out.println(name + " thread: shutdown cleanly");
            }
        } catch (InterruptedException e) {
            System.out.println(name + " thread: shutdown timer interrupted, forcing service shutdown");
            service.shutdownNow();
        }
    }));
}
```
To interrupt the Thread when SIGTERM is received, inside the shutdown hook call `.cancel(true)` on the Future. The boolean parameter allows the cancel method to interrupt a running Thread. Without that parameter, the Thread will not be interrupted.

Once the Future is cancelled, the service can begin shutting down. The ExecutorService has two types of shutdown. `.shutdown()` will stop any more work from being submitted to the service while allowing existing work to execute. `.shutdownNow()` on the other hand actively attempts to stop all running tasks.

After calling `.shutdown()` use the ExecutorService's `.awaitTermination` method to both give the Threads time to finish any current work and also to handle those that do not return in time. Set the timout argument to be less than that of the `docker stop` command so that there will be time to do damage control and attempt a final `.shutdownNow()` before docker kills the container for being non-responsive.

All together, this allows a continously running application to respond to shutdown requests in a timely manner and also complete the work that is currently in process.
