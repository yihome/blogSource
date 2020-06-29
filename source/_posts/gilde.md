---
title: Glidle杂记
date: 2019-07-09 10:18:26
tags: 
- Android
categories: Android
---


[TOC]
# Glide杂记
Glide使用过程加载过程主要有如下几个步骤
* with ReqestManager （Lifecycle，RequestTracker）
管理关联在同一个Activity（不包括子Activity）或者说关联在统一Fragment上的所有Request
* load RequestBuilder 设置请求的各种参数，用于创建请求
* into Request.begin 开始请求

总体来说Glide的使用可以分为Request构造和Request运行两个部分

## Reqest构造过程

Reqest的构造过程主要完成了两件事：

* 宿主（Activity、Fragment等）生命周期的绑定
* Request参数的创建的管理

#### 生命周期绑定

![img](./glide/request.png)

#### Request参数设置

Request参数除了我们手动设置的url、placeHolder、transform等参数之外在创建真正的Request之前还会确定几个重要的策略，这些策略在Glide的不同版本可能会有不同的组合，但整体结构没有太大变化

* ModelLoader
* DataLoadProvider
* DataFetcher

## Glide请求过程

Glide的请求过程主要分为

* 构造请求
    * DrawableTypeRequest
    * GifTypeRequest
    * BitmapTypeRequest
* 获取数据源 loadProvider
* 加载解码成可用的数据
* 显示

### loadProvider

可扩展的LoaderProvider
 * FixedLoadProvider
 集成了DataLoadProvider、ModelLoader和ResourceTranscoder
    * ModelLoader定义了数据源（从Uri、文件、String等获取原始的输入流或着其封装类）
    * DataLoadProvider定义了输入流到目标资源以及缓存文件三者之间的转换关系
    * ResourceTranscoder定义了一种资源格式到另一种资源格式（Drawable、Bitmap、gif）的转码方式
 * ChildLoadProvider

#### DataLoadProvider
* DataFetcher获取的数据源(InputSream等)
* 目标资源Resourse<?>（Bitmap，Drawable，Gif等）
* File 文件缓存


预设的DataLoadProvider

 ```java
 dataLoadProviderRegistry = new DataLoadProviderRegistry();

    StreamBitmapDataLoadProvider streamBitmapLoadProvider =
            new StreamBitmapDataLoadProvider(bitmapPool, decodeFormat);
    dataLoadProviderRegistry.register(InputStream.class, Bitmap.class, streamBitmapLoadProvider);

    FileDescriptorBitmapDataLoadProvider fileDescriptorLoadProvider =
            new FileDescriptorBitmapDataLoadProvider(bitmapPool, decodeFormat);
    dataLoadProviderRegistry.register(ParcelFileDescriptor.class, Bitmap.class, fileDescriptorLoadProvider);

    ImageVideoDataLoadProvider imageVideoDataLoadProvider =
            new ImageVideoDataLoadProvider(streamBitmapLoadProvider, fileDescriptorLoadProvider);
    dataLoadProviderRegistry.register(ImageVideoWrapper.class, Bitmap.class, imageVideoDataLoadProvider);

    GifDrawableLoadProvider gifDrawableLoadProvider =
            new GifDrawableLoadProvider(context, bitmapPool);
    dataLoadProviderRegistry.register(InputStream.class, GifDrawable.class, gifDrawableLoadProvider);

    dataLoadProviderRegistry.register(ImageVideoWrapper.class, GifBitmapWrapper.class,
            new ImageVideoGifDrawableLoadProvider(imageVideoDataLoadProvider, gifDrawableLoadProvider, bitmapPool));

    dataLoadProviderRegistry.register(InputStream.class, File.class, new StreamFileDataLoadProvider());

 ```



####  ModelLoader 装饰者+适配器

