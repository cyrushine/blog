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