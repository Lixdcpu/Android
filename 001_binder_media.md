# Android Binder 机制

## 一 说明

以MediaPlayerService为例来分析Binder的使用。

* service_manager，Android所有服务的管理程序；

* MediaService， 指代媒体播放的服务；

* MediaPlayerClient，指代与MediaService交互的客户端程序。

## 二 MediaService的诞生

MediaService是一个应用程序， 源码位于frameworks/av/media/mediaserver/main_mediaserver.cpp，
```
int main(int argc __unused, char **argv __unused)
{
    signal(SIGPIPE, SIG_IGN);

    /* sp这个东西是智能指针，sp<ProcessState>暂理解成ProcessState* 指针就好 */
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm(defaultServiceManager());

    /* 初始化MediaPlayerService服务并添加到service_manager */
    MediaPlayerService::instantiate();
    ResourceManagerService::instantiate();
    registerExtensions();

    /* 下面这两个东东，放到后面再详细说吧 */
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```

### 2.1 ProcessState

第一个调用函数，ProcessState::self()，代码位于frameworks/native/libs/binder/ProcessState.cpp，
```
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != nullptr) {
        return gProcess;
    }
    /* 这里就是构造ProcessState对象，打开的设备就是/dev/binder，
     * 这里是singleton，本进程内任何调用ProcessState::self()的地方返回的都是同一个 
     * gProcess */
    gProcess = new ProcessState(kDefaultDriver);
    return gProcess;
}

//下面看ProcessState的构造函数
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver)) //打开binder设备
    , mVMStart(MAP_FAILED) //映射内存的起始地址
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(nullptr)
    , mBinderContextUserData(nullptr)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
    , mCallRestriction(CallRestriction::NONE)
{
    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        /* 设备打开成功，从内核映射内存到用户空间，映射的大小是（1M-8K） */
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        ...
    }

    ...
}

//open_driver()
static int open_driver(const char *driver)
{
    int fd = open(driver, O_RDWR | O_CLOEXEC); //open /dev/binder
    if (fd >= 0) {
        ...
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
        /* 设置支持的最大线程数是15 */
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
        ...
    } else {
        ALOGW("Opening '%s' failed: %s\n", driver, strerror(errno));
    }
    return fd;
}
```

到这里ProcessState::self就分析完了，到底干了什么呢？

打开/dev/binder, 映射内存，ProcessState其实就是直接操作binder设备的。

### 2.2 defaultServiceManager

代码位置frameworks/native/libs/binder/IServiceManager.cpp,
```
sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != nullptr) return gDefaultServiceManager;

    {
        AutoMutex _l(gDefaultServiceManagerLock);
        while (gDefaultServiceManager == nullptr) {
            /* 实际在这里创建，interface_cast又做了什么呢？这个后面再说吧。
             * ProcessState::self()返回的是刚才创建的gProcess */
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(nullptr));
            if (gDefaultServiceManager == nullptr)
                sleep(1);
        }
    }

    return gDefaultServiceManager;
}

下面进入getContextObject，
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);//这里传入的是0
}

//注意下面接口的命名，proxy说明它是一个代理，handle类似一个句柄，有了这个句柄，客户
//端就可以通过binder驱动与服务端通信
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

/*
 struct handle_entry {
                IBinder* binder;
                RefBase::weakref_type* refs;
            };
*/
    handle_entry* e = lookupHandleLocked(handle);

    if (e != nullptr) {
        IBinder* b = e->binder;
        /* 第一次到这里，b是null */
        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
            /* 这里传入的handle是0，create就是创建BpBinder对象 */
            b = BpBinder::create(handle);
            e->binder = b;
            /* 增加弱引用计数，作用以后再研究吧 */
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}

BpBinder* BpBinder::create(int32_t handle) {
    ...
    return new BpBinder(handle, trackedUid);
}

代码有点长了，先不把BpBinder的构造函数贴出来了。
```

上面搞了这么长代码，到底做了什么呢？简化一下，就是干了下面的事情：

