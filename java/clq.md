# ConcurrentLinkedQueue vs Size

One day, you have been tasked to write a UDP server.

Unlike TCP, which is a session oriented protocol where you can have one thread per connection, UDP cannot have a per connection threads.
However, you know the best practices to write a UDP server in Java which is to have a thread that reads packets from UDP socket and have other threads to process the packet.


If you try to both read from socket and process the UDP packet in the same thread, then packet drops may occur when the processing speed and packet arrival speed mismatch.

So you have the textbook definition of producer consumer problem. Since your task requires you to write a high performance UDP server, you decided to use [ConcurrentLinkedQueue](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentLinkedQueue.html) because it is an efficient non-blocking collection.

```java

Queue<DatagramPacket> queue = new ConcurrentLinkedQueue<>();

// producer
while(isRunning()) {
    byte [] buffer = new byte[2048];
    DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
    socket.receive(packet);
    queue.offer(packet);
}

// consumer
while(isRunning()) {
    DatagramPacket packet = queue.poll();
    if(packet == null) {
        continue;
    }
    process(packet);
}
```

Everything was nice until one day your UDP server received so many UDP packets that consumers couldn't catch, therefore queue has been filled with packets and your server died with `OutOfMemoryError`

This was not nice at all. You need to fix this immediately. 
Since this is a UDP server, and the nature of the UDP includes packet drops, you thought that it is safe to ignore packets if queue becomes 'full' i.e size of the queue is bigger than some threshold value:
```java
private static final int THRESHOLD = 10_000_000;

Queue<DatagramPacket> queue = new ConcurrentLinkedQueue<>();

// producer
while(isRunning()) {
    byte [] buffer = new byte[2048];
    DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
    socket.receive(packet);
    // discard packets if queue is full
    if(queue.size() < THRESHOLD) {
        queue.offer(packet);
    }
}

// consumer
while(isRunning()) {
    DatagramPacket packet = queue.poll();
    if(packet == null) {
        continue;
    }
    process(packet);
}
```

You are now confident that no `OutOfMemoryError`s will occur.
You deployed the new version but suddenly throughput of the server dropped, you started to see a lot of packet drops.

How? What is going on? You just added a simple size check to the code, everything else is same.
You quickly rolled back to old version and started to some benchmarks using [JMH](https://github.com/openjdk/jmh):

```
Benchmark                          Mode  Cnt        Score       Error  Units
MyBenchmark.clqOffer               thrpt   10  2115651,826 ± 75657,033  ops/s


Benchmark                          Mode  Cnt   Score    Error  Units
MyBenchmark.clqOfferWithSizeCheck  thrpt   10  13,521 ± 11,457  ops/s
```

This looks weird. Throughput of the code with size check is a lot less than the one with no size check.

No free lunch, when you gain something with non-blocking / lock-free algorithm, you need to pay the price with something else, here the price is the `size` method, from [documentation](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentLinkedQueue.html#size()):
```
public int size()
...
Beware that, unlike in most collections, this method is NOT a constant-time operation. 
Because of the asynchronous nature of these queues, determining the current number of elements
requires an O(n) traversal. 
Additionally, if elements are added or removed during execution of this method, 
the returned result may be inaccurate. 
Thus, this method is typically not very useful in concurrent applications.
```

Let me repeat: "this method is **NOT** a constant-time operation" 
"requires an **O(n)** traversal"

This is a place where coding to the interface may do some harm because implementation of the interface method has a serious drawback and one needs to be aware of it.
Last time I checked, my IDE shows the javadoc of the interface, not the implementation, this may mislead the developer.

I have seen usage of `size` method of `ConcurrentLinkedQueue` in real world applications which could have been identified and eliminated with proper profiling and benchmarking.
However sometimes the code may seem so obvious that one may not need to write benchmarks for it like checking for size of collection which is a constant-time operation most of the time.

How can we fix this? We could have used [ArrayBlockingQueue](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ArrayBlockingQueue.html) which does not suffer from this problem:
```java
Queue<DatagramPacket> queue = new ArrayBlockingQueue<>(10_000_000);

// producer
while(isRunning()) {
    byte [] buffer = new byte[2048];
    DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
    socket.receive(packet);

    // offer discards elements if queue is full, so no need to check the size explicitly
    queue.offer(packet);
}
```

Here is the same benchmark:
```
Benchmark                          Mode  Cnt         Score        Error  Units
MyBenchmark.abqOffer               thrpt   10  42262008,442 ± 812286,214  ops/s
```

We have our nice throughput values back, actually they are higher, though we need to rune some more benchmarks to make sure that this is correct.
Most probably this is because no or little allocation is needed when `offer` is called in `ArrayBlockingQueue`, however `ConcurrentLinkedQueue` needs to allocate a `Node` per `offer`.

This is also another disadvantage of `ConcurrentLinkedQueue`, it creates a lot of garbage.
I have seen a sizeable reduction of throughput due to the garbage collection that is caused by the `ConcurrentLinkedQueue` node allocations/de-allocations.

There is also a very good alternative to both queues: [JCTools](https://github.com/JCTools/JCTools)
If gives you a nice bounded / backpressure capable, less garbage creating and fast / wait free / lock less queue.
(No affiliation with the project, just a happy user)

Moral of the story:
1. Documentation is always worth to read even when things seem obvious.
2. Benchmarking / profiling / monitoring saves a lot of CPU cycles and saves you from some headaches. 