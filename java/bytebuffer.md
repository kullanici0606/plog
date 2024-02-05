# Are all ByteBuffers Equal?

In Java, [ByteBuffer](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/ByteBuffer.html) is a nice utility class that makes several operations easier on byte arrays, such as type conversions (putInt / getInt).
Yesterday I noticed an interesting behavior of it.

What do you think the following code's output will be:

```java
ByteBuffer buf1 = ByteBuffer.allocate(9);
ByteBuffer buf2 = ByteBuffer.allocate(4);

buf1.put("123456789".getBytes());
buf2.put("1234".getBytes());

if(buf1.equals(buf2)) {
    buf1.flip();
    buf2.flip();
    if(buf1.equals(buf2)) {
        System.out.println("all buffers are equal");
    } else {
        System.out.println("wait a minute, I thought we were equal");
    }
} else {
    System.out.println("not equal");
}
```

One might say that the output of the program will be "not equal" since size of the buffers are different (i.e 9 vs 4) and the contents are different (i.e "123456789" vs "1234"). Therefore, two ByteBuffers must not be equal to each other.

However, this program will output "wait a minute, I thought we were equal". But why?

The definition of [equals](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/ByteBuffer.html#equals(java.lang.Object)) from the documentation is as follows:
```
Tells whether or not this buffer is equal to another object.

Two byte buffers are equal if, and only if,

    1. They have the same element type,
    2. They have the same number of remaining elements, and
    3. The two sequences of remaining elements, considered independently of their starting positions, are pointwise equal.

A byte buffer is not equal to any other type of object. 
```

So, two ByteBuffers are equal if they have same remaining elements. Their equality is not defined by their actual contents.
I am sure that there are good reasons and good use cases for this definition of equality but the way the equals defined is definitely not intuitive and may lead to some problems.

For example, since byte array (`byte[]`) does not implement `equals` and `hashCode`, the suggested way to use byte arrays as Map keys is to wrap them with ByteBuffer.

```java
Map<ByteBuffer, String> map = new HashMap<>();

private void put(byte[] key, String value) {
    map.put(ByteBuffer.wrap(key), value);
}

private String get(byte[] key) {
    return map.get(ByteBuffer.wrap(key));
}
```

This code works fine; however, let's say you want to build you keys, rather than just wrapping them. Then things can get interesting:

```java
Map<ByteBuffer, String> map = new HashMap<>();

private void put(byte[] field1, int field2, String value) {
    ByteBuffer buf = ByteBuffer.allocate(field1.length + 4); // 4 size of int
    buf.putInt(field2);
    buf.put(field1);
    map.put(buf, value);
}

private String get(byte[] field1, int field2) {
    ByteBuffer buf = ByteBuffer.allocate(field1.length + 4); // 4 size of int
    buf.putInt(field2);
    buf.put(field1);
    return map.get(buf);
}

// running
put("123456789".getBytes(), "123456789".length(), "123456789");
String value = get("1234".getBytes(), "1234".length());
System.out.println(value);
```

What will be the output of this program? "null" would be a good answer but no, it will print "123456789".
Why? Because we used a ByteBuffer with zero remaining elements as the key, and we tried to query map with a ByteBuffer with zero remaining elements.
Their `hashCode`s and `equals`s match in this case and therefore program printed "123456789"

Ok, can we fix it? Yes, by calling [flip](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/ByteBuffer.html#flip()) on ByteBuffer before using it on map operations
Is it surprising? For me, yes.

```java
private void put(byte[] field1, int field2, String value) {
    ByteBuffer buf = ByteBuffer.allocate(field1.length + 4); // 4 size of int
    buf.putInt(field2);
    buf.put(field1);
    buf.flip(); // make sure that you "flip" the buffer before storing in the map
    map.put(buf, value);
}

private String get(byte[] field1, int field2) {
    ByteBuffer buf = ByteBuffer.allocate(field1.length + 4); // 4 size of int
    buf.putInt(field2);
    buf.put(field1);
    buf.flip(); // make sure that you "flip" the buffer before querying the map
    return map.get(buf);
}
```

Another scenario where this can bite you is when you need to have a poison message to exit when using a [BlockingQueue](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/BlockingQueue.html).
Let's say you have a producer which fills some ByteBuffers and sends them the consumer to process them:

```java
// poison message
ByteBuffer EXIT_MESSAGE = ByteBuffer.allocate(0);

BlockingQueue<ByteBuffer> queue = new ArrayBlockingQueue<>(100);

int bufferSize = 8192; // 8KB
// producer
ByteBuffer buf = ByteBuffer.allocate(bufferSize);
while(isRunning()) {
    byte [] data = readFromSomewhere();
    buf.put(data);
    // buffer is full, let's pass it to consumer and start with a fresh one
    if(buf.remaining() < THRESHOLD) {
        queue.put(buf);
        buf = ByteBuffer.allocate(bufferSize);
    }
}
// we have finished producing, let's notify the consumer
queue.put(EXIT_MESSAGE);

// consumer
while(isRunning()) {
    ByteBuffer buf = queue.take();
    // exit if we have received exit notification message
    if(buf.equals(EXIT_MESSAGE)) { // --> this is wrong
        break;
    }
    process(buf);
}
```

Here we want consumer to continue processing until it receives exit message. However, when "equals" is used, consumer will exit if it receives a ByteBuffer with no remaining elements exists:
```java
// producer
byte [] data = readFromSomewhere(); // for simplicity let's say this method returned 8192 (bufferSize) bytes byte[]
buf.put(data);
// then buf.remaining() is zero
if(buf.remaining() < THRESHOLD) {
    queue.put(buf); // <- this ByteBuffer will be "equal" to END_MESSAGE
    buf = ByteBuffer.allocate(bufferSize);
}

```
So in order to fix it, we need to use reference equality (i.e == ):

```java
// consumer
while(isRunning()) {
    ByteBuffer buf = queue.take();
    // exit if we have received exit notification message
    if(buf == EXIT_MESSAGE) { // --> correct way
        break;
    }
    process(buf);
}
```

ByteBuffer is a very nice and handy utility class, but one needs to be careful while using it.
Even though behavior is documented, most of the time we do not read the documentation of things that might seem obvious such as equality. 
