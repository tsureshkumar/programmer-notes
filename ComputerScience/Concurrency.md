# Concurrency

This article is a quick reference to concurrency and parallel architectures and programs.  This article mainly uses Java
for showing examples


# Concurrency vs Parallelism

Concurrency is super set of Parallelism. In parallel programs, things happen in parallel and in specific order.  In
concurrent systems, the events may happen in a non deterministic way.

```
    e.g. concurrent: two users editing same word document. 
   e.g. parallel: spliting a big array into chunks and running summation on each chunk and finally summing the result.
   GPU parallelism. 
```

# Making Programs Concurrent

## Multi Processes

A process is a collection of program code, resources it uses like file handles, sockets, etc, memmory allocated. It is
an abstract concept introduced by operating systems. A process also builds a boundary around it so that no other process
can access other process' resources. When such access is requires the operating system felicitates it.  Processes share
nothing by design unless told so.

Usually a process is started by launching an executable or by other running processes.  All processes in Unix/Linux like
systems will have a parent/child/sibling relationship.  init will be parent to all processes.

In C a process is started programatically as

```C
    int fd;
    if((fd=fork()) { // parent. fd is the file descriptor of child process }
    else { // child. fd is 0. use getpid & getppid system calls to get fds }
```

All resources, open file handles, etc are duplicated after fork() call succeeds. 

## Multi-threading

A multi-threading is nothing but a collection of light weight processes which share same memory and few other resources.
A single process can launch many threads within itself and fully manage the threads inside.  All threads in the same
process has access to all memory of the process. 

In Java, a thread can be launched as

```java
// Sample.java
public class Sample {
public static void main (String [] args) throws Exception {
        Thread t1 = new Thread( () -> System.out.println("Hello") );
        t1.start(); t1.join();
    }
}
```

The multi threading poses a unique challenge of sharing resources between them and still offer consistent access to the
shared resource.  A incorrect simultaneous usage of the resource will result in failure the resource and failure of the
program and even the system.  So, it is important to implement concurrency control mechanism to order the correct access
to the shared resource.  Also when design a library or a shared object, it is critical to implement how the shared
object can be accessed concurrently and still maintain the object's invariants.

For multi processes, since not much are shared among them, the need for concurrency control is limited except for those
situations where still few things can be shared as wanted. For example, if all processes maintains a shared state like
who got maximum votes in a voting application, for maintaining the integrity, they need to employ concurrency control
mechanisms across process boundaries, like shared lock or protocols like two phase commit. This will be covered in later
sections

#  Creating Concurrent Computation

In Java, there are multiple ways a computation can be made to run in a different thread. 

##  Using Runnable

```java
// Sample.java
// inside main.
Thread t1 = new Thread( () -> System.out.println("new thread")); t1.start();
t1.join(); // wait for the threads to complete
```

In this model, the separate computation is simply another class which implements Runnable interface.  There is nothing
passed between the calling thread and the launched thread.  The calling thread can only check if the thread is launched,
is running, terminated or not, or simply wait for the other thread to complete by calling join.

## Using Executors

```java
// or with java.util.concurrent classes
// import java.util.concurrent.*;
ExecutorService e = Executors.newFixedThreadPool(5);
e.execute(() -> System.out.println("with executor"));
e.shutdown();
e.awaitTermination(5, TimeUnit.SECONDS);
```

In this model, you can control how the new thread is managed.  For example, you can limit the new computation to run on
a thread pool, when the thread pool is available. Until there are free threads in the pool, the computation will be
queued in a worker queue. There are also other kinds of executors, you can refer the documentation.  For example,
newThread executor will always launch the computation in a new thread.

## Futures

In the previous examples, we were not computing anything and simply printing a message. But, in some use cases, we may
need to execute a computation and that computation may take either long time or it may have to make an external call and
provide us with a result later.  This can be conceptually termed as a "computation with a future value".  This is
represented in Java as Future.  Futures can be created by calling "submit" of an Executor object. According to the
executor implementation, the execution can happen in separate thread.

