---
layout: post
title:  "java ByteBuffer用法"
date:   2017-11-06
categories: Java
---

## 0x01 capacity

```java
// it's right
ByteBuffer b = ByteBuffer.allocate(3);
b.put((byte)0x0a);
b.put((byte)0x0b);
b.put((byte)0x0c);

// throw exception
// Exception in thread "main" java.nio.BufferOverflowException
// 		at java.nio.Buffer.nextPutIndex(Unknown Source)
// 		at java.nio.HeapByteBuffer.put(Unknown Source)
// 		at com.ledo.kof.ByteBufferTest.main(ByteBufferTest.java:12)
ByteBuffer b = ByteBuffer.allocate(3);
b.put((byte)0x0a);
b.put((byte)0x0b);
b.put((byte)0x0c);
b.put((byte)0x0d);
```

## 0x02 position
```java
// output is 2
ByteBuffer b = ByteBuffer.allocate(3);
b.put((byte)0x0a);
b.put((byte)0x0b);
System.out.println(b.position());
```

## 0x03 limit
```java
// output is 3
// but throw exception
// Exception in thread "main" java.nio.BufferOverflowException
// 		at java.nio.Buffer.nextPutIndex(Unknown Source)
// 		at java.nio.HeapByteBuffer.put(Unknown Source)
// 		at com.ledo.kof.ByteBufferTest.main(ByteBufferTest.java:12)
ByteBuffer b = ByteBuffer.allocate(3);
b.put((byte)0x0a);
b.put((byte)0x0b);
b.limit(2);
System.out.println(b.capacity());
b.put((byte)0x0c);
```
limit代表当前读取和写入的上限，如果不设置特殊值，默认等于capacity。

## 0x04 flip
```java
// output is 3, 2, 0
ByteBuffer b = ByteBuffer.allocate(3);
b.put((byte)0x0a);
b.put((byte)0x0b);
b.flip();
System.out.format("%d, %d, %d", b.capacity(), b.limit(), b.position());
```
flip()方法可以吧Buffer从写模式切换到读模式。调用flip方法会把position归零，并设置limit为之前的position的值。 也就是说，现在position代表的是读取位置，limit标示的是已写入的数据位置。

## 0x05 rewind
```java
// output is 1, 0
ByteBuffer b = ByteBuffer.allocate(3);
b.put((byte)0x0a);
b.put((byte)0x0b);
b.flip();

b.get();
System.out.println(b.position());
b.rewind();
System.out.println(b.position());
```
rewind是的position重新置为0，可以重新读取。

## 0x06 clear
```java
// output is 3, 3, 0
ByteBuffer b = ByteBuffer.allocate(3);
b.put((byte)0x0a);
b.put((byte)0x0b);
b.clear();
System.out.format("%d, %d, %d", b.capacity(), b.limit(), b.position());
```
clear 就是将标志位重新置为初始状态，但是数据并没清，等待下次写入的覆盖。

## 0x07 compact
```java
// output is 3, 3, 2
ByteBuffer b = ByteBuffer.allocate(3);
b.put((byte)0x0a);
b.put((byte)0x0b);
b.flip();
b.compact();
System.out.format("%d, %d, %d\n", b.capacity(), b.limit(), b.position());
```
compact的作用是将position到limit之间的字节(position < limit)拷贝到0的位置上，然后position置为limit-position。

## 0x07 mark/reset
```java
output is 2 3 2
ByteBuffer b = ByteBuffer.allocate(3);
b.put((byte)0x0a);
b.put((byte)0x0b);
System.out.println(b.position());
b.mark();
b.put((byte)0x0c);
System.out.println(b.position());
b.reset();
System.out.println(b.position());
```
类似于ThreadLocal使用的save和restore。