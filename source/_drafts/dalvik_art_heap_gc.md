## Dalvik

### Dalvik Heap

Dalvik 的堆空间分为 `Zygote Heap` 和 `Active Heap`

Android 系统的第一个 Dalvik 虚拟机是由 Zygote 进程创建的。应用程序进程是由 Zygote 进程 `fork` 出来的。也就是说应用程序进程使用了一种 **写时拷贝技术**（`COW`）来复制了 Zygote 进程的地址空间。这意味着一开始的时候，应用程序进程和 Zygote 进程共享了同一个用来分配对象的堆。然而，当 Zygote 进程或者应用程序进程对该堆进行写操作时，内核就会执行真正的拷贝操作，使得 Zygote 进程和应用程序进程分别拥有自己的一份拷贝。

拷贝是一件费时费力的事情。因此，为了尽量地避免拷贝，Dalvik 虚拟机将自己的堆划分为两部分。事实上，Dalvik 虚拟机的堆最初是只有一个的。也就是 Zygote 进程在启动过程中创建 Dalvik 虚拟机的时候，只有一个堆。但是当 Zygote 进程在 `fork` 第一个应用程序进程之前，会将已经使用了的那部分堆内存划分为一部分，还没有使用的堆内存划分为另外一部分。前者就称为 **Zygote 堆**，后者就称为 **Active 堆**。以后无论是 Zygote 进程，还是应用程序进程，当它们需要分配对象的时候，都在 `Active 堆` 上进行。这样就可以使得 `Zygote 堆` 尽可能少地被执行写操作，因而就可以减少执行写时拷贝的操作。在 `Zygote 堆` 里面分配的对象其实主要就是 Zygote 进程在启动过程中预加载的类、资源和对象了。这意味着这些预加载的类、资源和对象可以在 Zygote 进程和应用程序进程中做到长期共享。这样既能减少拷贝操作，还能减少对内存的需求。

### Mark-Sweep

顾名思义，`Mark-Sweep` （标记-回收）算法就是为 `Mark` 和 `Sweep`两个阶段进行垃圾回收：
1. `Mark` 阶段从 `GC ROOT`开始递归地标记出当前所有被引用的对象
2. `Sweep` 阶段负责回收那些没有被引用的对象

Davlik 虚拟机定义了四种类型的 GC

```cpp
/* Not enough space for an "ordinary" Object to be allocated. */  
/* 在堆上分配对象时内存不足 */
extern const GcSpec *GC_FOR_MALLOC;  
  
/* Automatic GC triggered by exceeding a heap occupancy threshold. */  
/* 在已分配内存达到一定量之后触发 */
extern const GcSpec *GC_CONCURRENT;  
  
/* Explicit GC via Runtime.gc(), VMRuntime.gc(), or SIGUSR1. */  
/* 应用程序调用 System.gc、VMRuntime.gc接口或者收到 SIGUSR1 信号时触发 */
extern const GcSpec *GC_EXPLICIT;  
  
/* Final attempt to reclaim memory before throwing an OOM. */  
/* 在准备抛 OOM 异常之前进行的最后努力而触发的 GC */
extern const GcSpec *GC_BEFORE_OOM;  
```

`GC_FOR_MALLOC`、`GC_CONCURRENT` 和 `GC_BEFORE_OOM` 三种类型的 GC 都是在分配对象的过程触发的：

1. Dalvik 虚拟机在堆上分配对象的时候，如果分配失败会首先尝试 `GC_FOR_MALLOC`
2. GC 后再次进行对象的分配，如果失败这时候就得考虑先将堆的当前大小设置为 Dalvik 虚拟机启动时指定的堆最大值，再进行内存分配了
3. 如果上一步内存分配还是失败，这时候就得出狠招 `GC_BEFORE_OOM` 将 **软引用** 也回收掉
4. 这是最后一次努力了，如果还是分配失败就抛出 `OOM`
5. 成功地在堆上分配到一个对象之后，就会检查 `Active Heap` 当前已经分配的内存，如果大于阀值就会唤醒 GC 线程进行 `GC_CONCURRENT`。这个阀值是一个比指定的堆最小空闲内存小 128K 的数值，也就是说当堆的空闲内不足时就会触发 `GC_CONCURRENT`