gDefaultServiceManager = interface_cast\<IServiceManager\>(new BpBinder(0))；

构造BpBinder时还有一个参数trackedUid，这里就先不管了。

### 2.3 BpBinder

代码位置frameworks/native/libs/binder/BpBinder.cpp，
```
BpBinder::BpBinder(int32_t handle, int32_t trackedUid)
    : mHandle(handle) //如果是defaultServiceManager，这里传入的就是0
    , mAlive(1)
    , mObitsSent(0)
    , mObituaries(nullptr)
    , mTrackedUid(trackedUid)
{
    extendObjectLifetime(OBJECT_LIFETIME_WEAK);
    IPCThreadState::self()->incWeakHandle(handle, this);
}

//IPCThreadState::self()，代码位于frameworks/native/libs/binder/IPCThreadState.cpp
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        /* TLS即Thread Local Storage，可以理解成每个线程自己私有的空间，可以通过
         pthread_getspecific获取线程自己的私有的变量，第一次进来是走不到这里的，在下面通过pthread_key_create创建线程自己的私有空间*/
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState; //new一个对象
    }

    if (gShutdown) {
        ALOGW("Calling IPCThreadState::self() during shutdown is dangerous, expect a crash.\n");
        return nullptr;
    }

    pthread_mutex_lock(&gTLSMutex);
    if (!gHaveTLS) {
        int key_create_value = pthread_key_create(&gTLS, threadDestructor);
        if (key_create_value != 0) {
            pthread_mutex_unlock(&gTLSMutex);
            ALOGW("IPCThreadState::self() unable to create TLS key, expect a crash: %s\n",
                    strerror(key_create_value));
            return nullptr;
        }
        gHaveTLS = true;
    }
    pthread_mutex_unlock(&gTLSMutex);
    goto restart;
}

//在IPCThreadState的构造函数里，通过pthread_setspecific将new的对象放到线程的私有空间中
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()), //这个就是前面的gProcess，每个进程只有一个
      mWorkSource(kUnsetWorkSource),
      mPropagateWorkSource(false),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0),
      mCallRestriction(mProcess->mCallRestriction)
{
    pthread_setspecific(gTLS, this);
    clearCaller();
    /* mIn和mOut，两个Parcel对象，Parcel是binder通信存放数据的容器，这里大概是设置
     Parcel的容量吧*/
    mIn.setDataCapacity(256);
    mOut.setDataCapacity(256);
    mIPCThreadStateBase = IPCThreadStateBase::self();
}
```

到目前为止，都干了些什么呢？先总结一下吧：

* ProcessState有了，直接操作/dev/binder设备；
* IPCThreadState有了，这个每个线程对应一个；
* BpBinder有了，内部handle值为0。

service_manager就类似一个域名解析服务器，他的作用就是通过服务名找到服务对应的handle。类似DNS地址8.8.8.8，handle 0就是service_manager的DNS地址。

再回过头看下面的东东：

gDefaultServiceManager = interface_cast\<IServiceManager\>(new BpBinder(0))

class IServiceManager : public IInterface

class BpBinder : public IBinder

这里，一个Binder怎么就变成了Interface呢？来看interface_cast：
```
//代码位于frameworks/native/libs/binder/include/binder/IInterface.h
template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{
    return INTERFACE::asInterface(obj);
}

把上面的东西翻译一下，就是：
inline sp(IServiceManager) interface_cast(const sp<IBinder>& obj)
{
    return IServiceManager::asInterface(obj);
}
```

下面看看IServiceManager吧。

### 2.4 IServiceManager