```java
import java.util.concurrent.*;
public class FutureExample {
    public static void main (String [] args) throws Exception {
            ExecutorService e = Executors.newFixedThreadPool(5);
            Future f = e.submit(() -> { Thread.sleep(5000); return 500; });
            // do something else and allow the submitted task to happen in its own thread
            // when the program really needs the result of that future task, get it.
            System.out.println("Got results: "  +  f.get());
            e.shutdown();
            e.awaitTermination(5, TimeUnit.SECONDS);
    }   
}
```

## Promises: Completable Futures

CompletableFuture in java offers a Promise that the computation will be eventually completed. This is analogues to
"promise" in other languages like javascript.  With this, "Future" asynchronous computations can be composed to a bigger
Future. 

```java
import java.util.concurrent.*;
import java.util.function.*;
public class CompletableFutureExample {
    public static void main (String [] args) throws Exception {
            final ExecutorService e = Executors.newFixedThreadPool(5);
            
            // run a computation
            CompletableFuture<Integer> f = CompletableFuture.supplyAsync(() -> 100, e);

            //  compose another computation with f
            f = f.thenApply((s) -> s + 200 );

            // compose with another future computation of same type
            f = f.thenCompose(s -> CompletableFuture.supplyAsync(() -> s + 300, e));

            // compose with another future of different type and combine as you like
            f = f.thenCombine(CompletableFuture.supplyAsync(() -> "Hello", e), 
                                ((Integer s1, String s2) -> s1 + s2.length()));

            System.out.println("Got results: "  +  f.get());

            // handle errors in the chain
            CompletableFuture<Integer> f2 = 
                f.thenApply((Function<Integer,Integer>)( (s) -> {throw new RuntimeException(); }))
                 .handle((s,exp) -> -1);
           
            System.out.println("Got results in second future: "  +  f2.get());
            e.shutdown();
            e.awaitTermination(5, TimeUnit.SECONDS);
    }   
}
```

There are various other methods in this class that you can do different kind of composing of future computations.

Note that still the computation runs in different threads and all quirks of concurrent access problems are present. We
need to be careful about thread safety.

# Concurrency Control Primitives

When programming in concurrent systems, it very often occurs that there are common resources which these parallel
threads of execution wants access to.  Many of these shared resources exhibit incorrect behaviour if the access is to
them are interleaved. This is because the invariant associated with the resource is violated if not accessed in specific
way.  To avoid such situations, a certain mechanism is required. Concurrency primitives provide such mechanism and
according to many use cases, different primitives offer different benefits.

## Mutex

Mutex is the very basic concurrency control primitive intended for mutual exclusion. As the name suggest, concurrent
access to the resource is mutually exclusive to the current thread who acquires the mutex first.  A later thread who
tries to access the resource has to wait until the earlier thread releases the mutex once the use of the shared resource
completes.

Typically the process flow for mutually exclusive access to a resource is achieved using mutex as follows

   1. Call mutex.lock()
   2. do something
   3. mutex.unlock()

For example, let's say we have very old printer, which can print only one job at a time.  Its firmware is very old that
while it is printing, if another print command is issued it gets into Jam.  Now, let us say, we want to have two
threads, one prints odd numbers only and another prints even numbers. Here the slow printer becomes a shared resource
among the two concurrently executing threads. 

See what happens if the access to the shared resource is not mutually exclusive. Result: The printer gets into Jam