为了管理堆，Dalvik 虚拟机需要一些辅助数据结构，包括一个 `Card Table`、两个 `Heap Bitmap` 和一个 `Mark Stack`

### Heap Bitmap

堆的起始地址为 `base`，大小为 `maxSize`，由此我们就知道在堆上创建的对象的地址范围为 `[base, maxSize)`。 但是通过 C 库提供的 `mspace_malloc` 在堆分配内存时，得到的内存地址是以 8 bits 对齐的。这意味着我们只需要 `(maxSize / 8)` bits 来描述堆对象。结构体 `HeapBitmap` 的成员变量 `bits` 是一个类型为 `unsigned long` 的数组，也就是说数组中的每一个元素都可以描述 `sizeof(unsigned long)` 个对象的存活。在 32 位设备上，一个 `unsigned long` 占用 32 bits，这意味着需要一个大小为 `(maxSize / 8 / 32)` 的 `unsigned long` 数组来描述堆对象的存活。如果换成字节数来描述的话，就是说我们需要一块大小为 `(maxSize / 8 / 32) × 4` 的内存块来描述一个大小为 `maxSize` 的堆对象。

`Live Bitmap` 用来标记上一次 GC 时被引用的对象（也就是没有被回收的对象），`Mark Bitmap` 用来标记当前 GC Mark 阶段发现的被引用对象，那么在 `Live Bitmap` 标记为 1，但是在 `Mark Bitmap` 中标记为 0 的即为需要回收的对象

假设我们知道了一个对象的地址为 `ptr`，堆的起始地址为 `base`，那么就可以计算得到一个偏移值 `offset`。有了这个偏移值之后，可以计算得到用来描述该对象存活的 bit 位于 `HeapBitmap->bits` 的 `unsigned long` 数组的索引 `index`。有了这个 `index` 之后，我们就可以得到一个 `unsigned long` 值。接着再通过对象地址 `ptr` 的第 4 到第 8 位表示的数值为索引，在前面找到的 `unsigned long` 值取出相应的位，就可以得到该对象是否存活了。

相反，给出一个 `HeapBitmap->bits` 描述的 `unsigned long` 数组的索引 `index`，我们可以找到一个偏移值 `offset`，将这个偏移值加上堆的起始地址 `base`，就可以得到一个对象的地址 `ptr`

### Mark Stack

在 GC Mark 阶段，Dalvik 虚拟机能过 **递归方式** 来标记对象。但是这不是通过函数的递归调用来实现的，而是借助一个称为 `Mark Stack` 的栈来实现的。具体来说，当我们标记完 `GC ROOT` 之后，就按照它们的地址从小到大的顺序标记它们所引用的其它对象。

假设有 A、B、C 和 D 四个对象，它的地址大小关系为 A < B < C < D，其中 B 和 D 是 `GC ROOT`，A 被 D 引用，C 没有被 B 和 D 引用。那么我们将依次遍历 B 和 D，当遍历到 B 的时候没有发现它引用其它对象，然后就继续向前遍历 D 对象，发现它引用了 A 对象。按照递归的算法，这时候除了标记 A 对象是正在使用之外，还应该去检查 A 对象有没有引用其它对象，然后又再检查它引用的对象有没有又引用其它的对象，一直这样遍历下去，这样就跟函数递归一样。（树的 **深度优先** 遍历算法）

更好的做法是将对象 A 记录在一个 `Mark Stack` 中，然后继续检查地址值比对象 D 大的其它对象。对于地址值比对象 D 大的其它对象，如果它们引用了一个地址值比它们小的其它对象，那么这些其它对象同样要记录在 `Mark Stack` 中。等到该轮检查结束之后，再回过头来检查记录在 `Mark Stack`里面的对象。然后又重复上述过程，直到 `Mark Stack`等于空为止。（树的 **广度优先** 遍历算法）

### Concurrent GC

在 GC Mark 阶段，要求除了 GC 线程之外其它的线程都停止，否则的话就会可能导致不能正确地标记每一个对象。这种现象在 GC 中称为 `Stop The World`，会导致程序中止执行，造成停顿的现象。为了尽可能地减少停顿，我们必须要允许在 Mark 阶段有条件地允许程序的其它线程执行。这种 GC 称为 **并行垃圾收集算法**（`Concurrent GC`）。

