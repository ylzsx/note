## io流

- 字节流

  ByteArray/File/Object/Piped

  Filter:Data/Buffer

  Sequence

  Print

- 字符流

  CharArray/String/Buffered/File

## NIO

- nio

  块缓存、同步非阻塞、有选择器

  - Channel

    - FileChannel
    - DatagramChannel UDP
    - SocketChannel TCP client
    - ServerSocketChannel TCP Server

  - Buffer

    - ByteBuffer
    - CharBuffer
    - DoubleBuffer
    - FloatBuffer
    - IntBuffer
    - LongBuffer
    - ShortBuffer

    Buffer中的索引：mark、position、limit、capacity

    > capacity：**缓冲区能够容纳的数据元素的最大数量**。容量在缓冲区创建时被设定，并且永远不能被改变。
    >
    > limit：**缓冲区里的数据的总数**，代表了当前缓冲区中一共有多少数据。
    >
    > position：**下一个要被读或写的元素的位置**。position会自动由相应的 `get( )`和 `put( )`函数更新。
    >
    > mark：一个备忘位置。**用于记录上一次读写的位置**。
    >
    > 0 <= mark <= position <= limit <= capacity
    >
    > `filp()`：切换为读模式，`limit = postion`，`position = 0`
    >
    > ​	limit限制读到哪里，position控制从哪儿开始读。
    >
    >

- io

  流、同步阻塞、没有选择器 

- aio

  异步feizuse









```SAS
// 指令测试文件 inst.S
addi $3,$3,0x4000
addi $2,$2,0x4000
addi $3,$3,0x4000
addi $2,$2,0x4000
addi $3,$3,0x4000
addi $2,$2,0x4000
addi $3,$3,0x4000
addi $2,$2,0x4000
addi $4,$4,0x4000
addi $4,$4,0x4000
```

```SAS
// 上述指令对应的机器指令 inst.data
001000 00011 00011 0100 0000 0000 0000 -> 20634000
001000 00010 00010 0100 0000 0000 0000 -> 20424000
001000 00011 00011 0100 0000 0000 0000 -> 20634000
001000 00010 00010 0100 0000 0000 0000 -> 20424000
001000 00011 00011 0100 0000 0000 0000 -> 20634000
001000 00010 00010 0100 0000 0000 0000 -> 20424000
001000 00011 00011 0100 0000 0000 0000 -> 20634000
001000 00010 00010 0100 0000 0000 0000 -> 20424000
001000 00100 00100 0100 0000 0000 0000 -> 20844000
001000 00100 00100 0100 0000 0000 0000 -> 20844000
```

