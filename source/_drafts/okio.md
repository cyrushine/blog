Segment.limit 下一次写操作写入的位置
Segment.pos 下一次读操作读取的位置
正常情况下 0 <= pos <= limit <= max
Buffer 内部的 Segment 通过 Segment.prev 和 Segment.next 形成一个环（头尾相连的链表）
写入（write）的节点总是 tail 节点（head.prev），如果当前的 tail 不满足写入条件（Segment 不够容量 or 是个共享的 Segment），会插入一个新的 Segment 到 head.prev 作为当前 tail 来使用
读操作（read）总是从 head 开始，读操作会使 pos 逐步增长（往前走），当 pos 与 limit 相遇时（pos == limit），说明此节点的数据已读完，会从环里删除此节点，并将 next 作为新的 head
初始环是空的（null），第一次写操作会初始化第一个节点 head，此时 head.prev 和 head.next 都指向它自己

SegmentPool
当需要新的 Segment 时总是会从池里拿（Segment.take()），当一个 Segment 从环里移除不再需要时总是会放回到池里去（Segment.recycle(Segment)）
SegmentPool 是全局的，内部结构类似于 HashMap（数组 + 链表），只不过数组容量是固定的，通过 tid 决定用哪个链表

internal actual object SegmentPool {
  /**
   * The number of hash buckets. This number needs to balance keeping the pool small and contention
   * low. We use the number of processors rounded up to the nearest power of two. For example a
   * machine with 6 cores will have 8 hash buckets.
   */
  private val HASH_BUCKET_COUNT =
    Integer.highestOneBit(Runtime.getRuntime().availableProcessors() * 2 - 1)

  /**
   * Hash buckets each containing a singly-linked list of segments. We use multiple hash buckets so
   * different threads don't race each other. We use thread IDs as hash keys because they're handy,
   * and because it may increase locality.
   *
   * We don't use [ThreadLocal] because we don't know how many threads the host process has and we
   * don't want to leak memory for the duration of a thread's life.
   */
  private val hashBuckets: Array<AtomicReference<Segment?>> = Array(HASH_BUCKET_COUNT) {
    AtomicReference<Segment?>()
  }

  private fun firstRef(): AtomicReference<Segment?> {
    // Get a value in [0..HASH_BUCKET_COUNT).
    val hashBucket = (Thread.currentThread().id and (HASH_BUCKET_COUNT - 1L)).toInt()
    return hashBuckets[hashBucket]
  }
}  

回收时（recycle）插入链表头部，复用时（take）取链表头，为了适应多线程环境链表的节点是 AtomicReference<Segment?>，回收和复用都使用 CAS 操作
整个 SegmentPool 使用 CAS 替代锁（lock-free），竞争失败则放弃相关操作

/**
 * This class pools segments in a lock-free singly-linked stack. Though this code is lock-free it
 * does use a sentinel [LOCK] value to defend against races.
 *
 * When popping, a caller swaps the stack's next pointer with the [LOCK] sentinel. If the stack was
 * not already locked, the caller replaces the head node with its successor.
 *
 * When pushing, a caller swaps the head with a new node whose successor is the replaced head.
 *
 * If operations conflict, segments are not pushed into the stack. A [recycle] call that loses a
 * race will not add to the pool, and a [take] call that loses a race will not take from the pool.
 * Under significant contention this pool will have fewer hits and the VM will do more GC and zero
 * filling of arrays.
 *
 * This tracks the number of bytes in each linked list in its [Segment.limit] property. Each element
 * has a limit that's one segment size greater than its successor element.
 */