```
class IServiceManager : public IInterface
{
//ServiceManager，字面上理解就是Service管理类，作用是添加服务、查询服务等等。
//这里只把addService列出来。
public:
    DECLARE_META_INTERFACE(ServiceManager)
    ...
    virtual status_t addService(const String16& name, const sp<IBinder>& service,
                                bool allowIsolated = false,
                                int dumpsysFlags = DUMP_FLAG_PRIORITY_DEFAULT) = 0;
    ...
    enum {
        GET_SERVICE_TRANSACTION = IBinder::FIRST_CALL_TRANSACTION,
        CHECK_SERVICE_TRANSACTION,
        ADD_SERVICE_TRANSACTION,
        LIST_SERVICES_TRANSACTION,
    };
}

//DECLARE_META_INTERFACE是一个声明的宏，这连个宏是代码在刚才的IInterface.h中
#define DECLARE_META_INTERFACE(INTERFACE)                               \
public:                                                                 \
    static const ::android::String16 descriptor;                        \
    static ::android::sp<I##INTERFACE> asInterface(                     \
            const ::android::sp<::android::IBinder>& obj);              \
    virtual const ::android::String16& getInterfaceDescriptor() const;  \
    I##INTERFACE();                                                     \
    virtual ~I##INTERFACE();                                            \
    static bool setDefaultImpl(std::unique_ptr<I##INTERFACE> impl);     \
    static const std::unique_ptr<I##INTERFACE>& getDefaultImpl();       \
private:                                                                \
    static std::unique_ptr<I##INTERFACE> default_impl;                  \
public:                                                                 \

//这里大体就是定义一个描述字符串，定义一个转换接口asInterface，把IBinder转化成
//IInterface。
//有声明的宏就有定义的宏，也在IInterface.h中，
#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
    const ::android::String16 I##INTERFACE::descriptor(NAME);           \
    const ::android::String16&                                          \
            I##INTERFACE::getInterfaceDescriptor() const {              \
        return I##INTERFACE::descriptor;                                \
    }                                                                   \
    ::android::sp<I##INTERFACE> I##INTERFACE::asInterface(              \
            const ::android::sp<::android::IBinder>& obj)               \
    {                                                                   \
        ::android::sp<I##INTERFACE> intr;                               \
        if (obj != nullptr) {                                           \
            intr = static_cast<I##INTERFACE*>(                          \
                obj->queryLocalInterface(                               \
                        I##INTERFACE::descriptor).get());               \
            if (intr == nullptr) {                                      \
                intr = new Bp##INTERFACE(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
    std::unique_ptr<I##INTERFACE> I##INTERFACE::default_impl;           \
    bool I##INTERFACE::setDefaultImpl(std::unique_ptr<I##INTERFACE> impl)\
    {                                                                   \
        if (!I##INTERFACE::default_impl && impl) {                      \
            I##INTERFACE::default_impl = std::move(impl);               \
            return true;                                                \
        }                                                               \
        return false;                                                   \
    }                                                                   \
    const std::unique_ptr<I##INTERFACE>& I##INTERFACE::getDefaultImpl() \
    {                                                                   \
        return I##INTERFACE::default_impl;                              \
    }                                                                   \
    I##INTERFACE::I##INTERFACE() { }                                    \
    I##INTERFACE::~I##INTERFACE() { }

//IMPLEMENT_META_INTERFACE在
IMPLEMENT_META_INTERFACE(ServiceManager, "android.os.IServiceManager");

//把它翻译一下吧
    const ::android::String16 IServiceManager::descriptor("android.os.IServiceManager");           \
    const ::android::String16&                                          \
            IServiceManager::getInterfaceDescriptor() const {              \
        return IServiceManager::descriptor;                                \
    }                                                                   \
    ::android::sp<IServiceManager> IServiceManager::asInterface(              \
            const ::android::sp<::android::IBinder>& obj)               \
    {                                                                   \
        ::android::sp<IServiceManager> intr;                               \
        if (obj != nullptr) {                                           \
            intr = static_cast<IServiceManager*>(                          \
                obj->queryLocalInterface(                               \
                        IServiceManager::descriptor).get());               \
            if (intr == nullptr) {                                      \
                //关键就在这里，new BpServiceManager对象
                intr = new BpServiceManager(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
    std::unique_ptr<IServiceManager> IServiceManager::default_impl;           \
    bool IServiceManager::setDefaultImpl(std::unique_ptr<IServiceManager> impl)\
    {                                                                   \
        if (!IServiceManager::default_impl && impl) {                      \
            IServiceManager::default_impl = std::move(impl);               \
            return true;                                                \
        }                                                               \
        return false;                                                   \
    }                                                                   \
    const std::unique_ptr<IServiceManager>& IServiceManager::getDefaultImpl() \
    {                                                                   \
        return IServiceManager::default_impl;                              \
    }                                                                   \
    IServiceManager::IServiceManager() { }                                    \
    IServiceManager::~IServiceManager() { }                                   \
```

