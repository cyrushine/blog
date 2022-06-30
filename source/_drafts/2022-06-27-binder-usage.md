---
title: 深入 Binder 之
date: 2022-06-27 12:00:00 +0800
---

# startActivity

1. `startActivity` 最终走到 `IActivityTaskManager.startActivity`，[AIDL](../../../../2022/06/08/binder-aidl/) 的相关知识让我们可以从名字上看出这是一个 binder ipc 调用，会有一个 `IActivityTaskManager.aidl` 文件，编译出对应的 Interface 和 Stub 类，通过 `IActivityTaskManager.Stub.asInterface()` 获得接口实现

2. 但在此之前需要通过 `ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE)` 获得 IActivityTaskManager 服务（Binder）

3.  ServiceManager 内部是通过 `IServiceManager.getService` 来查找各种服务的，那 IServiceManager 服务的 Binder 又从哪里获得呢？答案是 `BinderInternal.getContextObject()`
    - 它返回一个 `BinderProxy` java 对象，所有的 `IBinder` 操作都由 `BinderProxy.mNativeData` 指向的 native 层 `BpBinder` 实例来实现
	- native 层也有 `IBinder` 接口对应 java 层的 `IBinder` 接口，BpBinder 实现了 IBinder 负责向 binder driver 传输数据
	- 特别之处在于 IServiceManager 对应的 BpBinder 它的 `mHandle` 是 0

4. 通过 `IServiceManager.Stub.asInterface(remote)` 将 BinderProxy(IBinder) 包装为 IServiceManager 实现，但所有接口实现（IPC 操作）最终都由 native 层的 BpBinder 实现：`boolean IBinder.transact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags)` 对应 `status_t BpBinder::transact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)`

```java
ContextImpl.startActivity(Intent intent, Bundle options)
Instrumentation.execStartActivity(
	Context who, IBinder contextThread, IBinder token, Activity target, 
	Intent intent, int requestCode, Bundle options) {
	// ...
    // IActivityTaskManager.startActivity(...)
    int result = ActivityTaskManager.getService().startActivity(whoThread,
            who.getOpPackageName(), who.getAttributionTag(), intent,
            intent.resolveTypeIfNeeded(who.getContentResolver()), token,
            target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);	
}

public class ActivityTaskManager {
    public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
    }

    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    // AIDL 那篇讲过同进程直接返回 Stub 实现类，跨进程是返回 Stub.Proxy
                    // Stub.Proxy 把所有方法调用转发给 binder 进行 IPC，也就是这个 b
                    // 所以重要的是看 IBinder b 是怎么获得的、是如何实现的
                    return IActivityTaskManager.Stub.asInterface(b);
                }
            };	
}

public class ServiceManager {
    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return Binder.allowBlocking(rawGetService(name));
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }

    private static IBinder rawGetService(String name) throws RemoteException {
        final IBinder binder = getIServiceManager().getService(name);  // IServiceManager.getService(name)
		// ...
        return binder;
    }	

    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative
                .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
        return sServiceManager;
    }		
}

public final class ServiceManagerNative {
    /**
     * Cast a Binder object into a service manager interface, generating
     * a proxy if needed.
     *
     * TODO: delete this method and have clients use
     *     IServiceManager.Stub.asInterface instead
     */
    @UnsupportedAppUsage
    public static IServiceManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }

        // ServiceManager is never local
        return new ServiceManagerProxy(obj);
    }
}

// This class should be deleted and replaced with IServiceManager.Stub whenever
// mRemote is no longer used
class ServiceManagerProxy implements IServiceManager {
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
        mServiceManager = IServiceManager.Stub.asInterface(remote);
    }

    @UnsupportedAppUsage
    public IBinder getService(String name) throws RemoteException {
        // getService(name) 最终由 IServiceManager.Stub.asInterface(remote) 实现
        // 这熟悉的签名就知道又是代理至 IBinder remote
        // 而这个 remote 是从 BinderInternal.getContextObject() 获得的
        return mServiceManager.checkService(name);
    }
}
```

