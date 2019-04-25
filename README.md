# Notes based on the boot Java Concurrency In Practice

count++ looks atomic, but isn't
value is read, updated, and written.
if two threads approach, both may read old value, increment and write incorrent value.

9 -> 9+1 -> 10  
9 -> 9+1 ->10  
++ isn't atomic, to make it thread same, make this operation atomic.

same with

```java
@NotThreadSafe
public class LazyInitRace {
    private ExpensiveObject instance = null;
    public ExpensiveObject getInstance() {
        if (instance == null)
        instance = new ExpensiveObject();
        return instance;
    }
}
```

if race condition? both thread will treat instance as null and get two different object reference.

They seem atomic, but aren't.

Check this out:  
@NotThreadSafe
public class UnsafeCountingFactorizer implements Servlet {
private long count = 0;
public long getCount() { return count; }
public void service(ServletRequest req, ServletResponse resp) {
BigInteger i = extractFromRequest(req);
BigInteger[] factors = factor(i);
++count;
encodeIntoResponse(resp, factors);
}
}

++count isnt't atomic. How to fix? AtomicLong
@ThreadSafe
public class CountingFactorizer implements Servlet {
private final AtomicLong count = new AtomicLong(0);
public long getCount() { return count.get(); }
public void service(ServletRequest req, ServletResponse resp) {
BigInteger i = extractFromRequest(req);
BigInteger[] factors = factor(i);
count.incrementAndGet();
encodeIntoResponse(resp, factors);
}
}

count.incrementAndGet(); is atomic operation.
this whole method is threadsafe now.

## 2.3 Locking

What about objects? we have AtomicReference.
Let's cache the integers and factors from above example for faster performance.
@NotThreadSafe
public class UnsafeCachingFactorizer implements Servlet {
private final AtomicReference<BigInteger> lastNumber
= new AtomicReference<BigInteger>();
private final AtomicReference<BigInteger[]> lastFactors
= new AtomicReference<BigInteger[]>();
public void service(ServletRequest req, ServletResponse resp) {
BigInteger i = extractFromRequest(req);
if (i.equals(lastNumber.get()))
encodeIntoResponse(resp, lastFactors.get() );
else {
BigInteger[] factors = factor(i);
lastNumber.set(i);
lastFactors.set(factors);
encodeIntoResponse(resp, factors);
}
}
}

looks thread safe, but isn't.
lastNumber.set(i);
lastFactors.set(factors);
are atomic individually, but not together. One thread setting lastNumber will result in old value which is getting the factors.
How to make both operations only available to one thread at a time?

### 2.3.1 Intrensic locking

wrap the operation in synchronization block.

synchronized (lock) {
// Access or modify shared state guarded by lock
}

synchronized method = sync block with whole method def + current opject as lock. (static method use Class object as block)

every java object can act as lock, known as intrinsic lock or monitor locks.
The lock is automatically acquired by the executing thread before entering a synchronized block and automatically released when control exits the synchronized block, whether by the normal control path or by throwing an exception out of the block. The only way to acquire an intrinsic lock is to enter a synchronized block or method guarded by that lock.

Intrinsic locks in Java act as mutexes (or mutual exclusion locks), which means that at most one thread may own the
lock. When thread A attempts to acquire a lock held by thread B, A must wait, or block, until B releases it. If B never
releases the lock, A waits forever.

= Atomicity, one thread at a time, seems atomic relative to each thread.
@ThreadSafe
public class SynchronizedFactorizer implements Servlet {
@GuardedBy("this") private BigInteger lastNumber;
@GuardedBy("this") private BigInteger[] lastFactors;
public synchronized void service(ServletRequest req,
ServletResponse resp) {
BigInteger i = extractFromRequest(req);
if (i.equals(lastNumber))
encodeIntoResponse(resp, lastFactors);
else {
BigInteger[] factors = factor(i);
lastNumber = i;
lastFactors = factors;
encodeIntoResponse(resp, factors);
}
}
}
Fixed problems, still bad. Atomic, but degrades perfromance.

### 2.3.2. Reentrancy

If thread A wants to aquire lock held by B, it has to wait.
If A wants to aquire lock held by A, no need to wait.
intrinsic locks are reentrant.
locks are acquired on a per􀍲thread rather than per􀍲invocation basis.
Reentrancy is implemented by
associating with each lock an acquisition count and an owning thread. When the count is zero, the lock is considered
unheld. When a thread acquires a previously unheld lock, the JVM records the owner and sets the acquisition count to
one. If that same thread acquires the lock again, the count is incremented, and when the owning thread exits the
synchronized block, the count is decremented. When the count reaches zero, the lock is released.

Helps avoid blocking in the below example.
public class Widget {
public synchronized void doSomething() {
...
}
}
public class LoggingWidget extends Widget {
public synchronized void doSomething() {
System.out.println(toString() + ": calling doSomething");
super.doSomething();
}
}

Thread A calls super class and then subclass, gets lock for both.

## 2.4. Guarding State with Locks

use synchroized block to make two atomic operations as one.
Use it only when writing the value? No, use synchronised block every time you access the variables.

For each mutable state variable that may be accessed by more than one thread, all accesses to that variable must be
performed with the same lock held. In this case, we say that the variable is guarded by that lock.

Every shared, mutable variable should be guarded by exactly one lock. Make it clear to maintainers which lock that is.

encapsule all fields and put in sync block or use lock
but it can be easily forgotten for new methods to use sync or lock
For every invariant that involves more than one variable, all the variables involved in that invariant must be guarded by
the same lock.

make all methods sync? Nah
if (!vector.contains(element))
vector.add(element);

contains and add both sync, but the combination is not. Also degrades performance.
