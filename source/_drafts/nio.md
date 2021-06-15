java 层
FileInputStream.read(byte b[], int off, int len)
IoBridge.read(FileDescriptor fd, byte[] bytes, int byteOffset, int byteCount)
Linux.readBytes(FileDescriptor fd, byte[] bytes, int byteOffset, int byteCount)
cpp 层
Linux_readBytes(JNIEnv* env, jobject, jobject javaFd, jobject javaBytes, jint byteOffset, jint byteCount) libcore_io_Linux.cpp
系统调用
read(int fd, void *buf, size_t count)
fd 有个隐藏属性 offset，read 从 offset 开始读取 count 个字节的数据到 buf，offset 也会随着增长 count

FileInputStream.skip(long n)
相当于 read n 个字节然后把数据丢弃，让 fd offset 前进 n

同样的 FileOutputStream 对应系统调用 write

FileInputStream 和 FileOutputStream 都属于“流式” API，就像流水一样只能朝着一个方向读/写，不能后退

FileInputStream(File file)
IoBridge.open(name, O_RDONLY)
Libcore.os.open(path, flags, 0666)
Linux.open(String path, int flags, int mode)
Linux_open(JNIEnv* env, jobject, jstring javaPath, jint flags, jint mode)
open(pathname, flags, modes) 系统调用

系统调用 open 的参数 flags 必须包含三个访问模式（access modes）其中之一：O_RDONLY，O_WRONLY 和 O_RDWR

FileInputStream 相当于只读模式（open(path, O_RDONLY)）打开 fd

FileInputStream.close()
IoBridge.closeAndSignalBlockedThreads(fd)
Libcore.os.close(fd)
Linux.close(fd)
Linux_close(JNIEnv* env, jobject, jobject javaFd)
close(fd) 系统调用

FileInputStream 相当于以只读模式读文件：open(O_RDONLY) -> read -> close(fd)

FileOutputStream 相当于以只写模式写文件：open(O_WRONLY | O_CREAT | (append ? O_APPEND : O_TRUNC)) -> write -> close(fd)

O_CREAT：如果文件不存在则创建之
O_APPEND：以 APPEND 的模式打开文件（附加）
O_TRUNC：如果文件存在且以写模式打开，则把文件长度置为 0

RandomAccessFile(File file, String mode)
IoBridge.open(name, imode)

其中 mode 与 imode 之间的映射关系为 
r - O_RDONLY - 只读
rw - O_RDWR | O_CREAT - 读写

RandomAccessFile 提供了读写操作，相当于 FileInputStream 和 FileOutputStream 的组合

read(byte b[], int off, int len) -> 系统调用 read
write(byte b[], int off, int len) -> 系统调用 write

除此之外 RandomAccessFile 提供了 seek(long pos) 用以调整 fd.offset

RandomAccessFile.seek(long pos)
Libcore.os.lseek(fd, pos, SEEK_SET)
Linux.lseek(FileDescriptor fd, long offset, int whence)
Linux_lseek(JNIEnv* env, jobject, jobject javaFd, jlong offset, jint whence)
lseek(int fd, off_t offset, int whence) 系统调用

lseek(int fd, off_t offset, int whence) 改变已打开的文件描述的文件偏移（fd.offset，指示下一次读写的位置）
SEEK_SET - fd.offset = offset
SEEK_CUR - fd.offset += offset
SEEK_END - fd.offset = fd.end + offset

总之，传统的 java.io 包是基于 open, read, write, lseek, close 的

FileChannelImpl.read(ByteBuffer dst, long position)
IOUtil.read(FileDescriptor fd, ByteBuffer dst, long position, NativeDispatcher nd)
IOUtil.readIntoNativeBuffer(FileDescriptor fd, ByteBuffer bb, long position, NativeDispatcher nd)
FileDispatcherImpl.pread0(FileDescriptor fd, long address, int len, long position)
FileDispatcherImpl_pread0(JNIEnv *env, jclass clazz, jobject fdo, jlong address, jint len, jlong offset) // 进入 C 层
pread64(fd, buf, len, offset) // 系统调用，如果 position == -1 表示从头开始读取，对应 read(fd, *buf, count)