```cpp
public class BinderInternal {
    /**
     * Return the global "context object" of the system.  This is usually
     * an implementation of IServiceManager, which you can use to find
     * other services.
     */
    @UnsupportedAppUsage
    public static final native IBinder getContextObject();	
}

// android_util_Binder.cpp
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);  // 得到一个 handle == 0 的 BpBinder
    return javaObjectForIBinder(env, b);                           // 创建并返回 java BinderProxy 对象
}

// ProcessState.cpp
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    sp<IBinder> context = getStrongProxyForHandle(0);

    if (context) {
        // The root object is special since we get it directly from the driver, it is never
        // written by Parcell::writeStrongBinder.
        internal::Stability::markCompilationUnit(context.get());
    } else {
        ALOGW("Not able to get context object on %s.", mDriverName.c_str());
    }

    return context;
}

sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)  // handle = 0
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

    if (handle == 0 && the_context_object != nullptr) return the_context_object;

    handle_entry* e = lookupHandleLocked(handle);

    if (e != nullptr) {
        // We need to create a new BpBinder if there isn't currently one, OR we
        // are unable to acquire a weak reference on this current one.  The
        // attemptIncWeak() is safe because we know the BpBinder destructor will always
        // call expungeHandle(), which acquires the same lock we are holding now.
        // We need to do this because there is a race condition between someone
        // releasing a reference on this BpBinder, and a new reference on its handle
        // arriving from the driver.
        IBinder* b = e->binder;
        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                // Special case for context manager...
                // The context manager is the only object for which we create
                // a BpBinder proxy without already holding a reference.
                // Perform a dummy transaction to ensure the context manager
                // is registered before we create the first local reference
                // to it (which will occur when creating the BpBinder).
                // If a local reference is created for the BpBinder when the
                // context manager is not present, the driver will fail to
                // provide a reference to the context manager, but the
                // driver API does not return status.
                //
                // Note that this is not race-free if the context manager
                // dies while this code runs.

                IPCThreadState* ipc = IPCThreadState::self();

                CallRestriction originalCallRestriction = ipc->getCallRestriction();
                ipc->setCallRestriction(CallRestriction::NONE);

                Parcel data;
                status_t status = ipc->transact(
                        0, IBinder::PING_TRANSACTION, data, nullptr, 0);

                ipc->setCallRestriction(originalCallRestriction);

                if (status == DEAD_OBJECT)
                   return nullptr;
            }

            sp<BpBinder> b = BpBinder::PrivateAccessor::create(handle);  // 创建 handle == 0 的 BpBinder
            e->binder = b.get();
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            // This little bit of nastyness is to allow us to add a primary
            // reference to the remote proxy when this team doesn't have one
            // but another team is sending the handle to us.
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}

class PrivateAccessor {
    static sp<BpBinder> create(int32_t handle) { return BpBinder::create(handle); }
}

sp<BpBinder> BpBinder::create(int32_t handle) {  // handle == 0
    int32_t trackedUid = -1;
    if (sCountByUidEnabled) {
        trackedUid = IPCThreadState::self()->getCallingUid();
        AutoMutex _l(sTrackingLock);
        uint32_t trackedValue = sTrackingMap[trackedUid];
        if (CC_UNLIKELY(trackedValue & LIMIT_REACHED_MASK)) {
            if (sBinderProxyThrottleCreate) {
                return nullptr;
            }
            trackedValue = trackedValue & COUNTING_VALUE_MASK;
            uint32_t lastLimitCallbackAt = sLastLimitCallbackMap[trackedUid];

            if (trackedValue > lastLimitCallbackAt &&
                (trackedValue - lastLimitCallbackAt > sBinderProxyCountHighWatermark)) {
                ALOGE("Still too many binder proxy objects sent to uid %d from uid %d (%d proxies "
                      "held)",
                      getuid(), trackedUid, trackedValue);
                if (sLimitCallback) sLimitCallback(trackedUid);
                sLastLimitCallbackMap[trackedUid] = trackedValue;
            }
        } else {
            if ((trackedValue & COUNTING_VALUE_MASK) >= sBinderProxyCountHighWatermark) {
                ALOGE("Too many binder proxy objects sent to uid %d from uid %d (%d proxies held)",
                      getuid(), trackedUid, trackedValue);
                sTrackingMap[trackedUid] |= LIMIT_REACHED_MASK;
                if (sLimitCallback) sLimitCallback(trackedUid);
                sLastLimitCallbackMap[trackedUid] = trackedValue & COUNTING_VALUE_MASK;
                if (sBinderProxyThrottleCreate) {
                    ALOGI("Throttling binder proxy creates from uid %d in uid %d until binder proxy"
                          " count drops below %d",
                          trackedUid, getuid(), sBinderProxyCountLowWatermark);
                    return nullptr;
                }
            }
        }
        sTrackingMap[trackedUid]++;
    }
    return sp<BpBinder>::make(BinderHandle{handle}, trackedUid);  // 创建 handle == 0 的 BpBinder
}

// If the argument is a JavaBBinder, return the Java object that was used to create it.
// Otherwise return a BinderProxy for the IBinder. If a previous call was passed the
// same IBinder, and the original BinderProxy is still alive, return the same BinderProxy.
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    // N.B. This function is called from a @FastNative JNI method, so don't take locks around
    // calls to Java code or block the calling thread for a long time for any reason.

    if (val == NULL) return NULL;

    if (val->checkSubclass(&gBinderOffsets)) {
        // It's a JavaBBinder created by ibinderForJavaObject. Already has Java object.
        jobject object = static_cast<JavaBBinder*>(val.get())->object();
        LOGDEATH("objectForBinder %p: it's our own %p!\n", val.get(), object);
        return object;
    }

    BinderProxyNativeData* nativeData = new BinderProxyNativeData();
    nativeData->mOrgue = new DeathRecipientList;
    nativeData->mObject = val;

	// 调用 java 层代码：android.os.BinderProxy#getInstance(long nativeData, long iBinder) 得到一个 BinderProxy 对象
    jobject object = env->CallStaticObjectMethod(gBinderProxyOffsets.mClass,
            gBinderProxyOffsets.mGetInstance, (jlong) nativeData, (jlong) val.get());
    if (env->ExceptionCheck()) {
        // In the exception case, getInstance still took ownership of nativeData.
        return NULL;
    }
    BinderProxyNativeData* actualNativeData = getBPNativeData(env, object);
    if (actualNativeData == nativeData) {
        // Created a new Proxy
        uint32_t numProxies = gNumProxies.fetch_add(1, std::memory_order_relaxed);
        uint32_t numLastWarned = gProxiesWarned.load(std::memory_order_relaxed);
        if (numProxies >= numLastWarned + PROXY_WARN_INTERVAL) {
            // Multiple threads can get here, make sure only one of them gets to
            // update the warn counter.
            if (gProxiesWarned.compare_exchange_strong(numLastWarned,
                        numLastWarned + PROXY_WARN_INTERVAL, std::memory_order_relaxed)) {
                ALOGW("Unexpectedly many live BinderProxies: %d\n", numProxies);
            }
        }
    } else {
        delete nativeData;
    }

    return object;  // 返回 java BinderProxy 对象
}
```

