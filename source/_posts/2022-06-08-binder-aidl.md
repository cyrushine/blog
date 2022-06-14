---
title: 深入 Binder 之 AIDL
date: 2022-06-08 12:00:00 +0800
---

应用层从 [AIDL](https://developer.android.com/guide/components/aidl) 说起，AIDL 全称 `Android Interface Definition Language`，是一个用以描述/定义 `接口` 的文本文件，有着与 java 类似的简单语法

下面是一个非常简单的 AIDL 文件，位于 `{projectPath}/app/src/main/aidl/work/dalvik/binder/example/IAidlExampleInterface.aidl`，一般不与 java/kotlin 代码同目录，它定义了一个返回所在进程 ID 的方法：`getPid()`

```java
// IAidlExampleInterface.aidl
package work.dalvik.binder.example;

// Declare any non-default types here with import statements

interface IAidlExampleInterface {

    int getPid();  // 获取进程 ID

}
```

上面所说的 `接口` 往往是 IPC 方法调用，当然也可以是本进程内的方法调用，实现这个接口的叫 `服务端`，那么调用接口的就是 `客户端`

使用 AIDL 时，服务端和客户端都有一份这个 AIDL 文件，执行一下 Android Studio 菜单：Build - Make Project，就会生成对应的 java 代码如下，主要有四个：

1. java 接口类 IAidlExampleInterface，包含了 AIDL 文件里定义的接口方法，相当于把 AIDL 翻译为 java 代码

2. Stub 类，服务端用，服务端的接口实现需要继承自 Stub 成为 Binder

3. Stub.asInterface() 方法，客户端用，将拿到的 IBinder 对象包装为 IAidlExampleInterface 对象，这样客户端才能调用 `getPid()` 方法

    1. 如果客户端和服务端在同一进程，asInterface() 返回服务端实现接口的那个对象

    2. 如果不在同一进程，需要 IPC，则返回 Proxy 实例，Proxy 实际上是把方法调用代理给客户端拿到的 IBinder 对象，也就是说此时是 IBinder 对象在实现接口

```java
public interface IAidlExampleInterface extends android.os.IInterface {
    public int getPid() throws android.os.RemoteException;

    public static abstract class Stub extends android.os.Binder implements work.dalvik.binder.example.IAidlExampleInterface {
        public static work.dalvik.binder.example.IAidlExampleInterface asInterface(android.os.IBinder obj) { ... }

        private static class Proxy implements work.dalvik.binder.example.IAidlExampleInterface {
            Proxy(android.os.IBinder remote) { mRemote = remote; }

            // ...
        }
        // ...
    }
    // ...
}
```

注册两个 Service，一个运行在 app 进程，一个运行在 `:child` 子进程，从日志可以看到：

1. app 进程 ID 为 13070，:child 子进程 ID 为 13122

2. 客户端和服务端同一进程的情况下，客户端拿到的 IBinder 对象和 asInterface() 返回的接口实现都是服务端里创建的 AidlExampleImpl 对象

3. 不在同一进程的情况下，客户端 asInterface() 返回的是 Proxy 对象，这个 Proxy 对象其实并没有实现接口逻辑，而是把方法调用代理至客户端拿到的 BinderProxy 对象

```kotlin
private class AidlExampleImpl: IAidlExampleInterface.Stub() {
    override fun getPid(): Int  = Process.myPid()
}

abstract class BaseService : Service() {
    private val mBinder = AidlExampleImpl()
    override fun onBind(intent: Intent): IBinder = mBinder
}

class ExampleService: BaseService()
class ChildService: BaseService()

<service android:name=".ChildService"
    android:enabled="true"
    android:exported="false"
    android:process=":child"
    />

<service android:name=".ExampleService"
    android:enabled="true"
    android:exported="false"
    />

val conn = object : ServiceConnection {
    override fun onServiceConnected(name: ComponentName, service: IBinder) {
        val binder = IAidlExampleInterface.Stub.asInterface(service)
        Log.d("cyrus", "my pid: ${Process.myPid()}, service pid: ${binder.pid}, ${binder.javaClass}, ${service.javaClass}")
    }
    override fun onServiceDisconnected(name: ComponentName?) {}
}
bindService(Intent(this, ExampleService::class.java), conn, Context.BIND_AUTO_CREATE)
bindService(Intent(this, ChildService::class.java), conn, Context.BIND_AUTO_CREATE)

// my pid: 13070, service pid: 13070, class work.dalvik.binder.example.AidlExampleImpl, class work.dalvik.binder.example.AidlExampleImpl
// my pid: 13070, service pid: 13122, class work.dalvik.binder.example.IAidlExampleInterface$Stub$Proxy, class android.os.BinderProxy
```

服务端的接口实现是继承自 Stub 的，`queryLocalInterface()` 和 `asInterface()` 返回的都是自己，所以同进程的情况下客户端拿到的 IBinder 和 asInterface() 返回的都是服务端对象

不同进程的情况下客户端拿到的 IBinder 是 BinderProxy，`BinderProxy.queryLocalInterface` 直接返回 null 导致 asInterface 返回 Proxy 实例，Proxy 实例把接口调用代理给 BinderProxy 实例，最终接口调用在 BinderProxy.transactNative() 进入 native 层

```java
public static work.dalvik.binder.example.IAidlExampleInterface asInterface(android.os.IBinder obj) {
  if ((obj==null)) {
    return null;
  }
  android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
  if (((iin!=null)&&(iin instanceof work.dalvik.binder.example.IAidlExampleInterface))) {  // 同进程
    return ((work.dalvik.binder.example.IAidlExampleInterface)iin);
  }
  return new work.dalvik.binder.example.IAidlExampleInterface.Stub.Proxy(obj);             // 不同进程
}

public class Binder implements IBinder {
    public @Nullable IInterface queryLocalInterface(@NonNull String descriptor) {
        if (mDescriptor != null && mDescriptor.equals(descriptor)) {
            return mOwner;
        }
        return null;
    }

    public void attachInterface(@Nullable IInterface owner, @Nullable String descriptor) {
        mOwner = owner;
        mDescriptor = descriptor;
    }            
}

// 服务端继承自 Stub 并将 mOwner 指向自己，将 mDescriptor 指向常量 DESCRIPTOR（接口的全限定名称）
public static abstract class Stub extends android.os.Binder implements work.dalvik.binder.example.IAidlExampleInterface {
  private static final java.lang.String DESCRIPTOR = "work.dalvik.binder.example.IAidlExampleInterface";

  /** Construct the stub at attach it to the interface. */
  public Stub() {
    this.attachInterface(this, DESCRIPTOR);
  }
}

// BinderProxy 主要是将方法调用转换为 transact 调用，类似于 RPC 的概念
public final class BinderProxy implements IBinder {
    public IInterface queryLocalInterface(String descriptor) {
        return null;
    }

    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        // ...
        try {
            return transactNative(code, data, reply, flags);
        } ...
    }

    // 不同进程的情况下，客户端的接口调用在这里进入 native 层
    public native boolean transactNative(int code, Parcel data, Parcel reply, int flags) throws RemoteException;     
}

// Proxy 如它的名称一样就是个代理模式，将所有方法调用代理至 remote
private static class Proxy implements work.dalvik.binder.example.IAidlExampleInterface
{
  private android.os.IBinder mRemote;
  Proxy(android.os.IBinder remote)
  {
    mRemote = remote;
  }

  // 把接口调用代理至 remote，也即是 BinderProxy
  @Override public int getPid() throws android.os.RemoteException
  {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    int _result;
    try {
      _data.writeInterfaceToken(DESCRIPTOR);
      boolean _status = mRemote.transact(Stub.TRANSACTION_getPid, _data, _reply, 0);
      if (!_status && getDefaultImpl() != null) {
        return getDefaultImpl().getPid();
      }
      _reply.readException();
      _result = _reply.readInt();
    }
    finally {
      _reply.recycle();
      _data.recycle();
    }
    return _result;
  }
}
```