将我们通过load传入的参入转换成特定的DataFetcher<Resource>输出（InputStream或者其封装类）
```java
Glide(Engine engine, MemoryCache memoryCache, BitmapPool bitmapPool, Context context, DecodeFormat decodeFormat) {
    ......
    //Model class ,Resource class, LoaderFactory
    register(File.class, ParcelFileDescriptor.class, new FileDescriptorFileLoader.Factory());
    register(File.class, InputStream.class, new StreamFileLoader.Factory());
    register(int.class, ParcelFileDescriptor.class, new FileDescriptorResourceLoader.Factory());
    register(int.class, InputStream.class, new StreamResourceLoader.Factory());
    register(Integer.class, ParcelFileDescriptor.class, new FileDescriptorResourceLoader.Factory());
    register(Integer.class, InputStream.class, new StreamResourceLoader.Factory());
    register(String.class, ParcelFileDescriptor.class, new FileDescriptorStringLoader.Factory());
    register(String.class, InputStream.class, new StreamStringLoader.Factory());
    register(Uri.class, ParcelFileDescriptor.class, new FileDescriptorUriLoader.Factory());
    register(Uri.class, InputStream.class, new StreamUriLoader.Factory());
    register(URL.class, InputStream.class, new StreamUrlLoader.Factory());
    register(GlideUrl.class, InputStream.class, new HttpUrlGlideUrlLoader.Factory());
    register(byte[].class, InputStream.class, new StreamByteArrayLoader.Factory());
    ......
}
```
#### DataFetcher 装饰者+适配器
通过网络或者文件、资源等方式获取数据源

## BitmapPool
###  LruPoolStrategy
#### SizeConfigStrategy
* `GroupedLinkedMap<Key, Bitmap> groupedMap` 自定义的LruCache，使用链表和HashMap实现
* `Map<Bitmap.Config, NavigableMap<Integer, Integer>> sortedSizes` 以Bitmap的Config为纬度存储了对象池中每种对象大小的数量，用于快速判断是否存在可用的Bitmap对象缓存，这里使用的`NavigableMap`为`TreeMap`,key的存储方式为红黑树结构（使用红黑树是因为在复用时可能没有完全大小相同的Bitmap对象，所以会遍历整个map按大小顺序找到第一个比需要的Bitmap大的复用）

## Glide 缓存

### Key的构造
```java
 EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
                loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
                transcoder, loadProvider.getSourceEncoder());
```
从EngineKey的构造可以看出，即使是同一个Url下的不同尺寸或者不同的transformation、transcoder，也会产生不同的Key

### 缓存
Glide的内存缓存中存在两种内存缓存机制、LruCache（使用LikedHashMap结构存储的强引用内存缓存，默认的缓存大小为两个整屏幕的图片大小）和使用弱引用存储的activeResources
* 优先从强引用中获取图片
* 其次试图从弱引用中获取图片
* 最后再考虑磁盘

LruCache的大小固定，即使内存情况良好，也不会缓存更多图片，使用弱引用就是为了在内存环境良好的情况下在内存中缓存更多的图片，从而提升缓存的加载速度。

```java
EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
if (cached != null) {
    cb.onResourceReady(cached);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
    }
    return null;
}

EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
if (active != null) {
    cb.onResourceReady(active);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
    }
    return null;
}

EngineJob current = jobs.get(key);
if (current != null) {
    current.addCallback(cb);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Added to existing load", startTime, key);
    }
    return new LoadStatus(cb, current);
}

EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
        transcoder, diskCacheProvider, diskCacheStrategy, priority);
EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
jobs.put(key, engineJob);
engineJob.addCallback(cb);
engineJob.start(runnable);
```
代码中可以看出从内存缓存中获取Resource文件时，只使用原始的EngineKey，这样获取的Resource时是指定宽高且经过Transform和transcoder转换后的，由此也可以看出，内存缓存中只会有经过各种转换之后的图片资源
### 磁盘缓存过程
在无法从内存中获取指定的图片资源之后，会尝试从磁盘或者网络获取，这个过程在`EngineRunnable`中完成
```java
private Resource<?> decode() throws Exception {
    if (isDecodingFromCache()) {
        return decodeFromCache();
    } else {
        return decodeFromSource();
    }
    
    private Resource<?> decodeFromCache() throws Exception {
        Resource<?> result = null;
        try {
            result = decodeJob.decodeResultFromCache();
        } catch (Exception e) {
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "Exception decoding result from cache: " + e);
            }
        }

        if (result == null) {
            result = decodeJob.decodeSourceFromCache();
        }
        return result;
    }
}
```
当从磁盘缓存中获取图片时会有两个过程
1. 试图使用完整的EngineKey（包括宽、高、Transform和transcoder）获取指定的文件，当然DataLoadProvider只提供从File到指定中间资源类型的转化，这里从磁盘获取资源之后还是会使用transcoder进行转化
2. 当无法通过完整的EngineKey获取到资源时，会尝试使用OriginalKey获取资源，OriginalKey只包含请求id和签名，只能查找原始没有经过任何大小或者其他转换前的资源文件
```java
public Resource<Z> decodeSourceFromCache() throws Exception {
    if (!diskCacheStrategy.cacheSource()) {
        return null;
    }

    long startTime = LogTime.getLogTime();
    Resource<T> decoded = loadFromCache(resultKey.getOriginalKey());
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Decoded source from cache", startTime);
    }
    return transformEncodeAndTranscode(decoded);
}
private Resource<Z> transformEncodeAndTranscode(Resource<T> decoded) {
    long startTime = LogTime.getLogTime();
    Resource<T> transformed = transform(decoded);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Transformed resource from source", startTime);
    }

    writeTransformedToCache(transformed);

    startTime = LogTime.getLogTime();
    Resource<Z> result = transcode(transformed);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Transcoded transformed from source", startTime);
    }
    return result;
}
```
在第二次尝试从磁盘获取资源成功后会先通过transform来对中间的资源类型进行转换后，并把转换后同时宽高以及变换的图片写入磁盘缓存，最后进行transcode