# Parcel

```java
// Item.java
public class Item implements Parcelable {

    private int id;
    private String title;
    private String category;
    private String mp3url;
    private String lrcurl;
    private String txturl;

    public Item() {}

    public Item(Parcel in) {
        id = in.readInt();
        title = in.readString();
        category = in.readString();
        mp3url = in.readString();
        lrcurl = in.readString();
        txturl = in.readString();
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(id);
        dest.writeString(title);
        dest.writeString(category);
        dest.writeString(mp3url);
        dest.writeString(lrcurl);
        dest.writeString(txturl);
    }

    public static final Creator<Item> CREATOR = new Creator<Item>() {
        @Override
        public Item createFromParcel(Parcel in) {
            return new Item(in);
        }

        @Override
        public Item[] newArray(int size) {
            return new Item[size];
        }
    };

    // getter and setter ...
}

// Item.aidl
package work.dalvik.binder.example;
parcelable Item;

// IAidlExampleInterface.aidl
package work.dalvik.binder.example;

import work.dalvik.binder.example.Item;

interface IAidlExampleInterface {
    Item fetchFromDB(in Item query, int size, long from, boolean distint);
}

// IAidlExampleInterface.java
public interface IAidlExampleInterface extends android.os.IInterface {
    public static abstract class Stub extends android.os.Binder implements work.dalvik.binder.example.IAidlExampleInterface {
        private static class Proxy implements work.dalvik.binder.example.IAidlExampleInterface {
            @Override 
            public work.dalvik.binder.example.Item fetchFromDB(work.dalvik.binder.example.Item query, int size, long from, boolean distint) throws android.os.RemoteException
            {
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            work.dalvik.binder.example.Item _result;
            try {
              _data.writeInterfaceToken(DESCRIPTOR);
              if ((query!=null)) {
                _data.writeInt(1);
                query.writeToParcel(_data, 0);
              }
              else {
                _data.writeInt(0);
              }
              _data.writeInt(size);
              _data.writeLong(from);
              _data.writeInt(((distint)?(1):(0)));
              boolean _status = mRemote.transact(Stub.TRANSACTION_fetchFromDB, _data, _reply, 0);
              if (!_status && getDefaultImpl() != null) {
                return getDefaultImpl().fetchFromDB(query, size, from, distint);
              }
              _reply.readException();
              if ((0!=_reply.readInt())) {
                _result = work.dalvik.binder.example.Item.CREATOR.createFromParcel(_reply);
              }
              else {
                _result = null;
              }
            }
            finally {
              _reply.recycle();
              _data.recycle();
            }
            return _result;
            }
        }
    }
}
```