从上面可以看出，asInterface的作用就是new了一个BpServiceManager对象：

gDefaultServiceManager = new BpServiceManager(new BpBinder(0));

进一步总结一下，interface_cast\<Ixxx\>(binder)的作用，就是以一个BpBinder为参数，new BpXXX对象：

serviceProxy = new BpXXX(binder);

* 如果binder =  BpBinder(0)，那么serviceProxy就是gDefaultServiceManager，也就是SM的代理；
* 如果binder = gDefaultServiceManager.getService(SERVICE_NAME)，那么serviceProxy就是向SM注册过的，名字是SERVICE_NAME的服务的代理。

### 2.5 BpServiceManager

p是proxy即代理的意思，Bp就是BinderProxy，BpServiceManager，就是SM的Binder代理。既然是代理，那肯定希望对用户是透明的，那就是说头文件里边不会有这个Bp的定义。
BpServiceManager就在刚才的IServiceManager.cpp中定义，
```
class BpServiceManager : public BpInterface<IServiceManager>
{
public:
    explicit BpServiceManager(const sp<IBinder>& impl)
        : BpInterface<IServiceManager>(impl) //impl就是传入的BpBinder(0)
    {
    }

    virtual sp<IBinder> addService(const String16& name, const sp<IBinder>& service,
                                bool allowIsolated, int dumpsysPriority)
    {
        //后面再说吧
    }

//下面看看BpInterface的构造函数吧，直接翻译出来
inline BpInterface<IServiceManager>::BpInterface(const sp<IBinder>& remote)
    : BpRefBase(remote)
{
}

//看下面，最后mRemote就是刚才的BpBinder(0)了
BpRefBase::BpRefBase(const sp<IBinder>& o)
    : mRemote(o.get()), mRefs(nullptr), mState(0)
{
    ...
}

```

好了，到这里该总结一下了：

sp\<IServiceManager\> sm = defaultServiceManager(); 返回的实际是BpServiceManager，它的remote对象BpBinder，传入的handle是0。

再回到main()函数看一下
```
int main(int argc __unused, char **argv __unused)
{
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm(defaultServiceManager());
    /* 上面，sm已经搞完了，我们明白了它是个什么东西 */

    /* 这里MediaPlayerService可以出来了 */
    MediaPlayerService::instantiate();

    ResourceManagerService::instantiate();
    registerExtensions();

    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```

### 2.6 MediaPlayerService

直接把MediaPlayerService::instantiate()贴出来
```
void MediaPlayerService::instantiate() {
    /* 这里调用BpServiceManager的addService，将new的MediaPlayerService添加到SM */
    defaultServiceManager()->addService(
            String16("media.player"), new MediaPlayerService());
}

MediaPlayerService::MediaPlayerService()
{
    /*  选择无视这里，那么MediaPlayerService构造基本什么都没干了 */
    MediaPlayerFactory::registerBuiltinFactories();
}

//来看看MediaPlayerService的定义吧，
class MediaPlayerService : public BnMediaPlayerService
{
    /* 原来MediaPlayerService是从BnMediaPlayerService派生的，那BnMediaPlayerService又是个什么东东呢？*/
    ...
}
```

Bn 是Binder Native的含义，是和Bp相对的，Bp的p是proxy代理的意思，Bp代理，Bn服务，BpXXX对应BnXXX，完美了。

我们在这里创建了MediaPlayerService，通过SM的代理BpServiceManager把它加入到SM，系统其它需要MediaService的地方就可以通过SM获取其代理BpMediaPlayerService了。