```java
import java.util.concurrent.*;
import java.util.function.*;

// incorrect implementation. Printer gets into Jam
public class MutexExample {
    public static class SingleUsePrinter {
        // a Printer which takes time to print
        // invariant: This printer cannot handle more than one job
        // if more than one job, it gets into Jam and none can print
        // after
        boolean inUse;
        void print(int i) {
            if(inUse) { System.out.println("Jam"); return; }
            inUse = true;
            try { Thread.currentThread().sleep(1000); } catch (InterruptedException e) {}
            System.out.println(i); 
            inUse = false;
        }
    }
    public static void main(String [] args) throws Exception {
        // shared printer
        final SingleUsePrinter printer = new SingleUsePrinter(); 
        Function<Integer, Runnable> incBy2AndPrint = (i) -> () -> { 
            int fi = i;
            while(true) {
                printer.print(fi);
                Thread.yield();
                fi += 2; 
            } 
        };
        Runnable evenPrinter = incBy2AndPrint.apply(0);
        Runnable oddPrinter = incBy2AndPrint.apply(1);

        ExecutorService ex = Executors.newFixedThreadPool(5);
        ex.submit(evenPrinter);
        ex.submit(oddPrinter);
        ex.shutdown();
        ex.awaitTermination(10, TimeUnit.MINUTES);
    }
}
```

The above behaviour can be fixed by using Mutexes.  In Java, mutexes can be created by two methods. One by "monitor"
another by expilicitly creating java.util.concurrent.locks.Lock interface.

### Implicit Monitors

Every object in Java, by default, holds a mutex called "monitor". This monitor can be locked by having "synchronized"
keyword on the object. "synchronized" keyworod can be applied at method level or with explicit block "synchronized(obj)
{}". The lock is released when control comes out of scope of method or the block.

Check what happens by chaing the line like below in the above example.

```java
    // printer works correctly
    synchronized void print(int i) {
```
Now, the printer no longer Jams.  The access to the printer's print method is given exclusively to the caller until the
method completes.  Whoever else calling the printer, has to wait. Once completed, the other thread gets access to the
printer. This way, the invariant "inUse" is always false, when print method is executed.

### Explicit Lock

With java.util.concurrent, Java introduced explicit concurrency control utilities. With this, we can implement an
explicit mutex. The disadvantage of "synchronized" monitor above, is that mutual exclusion is confined to the method or
block.  If we want to have an exclusion across methods/blocks, we need to use explicit locks.  With this, we can lock in
one method and release control when the caller calls another method.

The same above example can be re-written as 

```java
import java.util.concurrent.*;
import java.util.concurrent.locks.*;
import java.util.function.*;

public class MutexExample {
    public static class SingleUsePrinter {
        // a Printer which takes time to print
        // invariant: This printer cannot handle more than one job
        // if more than one job, it gets into Jam and none can print
        // after
        ReentrantLock lock = new ReentrantLock();
        int jobContent;
        private void prepare(int i) { 
            lock.lock(); 
            try {
                try { Thread.currentThread().sleep(1000); } catch (InterruptedException e) { e.printStackTrace(); }
                jobContent = i; 
            } catch (Exception e) { System.out.println(e.getMessage()); lock.unlock(); }
        }
        private void print() {
            if (!lock.isHeldByCurrentThread()) { System.out.println("rejected command. prepare first"); return; }
            try {
                System.out.println(jobContent); 
            } finally {
                lock.unlock();
            }
        }
        public void print(int i) {
            prepare(i);
            print();
        }
    }
    public static void main(String [] args) throws Exception {
        // shared printer
        final SingleUsePrinter printer = new SingleUsePrinter(); 
        Function<Integer, Runnable> incBy2AndPrint = (i) -> () -> { 
            try {
            int fi = i;
            while(true) {
                printer.print(fi);
                Thread.yield();
                fi += 2; 
            } 
            } catch(Exception e) { e.printStackTrace(); }
        };
        Runnable evenPrinter = incBy2AndPrint.apply(0);
        Runnable oddPrinter = incBy2AndPrint.apply(1);

        ExecutorService ex = Executors.newFixedThreadPool(5);
        ex.submit(evenPrinter);
        ex.submit(oddPrinter);
        ex.shutdown();
        ex.awaitTermination(10, TimeUnit.MINUTES);
    }
}
```
Note, now we have split the print method into "prepare" phase and another "print" phase.  Callers has to first get into
"prepare" phase and then complete the printing by calling "print".  Mutex lock makes sure that this invariant is
maintained and when one job is in printing, it bars another from even preparing.  This is not possible with
"synchronized" monitors that we discussed above. However, never expose these methods outside object for callers and
never trust that callers are going to call in the intended order only. For example, an evil caller can simply call
"prepare" and never call "print()" leading into system not available for other callers.