# BinderProxy

> std::variant 是 C++ 17 所提供的变体类型，`variant<X, Y, Z>` 是可存放 X, Y, Z 这三种类型数据的变体类型<br>
> 
> 与C语言中传统的 union 类型相同的是，variant 也是联合(union)类型。即 variant 可以存放多种类型的数据，但任何时刻最多只能存放其中一种类型的数据
> 与C语言中传统的 union 类型所不同的是，variant 是可辨识的类型安全的联合(union)类型。即 variant 无须借助外力只需要通过查询自身就可辨别实际所存放数据的类型<br>
> 
> `v = variant<int, double, std::string>`，则 v 是一个可存放 int, double, std::string 这三种类型数据的变体类型的对象<br>
> 
> `v.index()`                    返回变体类型 v 实际所存放数据的类型的下标，变体中第 1 种类型下标为 0，第 2 种类型下标为 1，以此类推  
> `std::holds_alternative<T>(v)` 可查询变体类型 v 是否存放了 T 类型的数据  
> `std::get<I>(v)`               如果变体类型 v 存放的数据类型下标为 I，那么返回所存放的数据，否则报错  
> `std::get_if<I>(&v)`           如果变体类型 v 存放的数据类型下标为 I，那么返回所存放数据的指针，否则返回空指针  
> `std::get<T>(v)`               如果变体类型 v 存放的数据类型为 T，那么返回所存放的数据，否则报错  
> `std::get_if<T>(&v)`           如果变体类型 v 存放的数据类型为 T，那么返回所存放数据的指针，否则返回空指针

