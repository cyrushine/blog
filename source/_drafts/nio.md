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