为了实现 `Concurrent GC`，Mark 阶段又划分两个子阶段：
1. 第一个子阶段只负责标记根集对象（`GC ROOT`）。所谓的根集对象，就是指在 GC 开始的瞬间，被全局变量、栈变量和寄存器等引用的对象
2. 有了这些根集变量之后，我们就可以顺着它们找到其余的被引用变量。例如，一个栈变量引了一个对象，而这个对象又通过成员变量引用了另外一个对象，那该被引用的对象也会同时标记为正在使用。这个 **标记被根集对象引用的对象** 的过程就是第二个子阶段

#### Card Table

在 `Concurrent GC`，第一个子阶段是不允许 GC 之外的线程运行的，但是第二个子阶段是允许的。不过，在第二个子阶段执行的过程中，如果一个线程修改了一个对象，那么该对象必须要记录起来，因为它很有可能引用了新的对象，或者引用了之前未引用过的对象。如果不这样做的话，那么就会导致被引用对象还在使用然而却被回收。这种情况出现在只进行部分 GC 的情况，这时候 `Card Table` 的作用就是用来记录非垃圾收集堆对象对垃圾收集堆对象的引用。Dalvik 虚拟机进行部分 GC 时实际上就是只收集在 `Active Heap` 上分配的对象。因此对 Dalvik 虚拟机来说，`Card Table` 就是用来记录在 `Zygote Heap` 上分配的对象在部分 GC 执行过程中对在 `Active Heap` 上分配的对象的引用。

`Card Table` 由 `Card` 组成，一个 `Card` 实际上就是一个字节，它的值要么是 `CLEAN` 要么是 `DIRTY`。如果一个 `Card` 的值是 `CLEAN`，就表示与它对应的对象在 Mark 第二子阶段没有被程序修改过。否则的话就意味着被程序修改过，对于这些被修改过的对象。需要在 Mark第二子阶段结束之后，再次禁止 GC 之外线程执行，以便 GC 线程再次根据 `Card Table` 记录的信息对被修改过的对象引用的其它对象进行重新标记。由于 Mark 第二子阶段执行的时间不会太长，因此在该阶段被修改的对象不会很多，这样就可以保证第二次子阶段结束后，再次执行标记对象的过程是很快的，因而此时对程序造成的停顿非常小。

在 `Card Table` 中，在连续 `GC_CARD_SIZE` 地址中的对象共用一个 `Card`。Dalvik 虚拟机将 `GC_CARD_SIZE` 的值设置为 128。因此，假设堆的大小为 `Max Heap Size`，那么我们只需要一块字节数为 `(Max Heap Size / 128)` 的 `Card Table`。相比大小为 `(Max Heap Size / 8 / 32) × 4` 的 `HeapBitmap` 减少了一半的内存需求



## ART

### ART Heap

`ART` 的堆分为四个部分：`Image Space`、`Zygote Space`、`Allocation Space` 和 `Large Object Space`

其中 `Image Space`、`Zygote Space` 和 `Allocation Space` 是在地址上连续的空间，称为 `Continuous Space`，而 `Large Object Space` 是一些离散地址的集合，用来分配一些大对象，称为 `Discontinuous Space`

`Image Space` 空间包含了那些需要预加载的系统类对象

`Zygote Space` 和 `Allocation Space` 与 Dalvik 虚拟机垃圾收集机制中的 Zygote 堆和 Active 堆的作用是一样的。`Zygote Space` 在 Zygote 进程和应用程序进程之间共享的，而 `Allocation Space` 则是每个进程独占的。同样的，Zygote 进程一开始只有一个 `Image Space` 和一个 `Zygote Space`。在 Zygote 进程 `fork` 第一个子进程之前，就会把 `Zygote Space` 一分为二，原来的已经被使用的那部分堆还叫 `Zygote Space`，而未使用的那部分堆就叫 `Allocation Space`。以后的对象都在 `Allocation Space` 上分配。


## 参考

[Dalvik虚拟机垃圾收集机制简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/41338251)
[ART运行时垃圾收集机制简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/42072975)