### 网络请求
在无法通过磁盘缓存获取到数据时，会进入一个异常处理流程，同时会判断异常是否是由于磁盘没有文件导致的
```java
 private void onLoadFailed(Exception e) {
    if (isDecodingFromCache()) {
        stage = Stage.SOURCE;
        manager.submitForSource(this);
    } else {
        manager.onException(e);
    }
}
```

## 纵观全局

在整个Glide的源码阅读过程中最大的印象就是缓存，图片有LRUcache，weakRefence、DiskLruCache，key有KeyPool，Bitmap有BitmapPool，ModelLoader有ModelLoaderCache

其次就是


# 储备知识
### ReferenceQueue
* FinalReference
* SoftReference
* WeakReference
* PhantomReference

ReferenceQueue 主要适用于追踪后三种引用类型的被系统回收的情况，可以作为构造入参传入他们的构造函数中，系统会自动把被系统回收的对象添加到这个`ReferenceQueue`中。

```/art/runtime/gc/heap.cc ```
```cpp
collector::GcType Heap::CollectGarbageInternal(collector::GcType gc_type,
                                               GcCause gc_cause,
                                               bool clear_soft_references) {
  Thread* self = Thread::Current();
  Runtime* runtime = Runtime::Current();
  // If the heap can't run the GC, silently fail and return that no GC was run.
  .....GC过程
  // Enqueue cleared references.
  reference_processor_->EnqueueClearedReferences(self);
  // Grow the heap so that we know when to perform the next GC.
  GrowForUtilization(collector, bytes_allocated_before_gc);
  LogGC(gc_cause, collector);
  FinishGC(self, gc_type);
  // Inform DDMS that a GC completed.
  Dbg::GcDidFinish();
  ......
  return gc_type;
}
```
在Android系统GC完成时会调用`ReferenceProcessor::EnqueueClearedReferences`来把回收掉的引用放入引用队列中。

```/art/runtime/gc/reference_processor.cc ```
```cpp
void ReferenceProcessor::EnqueueClearedReferences(Thread* self) {
  Locks::mutator_lock_->AssertNotHeld(self);
  // When a runtime isn't started there are no reference queues to care about so ignore.
  if (!cleared_references_.IsEmpty()) {
    if (LIKELY(Runtime::Current()->IsStarted())) {
      jobject cleared_references;
      {
        ReaderMutexLock mu(self, *Locks::mutator_lock_);
        cleared_references = self->GetJniEnv()->vm->AddGlobalRef(
            self, cleared_references_.GetList());
      }
      if (kAsyncReferenceQueueAdd) {
        // TODO: This can cause RunFinalization to terminate before newly freed objects are
        // finalized since they may not be enqueued by the time RunFinalization starts.
        Runtime::Current()->GetHeap()->GetTaskProcessor()->AddTask(
            self, new ClearedReferenceTask(cleared_references));
      } else {
        ClearedReferenceTask task(cleared_references);
        task.Run(self);
      }
    }
    cleared_references_.Clear();
  }
}
```

`ReferenceProcessor`主要通过`Thread`对象获取了回收的引用对象，创建了一个`ClearedReferenceTask`，这个task就是真正把回收的对象放入引用队列的方法

