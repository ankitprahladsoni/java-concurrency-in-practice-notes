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
locks are acquired on a per thread rather than per invocation basis.
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

## 2.5. Liveness and Performance

Limiting data to just one thread reduces performance, as a servlet will be able to entertain only ne request at a time,
instead, make sync block small, so that threads don't wait for a long time

@ThreadSafe
public class CachedFactorizer implements Servlet {
@GuardedBy("this") private BigInteger lastNumber;
@GuardedBy("this") private BigInteger[] lastFactors;
@GuardedBy("this") private long hits;
@GuardedBy("this") private long cacheHits;
public synchronized long getHits() { return hits; }
public synchronized double getCacheHitRatio() {
return (double) cacheHits / (double) hits;
}
public void service(ServletRequest req, ServletResponse resp) {
BigInteger i = extractFromRequest(req);
BigInteger[] factors = null;
synchronized (this) {
++hits;
if (i.equals(lastNumber)) {
++cacheHits;
factors = lastFactors.clone();
}
}
if (factors == null) {
factors = factor(i);
synchronized (this) {
lastNumber = i;
lastFactors = factors.clone();
}
}
encodeIntoResponse(resp, factors);

- no more big sync block, sync only essential part
- no AtomicLong, as we are using sync,
- other threads can do CPU intensive work without blocking

There is frequently a tension between simplicity and performance. When implementing a synchronization policy, resist
the temptation to prematurely sacrifice simplicity (potentially compromising safety) for the sake of performance.
Avoid holding locks during lengthy computations or operations at risk of not completing quickly such as network or
console I/O.

# 3. Sharing Objects

We want not only to prevent one thread from
modifying the state of an object when another is using it, but also to ensure that when a thread modifies the state of an
object, other threads can actually see the changes that were made.

## 3.1. Visibility

public class NoVisibility {
private static boolean ready;
private static int number;
private static class ReaderThread extends Thread {
public void run() {
while (!ready)
Thread.yield();
System.out.println(number);
}
}
public static void main(String[] args) {
new ReaderThread().start();
number = 42;
ready = true;
}
}
this simple example may not give the desired output
ReaderThread may never be able to read the new value of ready variable
and the write to number may happen later than the print in ReaderThread
it May! not always, but a possibility

### 3.1.1. Stale Data

avoid reading of stale data by syncing the whole value
@NotThreadSafe
public class MutableInteger {
private int value;
public int get() { return value; }
public void set(int value) { this.value = value; }
}

@ThreadSafe
public class SynchronizedInteger {
@GuardedBy("this") private int value;
public synchronized int get() { return value; }
public synchronized void set(int value) { this.value = value; }
}

### 3.1.2. Non atomic 64 bit Operations

When a thread reads a variable without synchronization, it may see a stale value, but at least it sees a value that was
actually placed there by some thread rather than some random value. This safety guarantee is called out of thin air
safety.
Out of thin air safety applies to all variables, with one exception: 64 bit numeric variables (double and long) that are
not declared volatile . The Java Memory Model requires fetch and store operations to be atomic,
but for nonvolatile long and double variables, the JVM is permitted to treat a 64 bit read or write as two separate 32
bit operations. If the reads and writes occur in different threads, it is therefore possible to read a nonvolatile long and
get back the high 32 bits of one value and the low 32 bits of another.Thus, even if you don't care about stale values, it
is not safe to use shared mutable long and double variables in multithreaded programs unless they are declared
volatile or guarded by a lock.

### 3.1.3. Locking and Visibility

When thread A executes a synchronized block, and subsequently thread B enters a
synchronized block guarded by the same lock, the values of variables that were visible to A prior to releasing the lock
are guaranteed to be visible to B upon acquiring the lock.Without synchronization, there is
no such guarantee.
![img](images/3_1_3_a.png)
Locking is not just about mutual exclusion; it is also about memory visibility. To ensure that all threads see the most up
to date values of shared mutable variables, the reading and writing threads must synchronize on a common lock.

### 3.1.4. Volatile Variables

When a field is declared volatile, the compiler and
runtime are put on notice that this variable is shared and that operations on it should not be reordered with other
memory operations. not cached in registers or in caches where they are hidden from other
processors, read of a volatile variable always returns the most recent write by any thread.
accessing a volatile
variable performs no locking , cannot cause the executing thread to block, making volatile variables a lighter
weight synchronization mechanism than synchronized.

writing a volatile variable is like exiting a synchronized block and reading a volatile variable is like entering
a synchronized block. However code that
relies on volatile variables for visibility of arbitrary state is more fragile and harder to understand than code that uses
locking.
Use only when they simplify implementing and verifying your synchronization policy; avoid when verifying correctness would require subtle reasoning about visibility. Good uses is ensuring the visibility of their own state, that of the object they refer to, or indicating that an
important lifecycle event (such as initialization or shutdown) has occurred.

most common use a
completion, interruption, or status flag,
For example, the semantics of volatile are
not strong enough to make the increment operation (count++) atomic.
Locking can guarantee both visibility and atomicity; volatile variables can only guarantee visibility.
use only when all the following criteria are met:

- Writes do not depend on its current value, or you can ensure that only a single thread ever
  updates the value;
- The variable does not participate in invariants with other state variables; and
- Locking is not required for any other reason while the variable is being accessed.

## 3.2. Publication and Escape

Publishing= maing code available outside the current scope, like getters or passing as reference.
compromise encapsulation
publishing objects
before they are fully constructed can compromise thread safety
An object that is published when it should not have
been is said to have escaped.
another way to publish an inner class instance
public class ThisEscape {
public ThisEscape(EventSource source) {
source.registerListener(
new EventListener() {
public void onEvent(Event e) {
doSomething(e);
}
});
}
}
When ThisEscape publishes the EventListener, it implicitly publishes the
enclosing ThisEscape instance as well, because inner class instances contain a hidden reference to the enclosing
instance.

### 3.2.1. Safe Construction Practices

Do not allow the this reference to escape during construction.
like starting the thread from constructor, create, but don't start
or use static methods to return object
public class SafeListener {
private final EventListener listener;
private SafeListener() {
listener = new EventListener() {
public void onEvent(Event e) {
doSomething(e);
}
};
}
public static SafeListener newInstance(EventSource source) {
SafeListener safe = new SafeListener();
source.registerListener(safe.listener);
return safe;
}
}

## 3.3. Thread Confinement

allow only one thread to work on an object.
Swing, JDBC pool.

### 3.3.1. Ad-hoc Thread Confinement

responsibility for maintaining thread confinement falls entirely on the
implementation. can be fragile because visibility
modifiers or local variables, don't confine the object to the target thread.

The decision to use thread confinement is often a consequence of the decision to implement a particular subsystem,
such as the GUI, as a single threaded subsystem.
It is safe to perform read modify write operations on
shared volatile variables as long as you ensure that the volatile variable is only written from a single thread. In this case,
you are confining the modification to a single thread to prevent race conditions, and the visibility guarantees for volatile
variables ensure that other threads see the most up to date value.
Because of its fragility, ad hoc thread confinement should be used sparingly; if possible, use one of the stronger forms of
thread confinement (stack confinement or ThreadLocal) instead.

### 3.3.2. Stack Confinement

an object can only be reached through local
variables. Local variables are intrinsically confined to the executing thread; they exist on the executing
thread's stack, which is not accessible to other threads. it is simpler to maintain and less fragile than ad hoc
thread confinement.

public int loadTheArk(Collection<Animal> candidates) {
SortedSet<Animal> animals;
int numPairs = 0;
Animal candidate = null;
// animals confined to method, don't let them escape!
animals = new TreeSet<Animal>(new SpeciesGenderComparator());
animals.addAll(candidates);
for (Animal a : animals) {
if (candidate == null || !candidate.isPotentialMate(a))
candidate = a;
else {
ark.load(new AnimalPair(candidate, a));
++numPairs;
candidate = null;
}
}
return numPairs;
}

Using a non thread safe object in a within thread context is still thread safe.

### 3.3.3. ThreadLocal

allows you to associate a per thread
value with a value holding object. Thread-Local provides get and set accessor methods that maintain a separate copy
of the value for each thread that uses it, so a get returns the most recent value passed to set from the currently
executing thread.
often used to prevent sharing in designs based on mutable Singletons or global variables.

```java
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {
    public Connection initialValue() {
        return DriverManager.getConnection(DB_URL);
    }
};
public static Connection getConnection() {
    return connectionHolder.get();
}
```

Conceptually, you can think of a ThreadLocal<T> as holding a Map<Thread,T> that stores the thread specific
values, though not how it is actually implemented. The thread specific values are stored in the Thread object
itself; when the thread terminates, the thread specific values can be garbage collected.
If you are porting a single threaded application to a multithreaded environment, you can preserve thread safety by
converting shared global variables into ThreadLocals, if the semantics of the shared globals permits this; an application
wide cache would not be as useful if it were turned into a number of thread local caches.

Like global variables, thread local variables can detract from reusability
and introduce hidden couplings among classes, and should therefore be used with care.

## 3.4. Immutability

state cannot be changed after construction. are inherently
thread safe; their invariants are established by the constructor, and if their state cannot be changed, these invariants
always hold.

An object is immutable if:

- Its state cannot be modified after construction;
- All its fields are final; and
- It is properly constructed (the this reference does not escape during construction).

Immutable objects can still use mutable objects internally to manage their state.The last
requirement, proper construction, is easily met since the constructor does nothing that would cause the this reference
to become accessible to code other than the constructor and its caller.

```java
@Immutable
public final class ThreeStooges {
    private final Set<String> stooges = new HashSet<String>();
    public ThreeStooges() {
        stooges.add("Moe");
        stooges.add("Larry");
        stooges.add("Curly");
    }
    public boolean isStooge(String name) {
        return stooges.contains(name);
    }
}
```

There is a difference between an object being immutable and the reference to it being
immutable.

Make variables final
return a copy of array instead of returning it directly.

## 3.5. Safe Publication

To publish an object safely, both the reference to the object and the object's state must be made visible to other
threads at the same time. A properly constructed object can be safely published by:

- Initializing an object reference from a static initializer;
- Storing a reference to it into a volatile field or AtomicReference;
- Storing a reference to it into a final field of a properly constructed object; or
- Storing a reference to it into a field that is properly guarded by a lock.

The publication requirements for an object depend on its mutability:

- Immutable objects can be published through any mechanism;
- Effectively immutable objects must be safely published;
- Mutable objects must be safely published, and must be either thread safe or guarded by a lock.

The most useful policies for using and sharing objects in a concurrent program are:

Thread confined. A thread confined object is owned exclusively by and confined to one thread, and can be modified by
its owning thread.

Shared read only. A shared read only object can be accessed concurrently by multiple threads without additional
synchronization, but cannot be modified by any thread. Shared read only objects include immutable and effectively
immutable objects.

Shared thread safe. A thread safe object performs synchronization internally, so multiple threads can freely access it
through its public interface without further synchronization. Like ConcurrentHashMap

Guarded. A guarded object can be accessed only with a specific lock held. Guarded objects include those that are
encapsulated within other thread safe objects and published objects that are known to be guarded by a specific lock.

# 4 Composing Objects

The design process for a thread􀍲safe class should include these three basic elements:

- Identify the variables that form the object's state;
- Identify the invariants that constrain the state variables;
- Establish a policy for managing concurrent access to the object's state.

You cannot ensure thread safety without understanding an object's invariants and post-conditions. Constraints on the
valid values or state transitions for state variables can create atomicity and encapsulation requirements.

Encapsulating data within an object confines access to the data to the object's methods, making it easier to ensure that
the data is always accessed with the appropriate lock held.
Confined objects must not escape their intended scope

Always return a copy.