pread64/pread(fd, buf, len, offset) 从 fd 的 offset 开始读取最多 len 字节的数据到 buf 里，64 版本的 API 貌似是指支持大文件

FileChannel 的写操作对应 pwrite64 和 write

FileInputStream.read(byte b[], int off, int len)
static jint Linux_readBytes(JNIEnv* env, jobject, jobject javaFd, jobject javaBytes, jint byteOffset, jint byteCount) {
    ScopedBytesRW bytes(env, javaBytes);
    if (bytes.get() == NULL) {
        return -1;
    }
    return IO_FAILURE_RETRY(env, ssize_t, read, javaFd, bytes.get() + byteOffset, byteCount);
}
// 系统调用 read 中的 *buf 地址
class ScopedBytesRW : public ScopedBytes<false> {
public:
    ScopedBytesRW(JNIEnv* env, jobject object) : ScopedBytes<false>(env, object) {}
    jbyte* get() {
        return mPtr;
    }
};
    ScopedBytes(JNIEnv* env, jobject object)
    : mEnv(env), mObject(object), mByteArray(nullptr), mPtr(nullptr)
    {
        if (mObject == nullptr) {
            jniThrowNullPointerException(mEnv);
        } else {
            jclass byteArrayClass = JniConstants::GetPrimitiveByteArrayClass(env);
            if (mEnv->IsInstanceOf(mObject, byteArrayClass)) { // ByteArray 的话走此路
                mByteArray = reinterpret_cast<jbyteArray>(mObject);
                mPtr = mEnv->GetByteArrayElements(mByteArray, nullptr);
            } else { // ByteBuffer 走此路
                mPtr = reinterpret_cast<jbyte*>(mEnv->GetDirectBufferAddress(mObject));
            }
        }
    }

https://docs.oracle.com/javase/1.5.0/docs/guide/jni/spec/functions.html#wp17382

Get<PrimitiveType>ArrayElements Routines
NativeType *Get<PrimitiveType>ArrayElements(JNIEnv *env,
ArrayType array, jboolean *isCopy);

A family of functions that returns the body of the primitive array. The result is valid until the corresponding Release<PrimitiveType>ArrayElements() function is called. Since the returned array may be a copy of the Java array, changes made to the returned array will not necessarily be reflected in the original array until Release<PrimitiveType>ArrayElements() is called.

If isCopy is not NULL, then *isCopy is set to JNI_TRUE if a copy is made; or it is set to JNI_FALSE if no copy is made.

https://stackoverflow.com/questions/21691356/ndk-does-getbytearrayelements-copy-data-from-java-to-c/21693632

Get<Primitive>ArrayElements may or may not copy the data as it sees fit. The isCopy output parameter will tell you whether it has been copied. If data is not copied, then you have obtained a pointer to the data directly in the Dalvik heap. Read more here.

You always need to call the corresponding Release<Primitive>ArrayElements, regardless of whether a copy was made. Copying data back to the VM array isn't the only cleanup that might need to be done, although (according to the JNI documentation already linked) it is feasible that changes can be seen on the Java side before Release... has been called (iff data has not been copied).

I don't believe the VM is going to allow you to make the conversions that would be necessary to do what you are thinking. As I see it, either way you go, you will need to convert a byte array to a float or a float to a byte array in Java, which you cannot accomplish by type casting. The data is going to be copied at some point.

从 jni doc 和 StackOverflow 上的回答了解到
jbyte* GetByteArrayElements(jbyteArray array, jboolean* isCopy) 里的 isCopy 是输出参数，如果 isCopy == true 那么返回地址是 JVM heap ByteArray 在进程内存空间里的一个复制，后续需要调用 ReleaseByteArrayElements 将它回写到 JVM heap ByteArray 里
如果 isCopy == false 返回 JVM heap 里 ByteArray 的地址？
总的来说 GetByteArrayElements 和 ReleaseByteArrayElements 之间会可能会发生一次复制和一次回写