```/art/runtime/gc/reference_processor.cc ```
```cpp
class ClearedReferenceTask : public HeapTask {
 public:
  explicit ClearedReferenceTask(jobject cleared_references)
      : HeapTask(NanoTime()), cleared_references_(cleared_references) {
  }
  virtual void Run(Thread* thread) {
    ScopedObjectAccess soa(thread);
    jvalue args[1];
    args[0].l = cleared_references_;
    InvokeWithJValues(soa, nullptr, WellKnownClasses::java_lang_ref_ReferenceQueue_add, args);
    soa.Env()->DeleteGlobalRef(cleared_references_);
  }

 private:
  const jobject cleared_references_;
};
```
`ClearedReferenceTask`主要作用就是回调Java层方法的相应方法

``` ReferenceQueue  ```
```java
static void add(Reference<?> list) {
    synchronized (ReferenceQueue.class) {
        if (unenqueued == null) {
            unenqueued = list;
        } else {
            // Find the last element in unenqueued.
            Reference<?> last = unenqueued;
            while (last.pendingNext != unenqueued) {
              last = last.pendingNext;
            }
            // Add our list to the end. Update the pendingNext to point back to enqueued.
            last.pendingNext = list;
            last = list;
            while (last.pendingNext != list) {
                last = last.pendingNext;
            }
            last.pendingNext = unenqueued;
        }
        ReferenceQueue.class.notifyAll();
    }
}
```
到此为止，被回收的对象已经回到java层的`ReferenceQueue` 中，但是只是作为静态变量存储了，并没有进入相应的引用队列中，这个就得说起另外一个东西了,Deamons守护线程
```cpp
bool Runtime::Start() {
    .....
    StartDaemonThreads();
    ......
}

Runtime::StartDaemonThreads() {
    ScopedTrace trace(__FUNCTION__);
    VLOG(startup) << "Runtime::StartDaemonThreads entering";

    Thread* self = Thread::Current();

    // Must be in the kNative state for calling native methods.
    CHECK_EQ(self->GetState(), kNative);

    JNIEnv* env = self->GetJniEnv();
    env->CallStaticVoidMethod(WellKnownClasses::java_lang_Daemons,
                            WellKnownClasses::java_lang_Daemons_start);
    if (env->ExceptionCheck()) {
        env->ExceptionDescribe();
        LOG(FATAL) << "Error starting java.lang.Daemons";
    }

    VLOG(startup) << "Runtime::StartDaemonThreads exiting";
}

```
在Android Runtime的start方法中会调用一个`StartDaemonThreads`方法，最终会调用到java层`Daemons.start`方法。
``` java
public static void start() {
    //引用队列守护线程
    ReferenceQueueDaemon.INSTANCE.start();
    //析构守护线程
    FinalizerDaemon.INSTANCE.start();
    //析构监护守护线程
    FinalizerWatchdogDaemon.INSTANCE.start();
    //守护堆任务线程
    HeapTaskDaemon.INSTANCE.start();
}
```
在Daemons中有好几个守护线程，我们主要关注的是`ReferenceQueueDaemon`引用队列守护线程
```java
public void runInternal() {
    while (isRunning()) {
        Reference<?> list;
        try {
            synchronized (ReferenceQueue.class) {
                while (ReferenceQueue.unenqueued == null) {
                    ReferenceQueue.class.wait();
                }
                list = ReferenceQueue.unenqueued;
                ReferenceQueue.unenqueued = null;
            }
        } catch (InterruptedException e) {
            continue;
        } catch (OutOfMemoryError e) {
            continue;
        }
        ReferenceQueue.enqueuePending(list);
    }
}
```
在`ReferenceQueueDaemon.runInternal`方法中会获取前面GC后存入`ReferenceQueue.unenqueued`然后把引用分别放入每个`ReferenceQueue`中，Over！

### MessageQueue
MessageQueue用于Handle机制中传递消息，但是在Handle机制中我们关注的是`MessageQueue.next`方法的前半部分,
```java
Message next() {
    ......
    for (;;) {
        ......
        synchronized (this) {
            //get Message and return

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```
上面段代码就是在通过`next`没有获取到下一条`Message`时运行，整段代码整体就是一个观察者默认，其它对象可以注策`IdleHandler`到`MessageQueue`中，当`MessageQueue`中没有消息时则会调用`IdleHandler`的`queueIdle`方法。

什么时候`MessageQueue`为空？
* 对于主线层来说，当页面渲染完成且没有新的交互时`MessageQueue`才会为空。
* 对于子线程，连续操作之后会有

目前Glide使用`IdleHandler`来进行弱引用缓存key的清理