至于BpMediaPlayerService，后面讲到MediaPlayerClient的时候再说吧。

### 2.7 addService

```
virtual status_t addService(const String16& name, const sp<IBinder>& service,
                                bool allowIsolated, int dumpsysPriority) {
        /* data是发送到BnServiceManager的命令包 */
        Parcel data, reply;

        /* 先把Interface名字写进去，也就是什么android.os.IServiceManager */
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());

        /* 再把新service的名字写进去 叫media.player */
        data.writeString16(name);
        data.writeStrongBinder(service);
        data.writeInt32(allowIsolated ? 1 : 0);
        data.writeInt32(dumpsysPriority);
        /* 这里remote()返回的啥？其实就是最初创建BpServiceManager出入的BpBinder */
        status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
        return err == NO_ERROR ? reply.readExceptionCode() : err;
    }
```


下面来看看位于frameworks/native/libs/binder/BpBinder.cpp里的transact(),
```
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        /* 注意啊，这里的mHandle就是0，IPCThreadState::self()就是一个singleton，想想前面的线程私有空间*/
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}

//看看IPCThreadState的transact()吧
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err;

    flags |= TF_ACCEPT_FDS;
    ...
    /* 是在这里把命令发走的吗？ NO NO 这里只是写到发送缓冲里 */
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);
    ...
    /* 在这里发送缓冲里的数据发走，需要回复的话等待回复 */
    err = waitForResponse(reply);
    ...
}

//先来看看writeTransactionData()干了些什么吧
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
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
    /* 上面的一大段，就是把需要发送的命令数据封装成binder_transaction_data */

    /* 这里把命令、组长好的tr写到mOut中，mOut是发送缓冲区，也是一个Parcel */
    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}

//再来看看waitForResponse()吧
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        /* 还是没有看到在哪里吧mOut发走了，但是这里开始操作mIn了，看来是在talkWithDriver中把mOut发出去，然后从driver中读到数据放到mIn中了*/
        cmd = (uint32_t)mIn.readInt32();
        ...
        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;
        ...
        }
        ...
    }
    ...
    return err;
}

//最终到talkWithDriver()里了，这里与驱动交互，不会再进一层了
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    ...
    binder_write_read bwr;
    ...
    /* 与驱动交互，封装binder_write_read */
    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    ...
    /* 看到ioctl了，读写驱动了 */
    if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
    ...
        /* 到这里，回复数据就在bwr中了，bwr接收回复数据的buffer就是mIn提供的 */
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
    ...
}
```

好了，addService()的流程走完了。

通过BpServiceManager的addService把一个服务加入到SM，然后收到回复。

继续MediaService的main()函数吧。
```
int main(int argc __unused, char **argv __unused)
{
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm(defaultServiceManager());
    /* 上面，sm已经搞完了，我们明白了它是个什么东西 */

    /* 这里也搞完了， new MediaPlayerService，然后调用addService加到SM中 */
    MediaPlayerService::instantiate();

    ResourceManagerService::instantiate();
    registerExtensions();

    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```

这里已经有MediaPlayerService服务了，但还是有点搞晕了，服务怎么运行，怎么响应代理端的请求呢？来捋一下：

MediaPlayerService服务派生自BnMediaPlayerService，它应该打开binder设备，然后有一个looper，循环等待BpMediaPlayerService发来的请求。

下面是不是要把BnMediaPlayerService先说一下？

不，先放一放，后面讲到MediaService的运行时，再详细说一下它。

前面还有一点没整明白，MediaPlayerService被加入到SM中，SM到底是个什么东东，下面先说说它吧。

大脑快宕机了。。。

### 2.8 service_manager

上面说了，defaultServiceManager返回的是一个BpServiceManager，通过它可以把命令请求发送到binder设备，而且handle的值为0。

那么根据BpXXX与BnXXX对应的原则，是不是应该存在一个BnServiceManager呢？很遗憾，并不存在。但有一个东西，它叫service_manager，位于frameworks/native/cmds/servicemanager/service_manager.c，它没有BnServiceManager的壳，但有BnServiceManager的魂。太啰嗦了，service_manager其实就是实现了BnServiceManage的功能。