对于 java.nio.DirectByteBuffer，jni doc 是这么说的

https://docs.oracle.com/javase/1.5.0/docs/guide/jni/spec/functions.html#nio_support


The NIO-related entry points allow native code to access java.nio direct buffers. The contents of a direct buffer can, potentially, reside in native memory outside of the ordinary garbage-collected heap. 

jobject NewDirectByteBuffer(JNIEnv* env, void* address, jlong capacity);

Allocates and returns a direct java.nio.ByteBuffer referring to the block of memory starting at the memory address address and extending capacity bytes.

Native code that calls this function and returns the resulting byte-buffer object to Java-level code should ensure that the buffer refers to a valid region of memory that is accessible for reading and, if appropriate, writing. An attempt to access an invalid memory location from Java code will either return an arbitrary value, have no visible effect, or cause an unspecified exception to be thrown.

void* GetDirectBufferAddress(JNIEnv* env, jobject buf);

Fetches and returns the starting address of the memory region referenced by the given direct java.nio.Buffer.

This function allows native code to access the same memory region that is accessible to Java code via the buffer object.

也就是说 DirectByteBuffer 引用的内存地址 native 是可以直接访问的，因为它就在 native memory 而不是 JVM heap 里，这样一来在 native 读写 DirectByteBuffer 就比 ByteArray 要少复制和回写两步，性能上要提高不少

FileChannel.lock()
FileChannel.lock(0, Long.MAX_VALUE, false)
FileChannelImpl.lock(long position, long size, boolean shared)
FileDispatcherImpl.lock(FileDescriptor fd, boolean blocking, long pos, long size, boolean shared)
FileDispatcherImpl.lock0(FileDescriptor fd, boolean blocking, long pos, long size, boolean shared)
FileDispatcherImpl_lock0(JNIEnv *env, jobject this, jobject fdo, jboolean block, jlong pos, jlong size, jboolean shared)
fcntl(int fd, int cmd, ... /* arg */ ) // 系统调用

fcntl - manipulate file descriptor
int fcntl(int fd, int cmd, ... /* arg */ );

Advisory record locking
       Linux implements traditional ("process-associated") UNIX record
       locks, as standardized by POSIX.  For a Linux-specific
       alternative with better semantics, see the discussion of open
       file description locks below.

       F_SETLK, F_SETLKW, and F_GETLK are used to acquire, release, and
       test for the existence of record locks (also known as byte-range,
       file-segment, or file-region locks).  The third argument, lock,
       is a pointer to a structure that has at least the following
       fields (in unspecified order).

           struct flock {
               ...
               short l_type;    /* Type of lock: F_RDLCK,
                                   F_WRLCK, F_UNLCK */
               short l_whence;  /* How to interpret l_start:
                                   SEEK_SET, SEEK_CUR, SEEK_END */
               off_t l_start;   /* Starting offset for lock */
               off_t l_len;     /* Number of bytes to lock */
               pid_t l_pid;     /* PID of process blocking our lock
                                   (set by F_GETLK and F_OFD_GETLK) */
               ...
           };

如果锁已被别的进程持有，F_SETLK 返回 -1 而 F_SETLKW 会阻塞直到锁被释放

F_RDLCK 读锁，共享锁，可以被多个进程持有
F_WRLCK 写锁，排它锁（包括读锁），只能被一个进程持有
F_UNLCK 释放锁

SEEK_SET 锁的 offset 由 l_start 决定
SEEK_CUR 锁的 offset = fd.offset + l_start
SEEK_END 锁的 offset = fd.end + l_start

所以 FileChannel.lock 就是通过 fcntl 获得一个文件锁，FileLock.release 则通过 F_UNLCK 释放锁

ByteBuffer

capacity 容量，固定不变的，在构造的时候就确定了
