原子操作是指操作是不可分割的，要么发生，要么不发生。事物只会处于原始状态和成功状态两种中的一种，不会处于一种完成一半的状态。

原子操作没规定两个核不可以同时执行该原子操作吧？比如一条汇编指令就是原子的操作，但这条指令可以被两个核同时执行。

TAS和CAS指令的原子性，我感觉不仅是原子操作而且还是排他的。

### C语言里的volatile

作用：

1. 保证可见性

   表示用到这个变量时必须每次都重新从内存里读取这个变量的值，而不是使用保存在寄存器里的备份。主要用于防止编译器的一些优化。

   ```
   //示例一
   #include <stdio.h>
   int main (void)
   {
   	int i = 10;
   	int a = i; //优化
   	int b = i;
    
   	printf ("i = %d\n", b);
   	return 0;
   }
   
   
   //示例二
   #include <stdio.h>
   int main (void)
   {
   	volatile int i = 10;
   	int a = i; //未优化
   	int b = i; //这里会继续从内存里读取i的值而不是用上面读过的缓存。
    
   	printf ("i = %d\n", b);
   	return 0;
   } 
   ```

2. 禁止指令重排（貌似只在Java里有这个功能）

   保证volatile所修饰的变量之前的代码不会在该变量之后执行，该变量之后的代码不会在该变量之前执行。

volatile不能保证原子性。仅使用 volatile 修饰 i，i++也不是线程安全的。

不知道C语言里的volatile是否和Java里的意义一样？

### TAS（Test And Set）

TAS指令的语义是：向某个内存地址写入值1，并且返回这块内存地址存的原始值。TAS指令是原子的，这是由实现TAS指令的硬件保证的（这里的硬件可以是CPU，也可以是实现了TAS的其他硬件）。

伪代码：

```
bool testAndSet(bool *target) {
		bool old = *target;
		*target = true;
		return old;
}
```

引用维基百科上的一段话：

In [computer science](https://en.wikipedia.org/wiki/Computer_science), the **test-and-set** instruction is an instruction used to write 1 (set) to a memory location and return its old value as a single [atomic](https://en.wikipedia.org/wiki/Atomic_(computer_science)) (i.e., non-interruptible) operation. **If multiple processes may access the same memory location, and if a process is currently performing a test-and-set, no other process may begin another test-and-set until the first process's test-and-set is finished.** A [CPU](https://en.wikipedia.org/wiki/Central_processing_unit) may use a test-and-set instruction offered by another electronic component, such as [dual-port RAM](https://en.wikipedia.org/wiki/DPRAM); a CPU itself may also offer a test-and-set instruction.

这段话消除了我的一个疑惑：对于同一个内存地址如果有一个线程正在执行test-and-set指令，那么其他线程需要等到它执行完成后才能继续执行test-and-set指令。这可以在硬件层面做到。具体是怎么做到的以后有空可以研究一下。

因此可以说TAS即是原子操作也是线程安全的，由硬件保证。

### CAS（Compare And Swap）

CAS的语义是：比较某个内存地址的值与一个给定值（**这个给定值是上一刻从此内存地址读出来的**），如果相等，则把一个新值写入到此内存地址，否则什么都不做。

CAS是项**乐观锁**技术，当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。

CAS指令需要三个参数，一个内存位置(V)、一个期望旧值(E)、一个新值(N)。

CAS指令的执行过程：

```
1.比较内存V的值是否与E相等？
2.如果相等，则用新值N替换内存位置V的旧值
3.如果不相等，不做任何操作。
4.无论哪个情况，CAS都会把内存V原来的值返回。
```

伪代码：

```
int CAS(int *ptr,int oldvalue,int newvalue)
{
   int temp = *ptr;
   if(temp == oldvalue)
       *ptr = newvalue;
   return temp;
}
```

另外一种是返回 bool 结果：

```
int cas(long *addr, long old, long new)
{
    /* Executes atomically. */
    if(*addr != old)
        return 0;
    *addr = new;
    return 1;
}
```

以上过程在CAS指令中是原子操作。

CAS应用示例：

```
do{  
   备份旧数据； 
   基于旧数据构造新数据； 
}while(!CAS( 内存地址，备份的旧数据，新数据 ))  //CAS修改失败则继续尝试，线程并不会被挂起阻塞。
```

**ABA 现象**

ABA 现象指的是线程 1 先读取了变量的值为A作为期望旧值，然后将要执行 CAS 指令时，变量的值在其他线程已经由 A 改为 B，又改回了 A。但是当线程 1 执行 CAS 时无法感知这一变化因为它判断是相等的所以依然执行成功。

ABA 现象不一定就会出问题，取决于具体的场景，在某些场景下 ABA 现象就会变成 ABA 问题。

解决的办法就是在变量前面加上版本号，每次变量更新的时候变量的**版本号都`+1`**，即`A->B->A`就变成了`1A->2B->3A`。添加了版本号后线程 1 读取到的 E 将是 1A，而当它执行 CAS 时重新获取变量的值将为 3A，于是不相等，CAS 将执行失败。

**疑问**

Q1：线程 1 将要执行 CAS(v, 1, 2)，线程 2 将要执行 CAS(v, 1, 2)，对于多核 CPU 会不会同时执行这两条CAS 指令？

A1：个人觉得对于同一变量的 CAS 指令，同一时刻CPU只会执行其中一条 CAS 操作。执行完其中一条后才会继续执行下一条。要不然一样导致结果不符合预期，线程不安全。

Q2：一条汇编指令能不能被多核 CPU 同时执行？

可以。但如果锁内存总线后，就可以保证同时只有一个核来执行该指令，从而避免了多核下发生的问题。

### 参考

[C++11中的原子操作 (Atomic Operation)](https://bitdewy.github.io/blog/2013/09/02/atomic-operation/)  只讲了使用没有讲原理。

[锁的硬件实现](https://juejin.im/post/5ce8f2a86fb9a07ee062f298)

[如何使用 C 语言中的 volatile 关键字?](https://steemit.com/cn/@cifer/c-volatile)  偏硬件层面对 volatile 的解释

[Test-and-set](https://en.wikipedia.org/wiki/Test-and-set)  Test-and-set维基百科介绍

CAS 相关：

[CAS原理分析及ABA问题详解](https://juejin.im/post/5c87afa06fb9a049f1550b04)  

[AtomicInteger、Unsafe类、ABA问题](https://blog.csdn.net/WSYW126/article/details/53979918)

[深入浅出CAS](https://www.jianshu.com/p/fb6e91b013cc)

[CAS操作在ARM和x86下的不同实现](https://blog.csdn.net/a7980718/article/details/82860505)  硬件汇编层面分析

[CH13-线程安全与锁优化](https://infilos.com/reading/RD02-VM/Dive-Into-Jvm/CH13.html)  一本书，还不错。

[多核环境下的内存屏障指令](https://blog.codingnow.com/2007/12/fence_in_multi_core.html)

[x86 cache locking 的猜想](https://yemablog.com/posts/cache-locking)