service_manager.c的main()函数。
```
int main(int argc, char** argv)
{
    struct binder_state *bs;
    union selinux_callback cb;
    char *driver;

    if (argc > 1) {
        driver = argv[1];
    } else {
        driver = "/dev/binder";
    }

    /* 打开binder设备 */
    bs = binder_open(driver, 128*1024);
    ...
    /* 成为manager */
    if (binder_become_context_manager(bs))
    ...
    /* 进入循环，处理BpServiceManager发过来的命令 */
    binder_loop(bs, svcmgr_handler);

    return 0;
}

struct binder_state *binder_open(const char* driver, size_t mapsize)
{
    ...
    /* 果然，打开binder设备 */
    bs->fd = open(driver, O_RDWR | O_CLOEXEC);
    ...
    /* 映射内存，常规操作，打开binder设备就要映射内存 */
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    ...
    return bs;
}

int binder_become_context_manager(struct binder_state *bs)
{
    ...
    /* 把自己设为MANAGER */
    int result = ioctl(bs->fd, BINDER_SET_CONTEXT_MGR_EXT, &obj);
    ...
}

void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));

    for (;;) { //循环了，处理请求了
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;

        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        ...
        /* 收到请求，解析它 */
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        ...
    }
}

//binder_loop传入一个钩子svcmgr_handler，所有解析到的命令都在这里处理
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data_secctx *txn_secctx,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si;
    uint16_t *s;
    size_t len;
    uint32_t handle;
    uint32_t strict_policy;
    int allow_isolated;
    uint32_t dumpsys_priority;

    struct binder_transaction_data *txn = &txn_secctx->transaction_data;
    ...
    switch(txn->code) {
    ...
    case SVC_MGR_ADD_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        handle = bio_get_ref(msg);
        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
        dumpsys_priority = bio_get_uint32(msg);
        if (do_add_service(bs, s, len, handle, txn->sender_euid, allow_isolated, dumpsys_priority,
                           txn->sender_pid, (const char*) txn_secctx->secctx))
            return -1;
        break;
    ...
    }
    ...
    bio_put_uint32(reply, 0);
    return 0;
}

//继续贴代码do_add_service
int do_add_service(struct binder_state *bs, const uint16_t *s, size_t len, uint32_t handle,
                   uid_t uid, int allow_isolated, uint32_t dumpsys_priority, pid_t spid, const char* sid) {
    struct svcinfo *si;

    ...
    /* 检查权限，服务能否被注册，不太明白 */
    if (!svc_can_register(s, len, spid, sid, uid)) {
        ALOGE("add_service('%s',%x) uid=%d - PERMISSION DENIED\n",
             str8(s, len), handle, uid);
        return -1;
    }

    /* 从一个链表中查找服务是否已经被注册过了，没有的话就添加到链表中 */
    si = find_svc(s, len);
    if (si) {
        if (si->handle) {
            ALOGE("add_service('%s',%x) uid=%d - ALREADY REGISTERED, OVERRIDE\n",
                 str8(s, len), handle, uid);
            svcinfo_death(bs, si);
        }
        si->handle = handle;
    } else {
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        if (!si) {
            ALOGE("add_service('%s',%x) uid=%d - OUT OF MEMORY\n",
                 str8(s, len), handle, uid);
            return -1;
        }
        si->handle = handle;
        si->len = len;
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->dumpsys_priority = dumpsys_priority;
        si->next = svclist;
        svclist = si; //新服务被添加到链表中了，相当于完成了注册
    }

    /* 下面这个东西吗，大概是服务退出后，我希望系统通知我一下，好释放上面malloc出来的资源。大概就是干这个事情的 */
    binder_acquire(bs, handle);
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}
```

service_manager讲完了，MediaPlayerService相关的信息被添加到自己维护的一个服务列表中了。

总结一下：