also note the line

```java
        void print() {
            if (!lock.isHeldByCurrentThread()) { System.out.println("rejected command. prepare first"); return; }
```

This line makes sure that lock is already acquired by the calling thread in "prepare" method. Otherwise, the caller is
directly calling without preparing.

We are also calling "unlock" typically in all exception blocks and finally blocks. Otherwise, system will go into a hung
state where the caller is not available to unlock the call.

Important notes when using mutex:

   * Keep the lock only for short duration and release them immediately once critical section is done
   * Never assume callers are going to call in correct order and they will always call the method where you unlock the
     mutex
   * The number of times you call "lock" or "tryLock", there should be same number or more calls to "unlock"

## Condition Variables

Sometimes, you want to block *all* other threads until a condition is satisfied. And also, the thread which satisfies
the condition needs a mechanism to unblock the threads waiting for the condition.

Condition variables comes to rescue for this situation. Usually, the condition variables are associated with a
condition. This condition mutation needs to be mutually exclusive. Hence, a lock is required to protect this condition.
So, condition variables are always associated with a mutex.  Condition variables' `await` typically releases the
_associated_ mutex and waits for some other thread to _signal_. Once released, the waiting thread *needs* to re-acquire
the lock.

A typical algorithm flow for conditional waiting and processing as follows

   1. define the condition
   1. create a mutex
   1. create a condition variable associated with the condition 1.
   1. check if condition is true.
   1. if condition is met, execute your code
   1. if not met, condition.await until someone satisfies the condition
   1. Anyone satisfies the condition, please call "condition.signal" so that whoever awaiting gets awoken

Let's see an example. We want to have two threads, one printing odd numbers and other printing even numbers.  But, we
want them to alternate, i.e. we want to print one odd and one even and keep continuing.  The output will be then
sequenced. If we use mutex, then always the faster thread wins. But, we want to alternate based on a condition. So a
condition variable best suits this use case.

```java
import java.util.concurrent.*;
import java.util.concurrent.locks.*;
import java.util.function.*;

public class ConditionVariableExample {
    public static void sleep(int msecs) {
        try { Thread.currentThread().sleep(msecs); } 
        catch (InterruptedException e) { e.printStackTrace(); }
    }

    static Lock lock = new ReentrantLock();
    // we want to alternate between one odd and one even
    // so that output comes as sequential numbers
    // Control execution based on last printed value's condition
    static boolean lastPrintedEven = false;
    static Condition oddPrinted = lock.newCondition();
    static Condition evenPrinted = lock.newCondition();

    public static void main(String [] args) throws Exception {
        // let's alternate between odd and even threads co-operatively
        Runnable evenPrinter = () -> {
            try {
            for(int i=0; ; ) { 
                lock.lock();
                if (lastPrintedEven) {
                    // wait until condition changes
                    oddPrinted.await(); // releases lock and waits
                    // automatically re-acquires the lock
                }
                System.out.println(i); i+=2;
                System.out.flush();
                // toggle the condition & signal
                lastPrintedEven = true; evenPrinted.signal();
                lock.unlock();
                sleep(1000);
                Thread.yield();
            }
            } catch (InterruptedException e) {e.printStackTrace(); }
        };
        Runnable oddPrinter = () -> {
            try {
            for(int i=1; ; ) {
                lock.lock();
                if (!lastPrintedEven) {
                    evenPrinted.await();
                }
                System.out.println(i); i+=2;
                System.out.flush();
                lastPrintedEven = false;
                oddPrinted.signal();
                lock.unlock();
                sleep(2000);
                Thread.yield();
            }
            } catch (InterruptedException e) {e.printStackTrace(); }
        };

        ExecutorService ex = Executors.newFixedThreadPool(5);
        ex.submit(evenPrinter);
        ex.submit(oddPrinter);
        ex.shutdown();
        ex.awaitTermination(10, TimeUnit.MINUTES);
    }
}
```
Even if the even threads completes fast printing one number and gets scheduled again, it'll check if lastPrintedEven is
true. If so, it'll await until the oddPrinted condition becomes true.

