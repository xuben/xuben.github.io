---
layout: post
title:  "ByteBuffer和String类型转换"
date:   2019-03-08 18:44:31 +0800
categories: Java
---
最近在学Java的NIO，自己写代码实践的时候数据需要在String和ByteBuffer之间互相转换，发现还挺别扭的，于是上网查了下。
# 网上常见做法
## String转ByteBuffer
```
ByteBuffer buffer = ByteBuffer.wrap("Content of the String".getBytes("utf-8"));
```

## ByteBuffer转String
buffer是ByteBuffer类型的对象
```
Charset charset = Charset.forName("utf-8");
CharBuffer charBuffer = charset.decode(buffer);
String s = charBuffer.toString();
```
也可以用
```
Charset charset = Charset.forName("utf-8");
CharsetDecoder decoder = charset.newDecoder();
CharBuffer charBuffer = decoder.decode(buffer);
String s = charBuffer.toString();
```
两者的区别在于对非法输入的处理

## 说明
第一次用`ByteBuffer.wrap`的时候惯性思维在后面加上了`flip`调用，结果发现数据并没有写入Channel。通过查源代码发现`wrap`方法返回的是一个HeapByteBuffer的对象，而HeapByteBuffer调用父类的构造函数传入的pos参数是off，也就是调用`wrap`方法的第二个参数，默认是0。
```
public static ByteBuffer wrap(byte[] array,
                                int offset, int length)
{
    try {
        return new HeapByteBuffer(array, offset, length);
    } catch (IllegalArgumentException x) {
        throw new IndexOutOfBoundsException();
    }
}
HeapByteBuffer(byte[] buf, int off, int len) { // package-private
    super(-1, off, off + len, buf.length, buf, 0);
}
ByteBuffer(int mark, int pos, int lim, int cap,   // package-private
            byte[] hb, int offset)
{
    super(mark, pos, lim, cap);
    this.hb = hb;
    this.offset = offset;
}
```
而`flip`方法的逻辑是
```
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```
因此调用`flip`方法后limit = position = 0，导致不能从Buffer中读取数据。


# 我的做法
顺便说一下我自己的转换方式
## String转ByteBuffer
```
ByteBuffer buffer = ByteBuffer.allocate(1024);
buffer.put("Content of the String".getBytes("utf-8"));
buffer.flip();
```
这种做法的缺点在于ByteBuffer大小固定，而且需要自己调用`flip`切换到读模式。

## ByteBuffer转String
buffer是ByteBuffer类型的对象
```
new String(buffer.array(), buffer.position(), buffer.limit(), "utf-8");
```