* MediaPlayerService向SM注册；
* MediaPlayerClient查询当前注册在SM中的MediaPlayerService的信息；
* 根据这个信息，MediaPlayerClient和MediaPlayerService交互。

SM的handle标示是0，所以只要往handle是0的服务发送消息，最终都会被传递到SM中去。

## 三 MediaService的运行

根据前面的知识，我们可以得到如下结论：

* defaultServiceManager得到了BpServiceManager，然后MediaPlayerService实例化后，调用BpServiceManager的addService函数向service_manager注册；
* service_manager收到addService的请求，然后把对应信息放到自己保存的一个服务list中；

service_manager有一个binder_looper函数，专门等着从binder中接收请求。虽然service_manager没有从BnServiceManager中派生，但是它肯定完成了BnServiceManager的功能。

我们已经在前面猜想过，既然创建了MediaPlayerService即BnMediaPlayerService，那它也应该完成下面的事情：

* MediaPlayerService打开binder设备；
* 搞一个looper，然后坐等BpMediaPlayerService发来的请求。

但是前面也说了，MediaPlayerService的构造函数并没有打开binder设备呀，那么看看它的父类BnMediaPlayerService干了些什么吧。

### 3.1 MediaPlayerService打开binder

惯例，先贴代码吧。
```
//MediaPlayerService从BnMediaPlayerService派生
class MediaPlayerService : public BnMediaPlayerService

//BnMediaPlayerService从BnInterface和IMediaPlayerService同时派生
class BnMediaPlayerService: public BnInterface<IMediaPlayerService>
{
public:
    virtual status_t    onTransact( uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);
};

//BnInterface继续贴
template<typename INTERFACE>
class BnInterface : public INTERFACE, public BBinder
{
public:
    virtual sp<IInterface>      queryLocalInterface(const String16& _descriptor);
    virtual const String16&     getInterfaceDescriptor() const;

protected:
    typedef INTERFACE           BaseInterface;
    virtual IBinder*            onAsBinder();
};

//又是个模板类，把它翻译一下
class BnInterface : public IMediaPlayerService, public BBinder
/* 上面这里出现BBinder，BBinder、BpBinder是不是服务端和代理端两边对应的呢？ */
/* 另外，从这里看出，BnMediaPlayerService其实是从IMediaPlayerService和BBinder同时派生*/
{
    ...
};

//这里先放着BBinder，后面再讲讲它的作用。
```

到这里，也并没有发现哪里有打开binder设备的地方啊。

回想下，Main_MediaService的main()函数，哪里打开过binder？
```
int main(int argc __unused, char **argv __unused)
{
    /* 就是这里呀，这里打开过binder */
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm(defaultServiceManager());
    ...
}
```

### 3.2 looper  

Main_MediaService的main()函数中，下面两个东西还没有分析，来看看，
```

//就是在这里进入looper的
ProcessState::self()->startThreadPool();
IPCThreadState::self()->joinThreadPool();

//startThreadPool()一步步跟进入
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true); //这里
    }
}

void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain); //isMain是true
        t->run(name.string()); //这里，起一个线程，起什么线程呢？
    }
}

class PoolThread : public Thread
{
public:
    explicit PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }

protected:
    virtual bool threadLoop()
    {
        /* 起线程后，会运行到这里，mIsMain是true，这里运行的是主线程 */
        /* 注意，这里线程是新创建的，所有也会新建IPCThreadState对象 */
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }

    const bool mIsMain;
};
```