```cpp
public final class BinderProxy implements IBinder {
    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        // ...
        return transactNative(code, data, reply, flags);
    }

    public native boolean transactNative(int code, Parcel data, Parcel reply, int flags) throws RemoteException;	
}

// frameworks/base/core/jni/android_util_Binder.cpp
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    if (dataObj == NULL) {
        jniThrowNullPointerException(env, NULL);
        return JNI_FALSE;
    }

    Parcel* data = parcelForJavaObject(env, dataObj);
    if (data == NULL) {
        return JNI_FALSE;
    }
    Parcel* reply = parcelForJavaObject(env, replyObj);
    if (reply == NULL && replyObj != NULL) {
        return JNI_FALSE;
    }

    // 取 BinderProxy.mNativeData 的值，它是一个指向 BinderProxyNativeData 的指针
	// BinderProxyNativeData.mObject 是上一章节介绍的、mHandle == 0 的 BpBinder
    IBinder* target = getBPNativeData(env, obj)->mObject.get();
    if (target == NULL) {
        jniThrowException(env, "java/lang/IllegalStateException", "Binder has been finalized!");
        return JNI_FALSE;
    }

    ALOGV("Java code calling transact on %p in Java object %p with code %" PRId32 "\n",
            target, obj, code);


    bool time_binder_calls;
    int64_t start_millis;
    if (kEnableBinderSample) {
        // Only log the binder call duration for things on the Java-level main thread.
        // But if we don't
        time_binder_calls = should_time_binder_calls();

        if (time_binder_calls) {
            start_millis = uptimeMillis();
        }
    }

    //printf("Transact from Java code to %p sending: ", target); data->print();
    status_t err = target->transact(code, *data, reply, flags);  // BpBinder::transact
    //if (reply) printf("Transact from Java code to %p received: ", target); reply->print();

    if (kEnableBinderSample) {
        if (time_binder_calls) {
            conditionally_log_binder_call(start_millis, target, code);
        }
    }

    if (err == NO_ERROR) {
        return JNI_TRUE;
    } else if (err == UNKNOWN_TRANSACTION) {
        return JNI_FALSE;
    }

    signalExceptionForError(env, obj, err, true /*canThrowRemoteException*/, data->dataSize());
    return JNI_FALSE;
}

// frameworks/native/libs/binder/BpBinder.cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        bool privateVendor = flags & FLAG_PRIVATE_VENDOR;
        // don't send userspace flags to the kernel
        flags = flags & ~static_cast<uint32_t>(FLAG_PRIVATE_VENDOR);

        // user transactions require a given stability level
        if (code >= FIRST_CALL_TRANSACTION && code <= LAST_CALL_TRANSACTION) {
            using android::internal::Stability;

            int16_t stability = Stability::getRepr(this);
            Stability::Level required = privateVendor ? Stability::VENDOR
                : Stability::getLocalLevel();

            if (CC_UNLIKELY(!Stability::check(stability, required))) {
                ALOGE("Cannot do a user transaction on a %s binder (%s) in a %s context.",
                      Stability::levelString(stability).c_str(),
                      String8(getInterfaceDescriptor()).c_str(),
                      Stability::levelString(required).c_str());
                return BAD_TYPE;
            }
        }

        status_t status;
        if (CC_UNLIKELY(isRpcBinder())) {  // false
            status = rpcSession()->transact(sp<IBinder>::fromExisting(this), code, data, reply, flags);
        } else {  // ServiceManager 对应的 BpBinder.mHandle 是 BinderHandle 且其 handle == 0
            status = IPCThreadState::self()->transact(binderHandle(), code, data, reply, flags);
        }
        if (data.dataSize() > LOG_TRANSACTIONS_OVER_SIZE) {
            Mutex::Autolock _l(mLock);
            ALOGW("Large outgoing transaction of %zu bytes, interface descriptor %s, code %d",
                  data.dataSize(),
                  mDescriptorCache.size() ? String8(mDescriptorCache).c_str()
                                          : "<uncached descriptor>",
                  code);
        }

        if (status == DEAD_OBJECT) mAlive = 0;

        return status;
    }

    return DEAD_OBJECT;
}

// 判断 mHandle 是否是 RpcHandle
bool BpBinder::isRpcBinder() const {
    return std::holds_alternative<RpcHandle>(mHandle);
}

// frameworks/native/libs/binder/include/binder/BpBinder.h
class BpBinder : public IBinder
{
private:
    using Handle = std::variant<BinderHandle, RpcHandle>;

    BpBinder(BinderHandle&& handle, int32_t trackedUid);
    Handle mHandle;	
}

// frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    LOG_ALWAYS_FATAL_IF(data.isForRpc(), "Parcel constructed for RPC, but being used with binder.");

    status_t err;

    flags |= TF_ACCEPT_FDS;

    IF_LOG_TRANSACTIONS() {
        TextOutput::Bundle _b(alog);
        alog << "BC_TRANSACTION thr " << (void*)pthread_self() << " / hand "
            << handle << " / code " << TypeCode(code) << ": "
            << indent << data << dedent << endl;
    }

    LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),
        (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);

    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }

    if ((flags & TF_ONE_WAY) == 0) {  // 一般为 false
        if (UNLIKELY(mCallRestriction != ProcessState::CallRestriction::NONE)) {
            if (mCallRestriction == ProcessState::CallRestriction::ERROR_IF_NOT_ONEWAY) {
                ALOGE("Process making non-oneway call (code: %u) but is restricted.", code);
                CallStack::logStack("non-oneway call", CallStack::getCurrent(10).get(),
                    ANDROID_LOG_ERROR);
            } else /* FATAL_IF_NOT_ONEWAY */ {
                LOG_ALWAYS_FATAL("Process may not make non-oneway calls (code: %u).", code);
            }
        }

        #if 0
        if (code == 4) { // relayout
            ALOGI(">>>>>> CALLING transaction 4");
        } else {
            ALOGI(">>>>>> CALLING transaction %d", code);
        }
        #endif
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
        #if 0
        if (code == 4) { // relayout
            ALOGI("<<<<<< RETURNING transaction 4");
        } else {
            ALOGI("<<<<<< RETURNING transaction %d", code);
        }
        #endif

        IF_LOG_TRANSACTIONS() {
            TextOutput::Bundle _b(alog);
            alog << "BR_REPLY thr " << (void*)pthread_self() << " / hand "
                << handle << ": ";
            if (reply) alog << indent << *reply << dedent << endl;
            else alog << "(none requested)" << endl;
        }
    } else {
        err = waitForResponse(nullptr, nullptr);
    }

    return err;
}

// [深入 Binder 之架构篇]介绍过，BC_TRANSACTION 表示 client 向 binder driver 传输数据，结构是 binder_transaction_data

status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,  // cmd == BC_TRANSACTION
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)    // handle == 0
{
    binder_transaction_data tr;  // 填充数据结构 binder_transaction_data

    tr.target.ptr = 0;
    tr.target.handle = handle;   // 目标进程在 binder driver 的标识符，handle == 0 表示 service manager 进程
                                 // binder driver 作为中间商负责查找进程，并将数据传递过去
    tr.code = code;              // 指令码，目标进程根据协商好的约定执行相应的逻辑，定义在 AIDL 里的方法按顺序从上到下编码
                                 // 第一个是 FIRST_CALL_TRANSACTION + 0，第二个是 FIRST_CALL_TRANSACTION + 1
    tr.flags = binderFlags;      // TF_ONE_WAY (this is a one-way call: async, no return), TF_ROOT_OBJECT (contents are the component's root object) 等等
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }

    mOut.writeInt32(cmd);         // Parcel 实例，将 [BC_TRANSACTION][binder_transaction_data] 写入 mOut，mOut 是 client 进程内的一块内存空间
    mOut.write(&tr, sizeof(tr));  // 后续与 binder driver 通讯时：ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)，作为 binder_write_read.write_buffer 参数发送给目标进程

    return NO_ERROR;
}

status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)  // reply == null, acquireResult == null
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;  // talkWithDriver() 通过 ioctl(BINDER_WRITE_READ) 与 binder driver 交互
        err = mIn.errorCheck();                        // binder_write_read.write_buffer(mOut) 被 binder driver 读取（copy 至内核空间）
        if (err < NO_ERROR) break;                     // binder_write_read.read_buffer(mIn) 是 server 的返回值，需要 client 读取
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();               // [深入 Binder 之架构篇] 介绍过 BR_ 是 binder 给 client 的响应码
        switch (cmd) {                                 
        case BR_ONEWAY_SPAM_SUSPECT:
        case BR_TRANSACTION_COMPLETE:
        case BR_DEAD_REPLY:
        case BR_FAILED_REPLY:
        case BR_FROZEN_REPLY:
        case BR_ACQUIRE_RESULT:
        case BR_REPLY:                                 // BR_REPLY 指示 client 接收 server 返回的响应数据，内存布局同 BC_TRANSACTION 一样是 binder_transaction_data
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {  // 将 replay 指向 server 返回的响应
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer);
                    } else {
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(nullptr,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t));
                    }
                } else {
                    freeBuffer(nullptr,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t));
                    continue;
                }
            }
            goto finish;

        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
        logExtendedError();
    }

    return err;
}

// https://android.googlesource.com/platform/frameworks/native/+/refs/tags/android-mainline-12.0.0_r114/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::talkWithDriver(bool doReceive)    // doReceive 默认为 true
{
    if (mProcess->mDriverFD < 0) {
        return -EBADF;
    }
    binder_write_read bwr;
    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    bwr.write_size = outAvail;                    // 就是上面 writeTransactionData() 里填充的数据：[BC_TRANSACTION][binder_transaction_data]
    bwr.write_buffer = (uintptr_t)mOut.data();    // write_buffer 是将要发送给目标进程的数据

    if (doReceive && needRead) {                  // read_buffer(mIn) 是一块用以接收 server 返回值的内存，它的容量是 mIn.dataCapacity()
        bwr.read_size = mIn.dataCapacity();       // 如果 server 返回值太大装不下，就会执行多次 talkWithDriver 以接收返回值，此时优先 read 暂停 write
        bwr.read_buffer = (uintptr_t)mIn.data();  // mIn data size 在 binder io 结束后被设置为返回值的长度（下面的代码会讲到）
    } else {                                      // data position 指示读取偏移，如果 data position >= data size 说明返回值以全部读取完，mIn 空出来了，可以继续接收返回值
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR; // Return immediately if there is nothing to do.
    bwr.write_consumed = 0;    // write_buffer 是 client 发送给 server 的，中间商 binder driver 需要将其 copy 至内核空间以便 server 读取
                               // write_consumed 表示 binder driver 读取了多少内容
    bwr.read_consumed = 0;     // read_buffer 是 server 给 client 的返回值，client 需要读取并处理 read_buffer 里的内容
                               // read_buffer 里的内容由 server 写入，并将其长度写在 read_consumed
                               // 也就说当 ioctl(BINDER_WRITE_READ) 返回后，如果 read_consumed > 0 表示 server 有返回值
    status_t err;
    do {
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)  // 通过 ioctl 与 binder driver 通讯
            err = NO_ERROR;                                            // 在[深入 Binder 之架构篇]介绍过 BINDER_WRITE_READ 指示 binder 进行收发数据的操作
        else
            err = -errno;
        if (mProcess->mDriverFD < 0) {
            err = -EBADF;
        }
    } while (err == -EINTR);
    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                LOG_ALWAYS_FATAL("Driver did not consume write buffer. "
                                 "err: %s consumed: %zu of %zu",
                                 statusToString(err).c_str(),
                                 (size_t)bwr.write_consumed,
                                 mOut.dataSize());
            else {
                mOut.setDataSize(0);  // write_buffer 指向的内存由 Parcel mOut 管理，binder driver 拷贝到内核空间后就可以把 data size 置空以便后续使用
                processPostWriteDerefs();
            }
        }
        if (bwr.read_consumed > 0) {             // binder io 结束后如果 read_consumed > 0 表示 server 有返回值，需要 client 读取
            mIn.setDataSize(bwr.read_consumed);  // 此时 read_buffer (mIn) 虚拟内存地址已被 binder driver 映射到保存 server 返回值的物理地址
            mIn.setDataPosition(0);              // 将 mIn 的 data size 设置为 read_consumed，data position（指示当前读取位移）置零以便后续在 waitForResponse 读取返回值
        }
        return NO_ERROR;
    }
    return err;
}
```

# 参考

1. [深入理解 Binder 机制 5 - binder 驱动分析](https://skytoby.github.io/2020/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Binder%E6%9C%BA%E5%88%B65-binder%E9%A9%B1%E5%8A%A8%E5%88%86%E6%9E%90/)