Another typical example is "bounded buffer problem". If we have a buffer of fixed size, and multiple threads are filling
the buffer, we want to wait before filling, if the buffer is full until someone makes the buffer not full. Similarly, we
want to wait before reading, if buffer is empty until someone makes it not empty.

```java

import java.util.concurrent.*;
import java.util.concurrent.locks.*;
import java.util.function.*;

public class BoundedBuffer {
    public static void sleep(int msecs) {
        try { Thread.currentThread().sleep(msecs); } 
        catch (InterruptedException e) { e.printStackTrace(); }
    }
    public static void print(Object o) {
        System.out.println(Thread.currentThread().getName() +  " " + o);
    }

    static Lock lock = new ReentrantLock();
    static final int MAX = 5;
    static int bottom = 0, top=0, counter = 0;
    static int [] buffer = new int [MAX];
    static Condition notFull = lock.newCondition();
    static Condition notEmpty = lock.newCondition();
    static boolean isFull() { return  counter == MAX; };
    static boolean isEmpty() { return counter == 0; };

    static void fill(int x) {
        try {
            lock.lock();
            while (isFull()) { notFull.await(); }
            buffer[top++] = x;
            counter++;
            top %= MAX;
            notEmpty.signalAll();
        } catch (Exception e) { e.printStackTrace();
        } finally { lock.unlock(); }
    }

    static int read() {
        try {
            lock.lock();
            while(isEmpty()) { notEmpty.await(); }
            int ret = buffer[bottom++];
            counter--;
            bottom %= MAX;
            notFull.signalAll();
            return ret;
        } catch (Exception e) { e.printStackTrace(); return -1;
        } finally { lock.unlock(); }
    }

    public static void main(String [] args) throws Exception {
        ExecutorService ex = Executors.newCachedThreadPool();
        ex.execute(() -> {for(int i=0; i<10; i++) { fill(i); }});
        ex.execute(() -> {for(int i=0; i<10; i++) { System.out.println(read()); }});
        ex.shutdown();
        ex.awaitTermination(10, TimeUnit.MINUTES);
    }
}
```

## Semaphore

Mutex allows mutually exclusive access to one single thread. There are usecases where the access can be given
concurrently to a restricted number of threads simultaneously and the shared resource can handle. For example, consider
the above printer we dealt with. If the printer can handle more than one job at time but not more than 5 jobs, such a
concurrency control can be employed using Semaphores. Mutex is nothing but a semaphore with just one access.

A typical usage of semaphore can be implemented as follows
   1. create a semaphore with upper bound capacity
   1. call acquire. 
   1. if there are enough place in semaphore, the thread will continue executing. If not, it will wait until some other
      thread releases the lock.
   1. use the printer
   1. call release. This releases the resource back to pool available for other threads

An example of a printer

