---
title: JAVA多线程--CAS
category: Java
tags: thread
---

> CAS: compare and set 比较并重置
> 并发中的一种乐观锁。
> **乐观锁**: 假设没有发生冲突，那么线程刚好可以进行操作，如果发生冲突，就重试直到成功。 Java 类库中 *ReenterLock*,还有各种以 Atomic开头的原子类都用到了CAS
> **悲观锁**: 假设冲突一定会发生。只有一个线程能够进来操作. *synchronized* 关键字就是这样的悲观锁

---------------------------------
## 什么是CAS

在多线程的内存模型中，每个线程都会有自己独立的内存。同时也会持有一份主存中变量的复制. 每个线程都会独立的对ta的复制进项修改等操作，线程结束后会把修改后的值写回到主存。并发的问题也因此而产生，**当前线程不知道主存中的值是否已经被其他线程修改过了**。

![]({{ site.url }}/images/java_thread/thread_cas_1.PNG)

那如果我们使用 **volatile** 关键字 能否解决这个问题呢？
我们知道关键字 **volatile**作用:
  - 防止重排序: JVM为了性能会对一些指令进行重新排序，在多线程下可能会造成一些问题。volatile禁止语义重排序。
  - 线程间共享变量的可见性： 每个线程对共享变量的修改都被立即写入主存，其他线程的复制会失效，会重新从主存中获取。

这样看起来可以保证线程之间数据的一致性了。
我们来看一个例子:

```java
public class Test {
    private static volatile int index;
    public static void main(String[] args) {
        index = index +1;
    }
}
```
在命令窗口运行下面命令，获取字节码的内容
```shell
javap -v Test.class > cmd.txt
```
打开cmd.txt
```c
0: getstatic     #2                  // Field index:I
3: iconst_1
4: iadd
5: putstatic     #2                  // Field index:I
8: return
```
字节码的执行顺序
1. getstatic: 获取类的静态字段，将其值压入栈顶
2. iconst_1: 常量1 进栈
3. iadd： 加法
4. putstatic：给静态字段赋值

我们可以看到 **加** 和 **赋值** 是两个不同的操作，在多线程下，可能会出现两个线程都进入这段指令，获取了主存中相同的值，进行了不同的操作，都尝试把操作后的值写回主存，从而造成数据不一致。

这时 CAS 就要发挥作用了。在写回主存时进行CAS，三个参数:
  - 当前主存中的值 V
  - 旧的预期值（也即在当前线程中值）A
  - 即将更新的值 B
仅当V和B 值相同时，才将B值真正写进去并返回true。否则什么都不做并返回false。

在代码中 如果CAS返回false，我们就需要重新读取主存的值A，直到CAS返回成功写入并返回true。


## CAS的在JAVA中的应用
ReentrantLock的lock就是结合**volatile**，CAS来实现的

ReenterLock.java 中内容类NonfairSync的lock方法
```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```
AbstractQueuedSynchronizer.java 的部分代码
```java
private volatile int state;
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long stateOffset;
stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
Unsafe: CAS的核心类，由本地方法(native)实现对内存的操作。
stateOffset: 变量state在内存中的偏移量，Unsafe可以通过内存偏移量来获取数据
state: 用volatile关键字修饰，初始值为0.

1. 线程进入ReenterLock 的lock方法，
2. 进入compareAndSetState方法，可以看到，期待state的值为 0（也即state的初始值）， 将要修改的state值为1，
3. unsafe.compareAndSwapInt(this, stateOffset, expect, update)，可以看到 **stateOffset**为主存中state的值，如果stateOffset也为0 （没有期待线程锁定），就把state set为1（当前线程已经占有）。
4. 其他线程进来想要修改state的值是，发现主存中的值为1，期待值为0，compareAndSwapInt返回false.
5. 返回false之后，执行acquire(1); 继续尝试获取1, acquire的具体代码里会不断for循环尝试。

unlock也是类似的操作把state重置为0。

--------------------------------------------
CAS中不需要休眠当前线程，减少了一些不必要线程上下文切换。能提供并发性能。但是如果长时间不能获取锁，会给CPU造成更大的开销。 所以要选择好使用场景，线程持有锁时间不长的场景会比较合适。