internal actual object SegmentPool {

  @JvmStatic
  actual fun take(): Segment {
    val firstRef = firstRef()

    val first = firstRef.getAndSet(LOCK)
    when {
      first === LOCK -> {
        // We didn't acquire the lock. Don't take a pooled segment.
        return Segment()
      }
      first == null -> {
        // We acquired the lock but the pool was empty. Unlock and return a new segment.
        firstRef.set(null)
        return Segment()
      }
      else -> {
        // We acquired the lock and the pool was not empty. Pop the first element and return it.
        firstRef.set(first.next)
        first.next = null
        first.limit = 0
        return first
      }
    }
  }

  @JvmStatic
  actual fun recycle(segment: Segment) {
    require(segment.next == null && segment.prev == null)
    if (segment.shared) return // This segment cannot be recycled.

    val firstRef = firstRef()

    val first = firstRef.get()
    if (first === LOCK) return // A take() is currently in progress.
    val firstLimit = first?.limit ?: 0
    if (firstLimit >= MAX_SIZE) return // Pool is full.

    segment.next = first
    segment.pos = 0
    segment.limit = firstLimit + Segment.SIZE

    if (!firstRef.compareAndSet(first, segment)) segment.next = null
    // If we raced another operation: Don't recycle this segment.
  }
}

inputStream() 把 Buffer 作为输入流的源
outputStream() 把 Buffer 作为输出流的目的地
copyTo(OutputStream) 将 Buffer 里的数据拷贝一份到 OutputStream，不移动 Segment.pos（也即不会释放已读数据）
writeTo(OutputStream) 将 Buffer 里的数据写入到 OutputStream，会移动 Segment.pos
readFrom(InputStream) 将 InputStream 里的数据读到 Buffer 里

peek() 返回一个可以重复读的 Source（一般的 Source 跟 InputStream 一样是单向/流式的，不能重复读），但是这个 Buffer 作为 Source 的 backend，已读的数据 Source 也是无法读取的

copyTo(Buffer)
这个就比较有意思了，我们知道 Buffer 用 Segment 对数据进行分片管理（类似于内存的分页），这里并没有真正将数据拷贝一份到另一个 Buffer（要节约内存嘛），而是让另一个 Buffer 里的 Segment 与当前 Buffer 里的 Segment 共享用一个 ByteArray
新的 Segment 可以有自己的 pos 和 limit，可以读但不能写（Segment.owner == false，写入的数据会放到新创建的/自己的 Segment 里去），不然会把别人的数据搞乱，当然 Segment 的持有者是可以写的（Segment.owner == true）
共享了数据的 Segment 不能被 SegmentPool 回收（因为可能别人还在用，而且内部没有计算器指示被多少人持有），它的 Segment.share == true

copy()/clone() 同样是返回一个共享数据的 Buffer


### ByteString -> String

虽然 `okio.ByteString` 对标的是 `java.lang.String`，但是它操作的对象却是 String 内部的 ByteArray，相较于 String 的 “字符数据” 更贴近于 “字节数据”

| 类别                            | API             |
|---------------------------------|-----------------|
| 字符编码（charset）              | utf8()          |
|                                 | string(charset) |
| 消息摘要（message digest）       | `md5()` 128-bit |
|                                 | `hmacSha1(key)` 160-bit, `hmacSha256(key)` 256-bit, `hmacSha512(key)` 512-bit <br> hmac 是额外添加了一个秘钥作为影响因子的 sha |
|                                 | `sha1()` 160-bit, `sha256()` 256-bit, `sha512()` 512-bit                |
|                                 | digest(algorithm)                                                       |
| 基于字节的匹配和查找（而不是字符） | rangeEquals(offset, other ByteString/ByteArray, otherOffset, byteCount) |
|                                 | startsWith(ByteString/ByteArray)                                        |
|                                 | endsWith(ByteString/ByteArray)                                          |
|                                 | indexOf(other ByteString/ByteArray, fromIndex)                          |
|                                 | lastIndexOf(other ByteString/ByteArray, fromIndex)                      |
| 其他                            | hex()                                                                   |
|                                 | toAsciiLowercase()                                                      |
|                                 | toAsciiUppercase()                                                      |

