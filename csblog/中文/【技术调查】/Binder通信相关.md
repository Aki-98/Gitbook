# 议题：Binder断联的情况下，调用如何处理？

拓展阅读：[Android跨进程抛异常的原理的实现](https://cloud.tencent.com/developer/article/1741320)

情况1：执行方法的过程中Service端进程被杀死

- 结果：Client端触发DeadObjectException，如Client端未作异常处理会导致其Fatal Crash，调用丢失
- 对策：由Client端做异常处理，至少保证Client端不Crash

情况2：执行方法的过程中Service端进程抛出Aidl不支持的Exception

- 结果：Aidl方法在Client端返回默认值，Client端没有接收到异常，Service端不会Crash，Binder不会断联
- 对策：在Binder中不应抛出不支持跨进程处理的异常类型

情况3：执行方法的过程中Service端进程抛出Aidl支持的Exception

- 结果：Exception被转递给Client，在方法调用中可以catch Exception，调用丢失，Service端不会Crash，Client不shutdown的情况下Binder不会断联
- 对策：在Binder中不应抛出不支持跨进程处理的异常类型

情况4：执行方法前，Binder连接已经断联

- 结果：Client端触发DeadObjectException，如未作异常处理会导致Fatal Crash，调用丢失
- 对策：onServiceDisConnected回调中应该将Binder置空，所有对Binder的调用必须先判断Binder是否为null



# 议题：Binder支持在进程间转递的异常只有九种

1、跨进程通讯中，从一端到另外一端，只支持传递以下9种异常:

- SecurityException

- BadParcelableException

- IllegalArgumentException

- NullPointerException

- IllegalStateException

- NetworkOnMainThreadException

- UnsupportedOperationException

- ServiceSpecificException

- Parcelable的异常

2、对于不支持的异常，会在程序内部处理，可能导致崩溃，但不会传递给对方。常见的不支持的异常，

- 运行时异常：RuntimeException

- 算术异常类：ArithmeticExecption

- 类型强制转换异常：ClassCastException

- 数组下标越界异常：ArrayIndexOutOfBoundsException

- 文件未找到异常：FileNotFoundException

- 字符串转换为数字异常：NumberFormatException

- 输入输出异常：IOException

- 方法未找到异常：NoSuchMethodException

# 议题：为什么Binder通信只需要拷贝一次数据？

Linux中的进程是被隔离不能直接进行通信的，A进程向B进程传递数据，需要先将数据从A的用户空间拷贝到内核空间，然后从内核空间拷贝到B的用户空间。

![WMc6qL7vt9p](Binder通信相关_imgs\WMc6qL7vt9p.png)

Binder通信采用了mmap()函数，将内核空间和接收方用户空间的数据缓存区之间做了一层内存映射，这样，A进程向B进程传递数据，只需要从A的用户空间拷贝到B的用户空间数据缓存区在内核空间中的内存映射，就相当于直接拷贝到了B的用户空间。

由于Binder是双向通信，还会有A的用户空间数据缓存区在内核空间中的内存映射

![SWTymqDx5gh](Binder通信相关_imgs\SWTymqDx5gh.png)