---
title: I/O Streams
date: 2018-08-01 14:57:03
tags:
    - EssentialClasses
---

## I/O Streams
* [Byte Streams](https://docs.oracle.com/javase/tutorial/essential/io/bytestreams.html) handle I/O of raw binary data.
* [Character Streams](https://docs.oracle.com/javase/tutorial/essential/io/charstreams.html) handle I/O of character data, automatically handling translation to and from the local character set.
* [Buffered Streams](https://docs.oracle.com/javase/tutorial/essential/io/buffers.html) optimize input and output by reducing the number of calls to the native API.
* [Scanning and Formatting](https://docs.oracle.com/javase/tutorial/essential/io/scanfor.html) allows a program to read and write formatted text.
* [I/O from the Command Line](https://docs.oracle.com/javase/tutorial/essential/io/cl.html) describes the Standard Streams and the Console object.
* [Data Streams](https://docs.oracle.com/javase/tutorial/essential/io/datastreams.html) handle binary I/O of primitive data type and String values.
* [Object Streams](https://docs.oracle.com/javase/tutorial/essential/io/objectstreams.html) handle binary I/O of objects.


## InputStream

[InputStream](https://docs.oracle.com/javase/8/docs/api/java/io/InputStream.html)表示输入流，它是所有字节输入流实现类的父类。
```java
public abstract class InputStream implements Closeable {
    // MAX_SKIP_BUFFER_SIZE is used to determine the maximum buffer size to
    // use when skipping.
    private static final int MAX_SKIP_BUFFER_SIZE = 2048;

    // 读取字节流中的一个字节，返回该字节
    public abstract int read() throws IOException;

    // 读取字节流中的b.length个字节，将读取到的字节存放到b数组
    // 返回读取到的字节个数
    public int read(byte b[]) throws IOException {
        return read(b, 0, b.length);
    }
    
    // 读取字节流中的len个字节，将读取到的字节存放到b数组，从b数组的off位置开始存放
    // 返回读取到的字节个数
    public int read(byte b[], int off, int len) throws IOException {
        // ...
    }

    // 跳过字节流的n个字节，返回实际跳过的字节个数
    public long skip(long n) throws IOException {
        // ...
    }

    // 返回字节流中剩余的字节个数
    public int available() throws IOException {
        return 0;
    }

    // 关闭输入流，释放占用的系统资源
    public void close() throws IOException {}

    // 标记输入流的当前位置，后面调用reset方法的时候可以回到这个位置
    // 然后就能够在这个位置继续向后读取最多readlimit个字节
    public synchronized void mark(int readlimit) {}

    // 回到mark方法标记的位置
    public synchronized void reset() throws IOException {
        throw new IOException("mark/reset not supported");
    }
    
    // 判断输入流是否支持mark和reset方法
    public boolean markSupported() {
        return false;
    }
}
```

* __read__ 读取输入流的字节，然后返回读取的字节或者读取的字节数，如果已到输入流的末尾，则返回-1

* __skip__ 跳过输入流的n个字节，然后下次读取的时候直接读取n个字节后面的自己，不再读取这些跳过的字节，返回实际跳过的字节数

* __available__ 返回输入流中剩余的字节数，如果已到输入流的末尾，则返回0

* __close__ 关闭输入流，释放占用的系统资源

* __mark__ 标记输入流当前的位置，后面调用reset方法的时候能够回到这个位置继续向后读取最多readlimit个字节。在`InputStream`中，该方法默认什么也不做。

* __reset__ 回到mark方法标记的位置，如果输入流的markSupported方法返回false，则reset方法可能抛出`IOException`，如果不抛出异常，输入流将被重置。如果没有调用过mark方法，则抛出异常。在`InputStream`中，该方法默认什么也不做。

* __markSupported__ 测试输入流是否支持mark和reset操作。在`InputStream`中，该方法默认返回false，表示默认不支持mark和reset操作，子类需要自己重写这个方法来支持这些操作。


## OutputStream

[OutputStream](https://docs.oracle.com/javase/8/docs/api/java/io/OutputStream.html)表示输出流，它是所有字节输出流的实现了的父类。
```java
public abstract class OutputStream implements Closeable, Flushable {

    // 写一个字节到输出流
    public abstract void write(int b) throws IOException;

    // 从b数组写b.length个字节到输出流
    public void write(byte b[]) throws IOException {
        write(b, 0, b.length);
    }

    // 从b数组的off位置开始写len个字节到输出流
    public void write(byte b[], int off, int len) throws IOException {
        if (b == null) {
            throw new NullPointerException();
        } else if ((off < 0) || (off > b.length) || (len < 0) ||
                   ((off + len) > b.length) || ((off + len) < 0)) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }
        for (int i = 0 ; i < len ; i++) {
            write(b[off + i]);
        }
    }

    // 清除输出流，把在缓冲区的数据强制写入到存储设备（比如磁盘）
    public void flush() throws IOException {
    }

    // 关闭输出流，释放占用的系统资源
    public void close() throws IOException {
    }

}

```

* __write__ 把1个或者多个字节写到输出流，接受的参数为int类型，大小为32bits（比特），即8bytes（子节），因为是字节流，所以会将int类型数据的低8位，也就是最后面的8bits写到输出流，高24位的数据将会被丢弃

* __flush__ 清除输出流，这会使得在缓冲区的数据（之前调用write方法写入的数据一般都保存在缓冲区中）被立即强制写入到存储设备（比如硬盘）

* __close__ 关闭输出流，释放占用的系统资源

## Reader

[Reader](https://docs.oracle.com/javase/8/docs/api/java/io/Reader.html)是读取字符流的抽象类，它是所有读取字符流的实现类的父类。

```java
public abstract class Reader implements Readable, Closeable {

    /**
     * The object used to synchronize operations on this stream.  For
     * efficiency, a character-stream object may use an object other than
     * itself to protect critical sections.  A subclass should therefore use
     * the object in this field rather than <tt>this</tt> or a synchronized
     * method.
     */
    protected Object lock;

    /**
     * Creates a new character-stream reader whose critical sections will
     * synchronize on the reader itself.
     */
    protected Reader() {
        this.lock = this;
    }

    /**
     * Creates a new character-stream reader whose critical sections will
     * synchronize on the given object.
     *
     * @param lock  The Object to synchronize on.
     */
    protected Reader(Object lock) {
        if (lock == null) {
            throw new NullPointerException();
        }
        this.lock = lock;
    }

    // 尝试从字符流中读取字符到CharBuffer对象（target）中，返回读取的字符数
    public int read(java.nio.CharBuffer target) throws IOException {
        int len = target.remaining();
        char[] cbuf = new char[len];
        int n = read(cbuf, 0, len);
        if (n > 0)
            target.put(cbuf, 0, n);
        return n;
    }

    // 从字符流中读取一个字符，然后返回该字符
    public int read() throws IOException {
        char cb[] = new char[1];
        if (read(cb, 0, 1) == -1)
            return -1;
        else
            return cb[0];
    }

    // 从字符流中读取cbuf.length个字符到cbuf数组中，然后返回读取的字符个数
    public int read(char cbuf[]) throws IOException {
        return read(cbuf, 0, cbuf.length);
    }

    // 从字符流中读取len个字符，存放到cbuf数组中，从cbuf数组的off位置开始存放
    abstract public int read(char cbuf[], int off, int len) throws IOException;

    /** Maximum skip-buffer size */
    private static final int maxSkipBufferSize = 8192;

    /** Skip buffer, null until allocated */
    private char skipBuffer[] = null;

    // 跳过n个字符
    public long skip(long n) throws IOException {
        // ...
    }

    // 测试字符输入流是否已经可以被读取
    public boolean ready() throws IOException {
        return false;
    }
    
    // 测试字符流是否支持mark和reset操作
    public boolean markSupported() {
        return false;
    }

    public void mark(int readAheadLimit) throws IOException {
        throw new IOException("mark() not supported");
    }

    public void reset() throws IOException {
        throw new IOException("reset() not supported");
    }

    abstract public void close() throws IOException;

}
```

可以看到和`InputStream`相比，`Reader`仅仅多了一个ready方法，其它方法名都是相同的，比如除了方法的参数类型由`byte`换成了`char`以外，基本上没有区别。

* __ready__ 测试字符流是否已经可以被读取，如果返回true，则说明下一次调用read可以马上读取到数据，不需要等待数据到来，如果返回false，则可能可以马上读取到数据，也可能需要等待数据到来（一直阻塞，直到有数据到来）

其它方法就不一一列举了，和`InputStream`的方法基本上是一样的，只是换了个参数类型。

## Writer

[Writer](https://docs.oracle.com/javase/8/docs/api/java/io/Writer.html)是写数据到字符流的抽象类，它是所有写数据到字符流的实现类的父类。

```java
public abstract class Writer implements Appendable, Closeable, Flushable {

    /**
     * Temporary buffer used to hold writes of strings and single characters
     */
    private char[] writeBuffer;

    /**
     * Size of writeBuffer, must be >= 1
     */
    private static final int WRITE_BUFFER_SIZE = 1024;

    /**
     * The object used to synchronize operations on this stream.  For
     * efficiency, a character-stream object may use an object other than
     * itself to protect critical sections.  A subclass should therefore use
     * the object in this field rather than <tt>this</tt> or a synchronized
     * method.
     */
    protected Object lock;

    /**
     * Creates a new character-stream writer whose critical sections will
     * synchronize on the writer itself.
     */
    protected Writer() {
        this.lock = this;
    }

    /**
     * Creates a new character-stream writer whose critical sections will
     * synchronize on the given object.
     *
     * @param  lock
     *         Object to synchronize on
     */
    protected Writer(Object lock) {
        if (lock == null) {
            throw new NullPointerException();
        }
        this.lock = lock;
    }

    // 写一个字符到字符流
    public void write(int c) throws IOException {
        synchronized (lock) {
            if (writeBuffer == null){
                writeBuffer = new char[WRITE_BUFFER_SIZE];
            }
            writeBuffer[0] = (char) c;
            write(writeBuffer, 0, 1);
        }
    }

    // 把cbuf数组里面的字符都写入到字符流
    public void write(char cbuf[]) throws IOException {
        write(cbuf, 0, cbuf.length);
    }
    
    // 把cbuf数组从off位置开始的字符都写入到字符流
    abstract public void write(char cbuf[], int off, int len) throws IOException;

    // 把一个字符串的所有字符写入到字符流
    public void write(String str) throws IOException {
        write(str, 0, str.length());
    }

    // 把字符串从off位置开始的len个字符写入到字符流
    public void write(String str, int off, int len) throws IOException {
        synchronized (lock) {
            char cbuf[];
            if (len <= WRITE_BUFFER_SIZE) {
                if (writeBuffer == null) {
                    writeBuffer = new char[WRITE_BUFFER_SIZE];
                }
                cbuf = writeBuffer;
            } else {    // Don't permanently allocate very large buffers.
                cbuf = new char[len];
            }
            str.getChars(off, (off + len), cbuf, 0);
            write(cbuf, 0, len);
        }
    }

    // 把一个字符序列的所有字符写入到字符流，并返回该Writer的引用
    public Writer append(CharSequence csq) throws IOException {
        if (csq == null)
            write("null");
        else
            write(csq.toString());
        return this;
    }

    // 把一个字符序列的start位置到end位置的字符写入到字符流，并返回该Writer的引用
    public Writer append(CharSequence csq, int start, int end) throws IOException {
        CharSequence cs = (csq == null ? "null" : csq);
        write(cs.subSequence(start, end).toString());
        return this;
    }

    // 把一个字符写入到字符流，并返回该Writer的引用
    public Writer append(char c) throws IOException {
        write(c);
        return this;
    }

    abstract public void flush() throws IOException;

    abstract public void close() throws IOException;

}

```

* __write__ 写字符到字符流，甚至写字符串和字符序列（实际上也是字符，因为它们都是由字符组成的）到字符流。需要注意的是，字符（char）为16位，而int为32位，所以当传入int类型的参数时，会把低16位的数据写入字符流，高16位的数据将被丢弃

* __append__ 该方法写字符序列到字符流之后，还会返回`Writer`的引用，这样就可以链式调用append方法不断写入字符序列到字符流了，比如`writer.appeend("abc").appeend("def")`

其它方法和`OutputStream`基本上是一样的。