```java
import java.util.concurrent.*;
import java.util.*;

public class SemaphoreExample {
    static class Printer {
        Queue<Integer> jobQueue = new LinkedList<>();
        Semaphore sem = new Semaphore(5, true /*fairness*/);
        ExecutorService ex = Executors.newFixedThreadPool(15);
        void postJob(int job) {
            try {
                sem.acquire();
                jobQueue.add(job);
                System.out.println("Posted job: " + job);
            }catch (Exception e) { e.printStackTrace(); }
        }
        void powerup() {
            ex.execute(() -> {
                sleep(5000); // printer warming up
                while(true) {
                    if(!jobQueue.isEmpty()) {
                        int x = jobQueue.remove();
                        System.out.println("Executing job: " + x);
                        sleep(1000);
                        sem.release();
                    }
                    sleep(1000);
                }
            });
        }
        void powerdown(int waitTime, TimeUnit units) { 
            try {
                ex.shutdown(); 
                ex.awaitTermination(waitTime, units);
            } catch (InterruptedException e) { e.printStackTrace(); }
        }
    }

    public static void sleep(int msecs) {
        try { Thread.currentThread().sleep(msecs); } 
        catch (InterruptedException e) { e.printStackTrace(); }
    }

    public static void main (String [] args) {
        Printer printer = new Printer();
        printer.powerup();
        for(int i=1; i < 10; i++) {
           printer.postJob(i*100); 
        }
        printer.powerdown(5, TimeUnit.MINUTES);
    }
}
```

## Reader Writer Locks
In some situations, your applications may be doing more read operations than a write operation to a shared resource. The
shared resource doesn't change much. In that case, using a mutex is costlier as all reader threads will be blocked
until current reading thread finishes. Since the resource is not modified, it doesn't need a lock. 

But, when a thread writes, the other threads has to wait for the thread to complete otherwise they will be getting a
stale copy or worst a inconsistent value if it interleaves the writer's critical section

Notes:

   * If the read operations are really short lived, it doesn't offset the overhead of read write lock
   * If writes are more, obviously it'll make all threads spend time exclusively on thread defeating the purpose of read
     write locks
   * When a write lock is released and there are both new read lock and write lock are pending. Which one to give
     preference? Most commonly, the write lock is given
   * Java Implementation of ReentrantReadWriteLock
       - Doesn't implement Read/Write preference. Instead implements a fairness parameter.  If chosen, when a write lock
         is released, it gives preference to a longest waiting writer thread waiting more time than all the read. If
         group of reader threads waits loger, then that group will be assigned the lock making writer wait
       - With fairness setting, the write lock will block unless both write lock and read lock are free. If read is in
         currently progress, writer has to wait until all reads are done
       - Without fairness setting, the order of read/write lock is unspecified. Continously contending thread may be
         postponed for longer period. 
       - non-fairness gives better throughput
       - A writer can acquire a read lock, not vice versa.
       - A write lock can be downgraded to read lock, again not vice versa.
       - writeLock supports condition so that you can await and signal and not read lock.

Example in Java. A cache which is read more often then updated
```
import java.util.concurrent.*;
import java.util.concurrent.locks.*;
public class Cache {
    ReadWriteLock lock = new ReentrantReadWriteLock();
    void updateCache() { /*...*/ }
    boolean isDirty() { /*...*/ return false; }
    void getEntry(String itemId) {
        lock.readLock().lock();
        if( isDirty()) {
            lock.readLock().unlock();// a reader can't acquire a write lock, so release first
            lock.writeLock().lock(); 
            updateCache();
            lock.writeLock().unlock();
        }
        lock.readLock().unlock();
    }
}
```
### StampedLock
Similar to Read Write Lock above except it returns a stamp value, which can later be used for unlock or check if still
lock is valid. It also supports optimistic lock mode.

## Deadlock
### Detection
### Prevention
## Resource Starvation
# Concurrency Models
## Worker
## Worker Pool
## Continous Sequential Processing
## Actor Model
## Reactive
## Main Loop
## Publisher Subscriber
## Observable and Observer

# Thread Safety

## Safely Publishing Objects