原来的线程和主线程都调用了joinThreadPool()，下面看看它干了啥。
```
void IPCThreadState::joinThreadPool(bool isMain)
{
    ...
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    status_t result;
    /* 下面有循环了，这里应该是在跟binder交互吧，去看看 */
    do {
        result = getAndExecuteCommand();
        ...
    } while (result != -ECONNREFUSED && result != -EBADF);

    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}

status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;

    result = talkWithDriver();//这里就收到代理端发来的请求了
    if (result >= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN < sizeof(int32_t)) return result;
        cmd = mIn.readInt32();

        ...

        /* 处理代理端请求 */
        result = executeCommand(cmd);

        ...
    }

    return result;
}

status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch ((uint32_t)cmd) {
    ...
    case BR_TRANSACTION:
        {
            binder_transaction_data_secctx tr_secctx;
            binder_transaction_data& tr = tr_secctx.transaction_data;
            ...
            if (tr.target.ptr) {
                ...
                    /* 这里调用BBinder，tr.cookie哪里来的？怎么就能强制转换成BBinder？应该是Service向service_manager注册的时候保存起来，代理端向服务端发起请求时获取到，这里是不是类似回调？以后再具体研究吧。*/
                    error = reinterpret_cast<BBinder*>(tr.cookie)->transact(tr.code, buffer,
                            &reply, tr.flags);

            } else {
                error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
            }
            ...
        }
    ...
    }
    ...
}

//调用BBinder的transact干了啥？
status_t BBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    data.setDataPosition(0);

    status_t err = NO_ERROR;
    switch (code) {
        case PING_TRANSACTION:
            reply->writeInt32(pingBinder());
            break;
        default:
            /* 调用自己的onTransact函数 */
            err = onTransact(code, data, reply, flags);
            break;
    }

    if (reply != nullptr) {
        reply->setDataPosition(0);
    }

    return err;
}
```

BnMediaPlayerService从BBinder派生，所以会调用它的onTransact。
终于水落石出了，来看看BnMediaPlayerService的onTransact函数吧。
```

//代码位于frameworks/av/media/libmedia/IMediaPlayerService.cpp
status_t BnMediaPlayerService::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    /* BnMediaPlayerService从BBinder和IMediaPlayerService派生，所有IMediaPlayerService提供的函数都通过命令类型来区分 */
    switch (code) {
        case CREATE: {
            CHECK_INTERFACE(IMediaPlayerService, data, reply);
            sp<IMediaPlayerClient> client =
                interface_cast<IMediaPlayerClient>(data.readStrongBinder());
            audio_session_t audioSessionId = (audio_session_t) data.readInt32();
            /* create是一个纯虚函数，它由MediaPlayerService来实现 */
            sp<IMediaPlayer> player = create(client, audioSessionId);
            reply->writeStrongBinder(IInterface::asBinder(player));
            return NO_ERROR;
        } break;
        ...
    }
    ...
}
```

到这里，我们就搞明白了，BnXXX通过onTransact收取命令，onTransact一般在IXXX中实现，onTransact再将说去的命令派发到派生类，由派生类完成实际的工作。

## 四 MediaPlayerClient

这节讲讲MediaPlayerClient怎么和MediaPlayerService交互。
使用MediaPlayerService的时候，先要创建它的代理即BpMediaPlayerService。我们看看一个例子：
```

//代码位于frameworks/av/media/libmedia/IMediaDeathNotifier.cpp
IMediaDeathNotifier::getMediaPlayerService()
{
    ALOGV("getMediaPlayerService");
    Mutex::Autolock _l(sServiceLock);
    if (sMediaPlayerService == 0) {
        sp<IServiceManager> sm = defaultServiceManager();
        sp<IBinder> binder;
        do {
            /* 获取binder */
            binder = sm->getService(String16("media.player"));
            if (binder != 0) {
                break;
            }
            ALOGW("Media player service not published, waiting...");
            usleep(500000); // 0.5 s
        } while (true);

        if (sDeathNotifier == NULL) {
            sDeathNotifier = new DeathNotifier();
        }
        binder->linkToDeath(sDeathNotifier);
        /* 通过interface_cast，将这个binder转化成BpMediaPlayerService */
        /* 注意，这个binder只是用来和binder设备通信用的，实际上和IMediaPlayerService的功能一点关系都没有。BpMediaPlayerService用这个binder和BnMediaPlayerService通信。*/
        sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);
    }
    ALOGE_IF(sMediaPlayerService == 0, "no media player service!?");
    return sMediaPlayerService;
}
```

Binder基本说完了，虽然不深入，有些地方表达的意思可能还有些偏差，但大体上大差不差吧。