Reference: [Book Java Concurrency and Practice](https://www.amazon.in/Java-Concurrency-Practice-Tim-Peierls-ebook/dp/B004V9OA84)

Publishing a object here means making them available to other objects.  Watch out for unsafe publishing. For example

```java
public class Books {
    public static ArrayList<Book> books;
    public Books() { books = new ArrayList<>(); }  
}
```
Even before the object is constructed, we are publishing the books variable which can be accessed by anyone and that
will be null until the constructor completes.

Similarly follow the best practices so as not to publish objects partially, before fully constructed. Once published,
the reference may be kept outside or misused.

   * never subscribe to any listener in constructor. The `this` reference pointer escapes
   * never start a thread in constructor. Since object is not fully constructed, the thread will have reference to the
     partial state. Instead construct the thread but start it outside constructor
   * Use private constructor and public factory method for properly publishing objects
   * Publish the reference and fully constructed object at the same time
      - by using final field of a fully constructured object
      - by storing into a volatile  varilable or AtomicReference
      - using static initializer
      - using synchronization primitives


# Lock Free Programming

Concurrency  provides benefit in lot of cases. But, a mis-behaving concurrent program is often very hard to debug.
Since concurrent execution order of statements are non-deterministic, it is very difficult to reason about a program
just looking at small portion of logic. 

Also, concurrency requires the access to common resource is synchronized and sometimes we may need to have deterministic
order. In those cases, we have to sequence the program through a sequencer making them single threaded, which defeats
the advantages of concurrent programming.

So where ever possible, try to use lock free primitives and avoid concurrency which requires concurrency control.  New
programming paradigms are emerging where concurrency controls are minimized and same use case can be achieved without
resorting to complex ordering techniques.  For example, message passing instead of shared memory, immutable data,
immutable data structures are techniques used for avoiding shared state.

The advantage of avoiding synchronization or locks is greater freedom to run concurrent threads/process without worrying
about race conditions or invalid access patterns.  With mutli core processes and distributed systems, these kind of
programs are easy to reason about and can easily parallelized across cores or machines.

Let us see some techniques to avoid locks and synchronization.

## Compare and Swap: AtomicInteger, AtomicLong, AtomicBoolean, AtomicReference, LongAdder, LongAccumulator

Simplest technique to avoid locks is to use atomic data structures.  These structures uses machine instructions which
execute in a single pass, not allowing multiple execution paths interleave.

Java's AtomicInteger offers an integer which can be increated and read in a single call.

```java
AtomicInteger value = new AtomicInteger(0);
System.out.println(value.incrementAndGet());
```
It also supports various methods `updateAndGet(n -> n+1)`, `accumulateAndGet(i, (x,y) -> x+y)`

## Volatile Fields
Java's volatile keyword makes the value of a variable  stored in main memory  instead of a cpu cache. i.e. when a thread
is run a multi-core machine, each cpu has its own registry and cache.  The value can be copied into these registers.  An
update to these values may not be visible to other threads. Volatile makes it to use the main memory instead of cache so
that an update of the variable is immediately visible to other threads.

Note that, this still does not guarantee synchronization if the writer depends on previous value of volatile. i.e. read
and write are different operations which can interleave.

When a update to a volatile is done, it's not just this variable, all other volatile values also gets flushed to main
memory.

To make full use of volatile, use Atomic variables we discussed in previous section. Or more useful if only one thread
updates and other threads only reads the value very often.

Careful evaluation is needed before using volatile, because accessing from cpu cache is faster.

## Final fields
When many of the class fields are constants and does not change during course of the program, it is better to mark them
as final. Marking them final guarantees that these variables are initialized immediately after constructor completes.

## Immutable
Immutable objects can never change stage. Hence they are free to use across multiple threads without worrying about
synchronization. For threads which want a mutation of the object, they can mutate by copying. Example for such object is
Java's String objects. 

In Java, a class can be immutable by declaring all fields as final and class itself as final and internal state is not
escaped outside the class during construction or there times.

## Thread local storage
When you store the object within a thread local storage and never shared or escape them out of scope, you can freely use
it without synchronization

### Fork Join
## Immutablity
## Lock Free Data Structures
## Message Passing
## Reactive Programming
