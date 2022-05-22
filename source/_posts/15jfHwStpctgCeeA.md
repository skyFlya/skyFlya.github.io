---
title: 源码学习 - Glide
tags:
  - 源码解析
categories: Android
updated: 1635686911000
date: 2021-10-31 21:29:17
---

```java
Glide.with(fragment)
    .load(myUrl)
    .into(imageView);
```

![5oUW5R.png](https://z3.ax1x.com/2021/10/26/5oUW5R.png)

### with()

> 用Method初始化glide的一些必需的环境，然后调用Requestmanagerretriver的`get()`获取requestManager。如果传入的对象是全局Context，你就不需要处理生命周期;如果输入是具有生命周期的View(包含Frg或Act)则将添加一个隐藏的Fragment来感知生命周期

<!-- more -->
```java
public static RequestManager with(@NonNull Context context) {  
  return getRetriever(context).get(context);  
}
```
#### getRetriever() 

> RequestManagerRetriever,用于创建新的RequestManager或从Activity和Fragment中检索现有的

```java
  @NonNull
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    // 由于其他原因，上下文可能为空（即用户传入空值），但实际上它只会由于 Fragment 生命周期的错误而发生。
    Preconditions.checkNotNull(
        context,"You cannot start a load on a not yet attached View or a Fragment where getActivity() "
            + "returns null (which usually occurs when getActivity() is called before the Fragment "
            + "is attached or after the Fragment is destroyed).");
    // 获取RequestManagerRetriever
    return Glide.get(context).getRequestManagerRetriever();
  }
```

#### get()

> 单例实现Glide的初始化

```kotlin
  //双重检查锁定在这里是安全的
  public static Glide get(@NonNull Context context) {
    if (glide == null) {
      // 通过反射GeneratedAppGlideModuleImpl实例化Glide
      GeneratedAppGlideModule annotationGeneratedModule =
          getAnnotationGeneratedGlideModules(context.getApplicationContext()); 
      synchronized (Glide.class) {
        if (glide == null) {
          checkAndInitializeGlide(context, annotationGeneratedModule);
        }
      }
    }
    return glide;
  }
```

#### RequestManagerRetriever.get() 

> 创建对应生命周期的RequestManager 

- 首先判断是在子线程,则拿一个全Context然后在工厂模式创建下RequestManager,所以推荐不要在子线程执行此操作
- 如为FragmentActivity,则通过FragmentManager,创建一个空Fragment放进当前`Fragment`或者`Activity`,这样就可以感知宿主的生命周期,然后在工厂模式创建下RequestManager 
- 如为Activity...
- 如为ContextWrapper...

```java
  public RequestManager get(@NonNull Activity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else if (activity instanceof FragmentActivity) {
      return get((FragmentActivity) activity);
    } else {
      assertNotDestroyed(activity);
      frameWaiter.registerSelf(activity);
      android.app.FragmentManager fm = activity.getFragmentManager();
      return fragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }
  // FragmentActivity Simple 
  @NonNull
  public RequestManager get(@NonNull FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      frameWaiter.registerSelf(activity);
      FragmentManager fm = activity.getSupportFragmentManager();
      return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }
  // 通过supportFragment感知创建
  private RequestManager supportFragmentGet(
      @NonNull Context context,
      @NonNull FragmentManager fm,
      @Nullable Fragment parentHint,
      boolean isParentVisible) {
    SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      // 工厂模式创建
      Glide glide = Glide.get(context);
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      // 这是一个小技巧，我们将启动 RequestManager，而不是相应的 Lifecycle。启动 RequestManager 是安全的，但启动 Lifecycle 可能会引发内存泄漏
      if (isParentVisible) {
        requestManager.onStart();
      }
      current.setRequestManager(requestManager);
    }
    return requestManager;
  }
```

### load()

> 对全局变量赋值,构建ReqeustBuilder

```java
  // Drawable Simple
  public RequestBuilder<Drawable> load(@Nullable Drawable drawable) {
    return asDrawable().load(drawable);
  }
```

### into()


设置资源到加载的ImageView中 ，取消任何现有的加载，并释放 Glide 之前可能加载到ImageView的任何资源，以便它们可以被重用
```java
// 设置资源配置到TargetView
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    Util.assertMainThread();
    ···
    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
  }
// 创建TargetView
public <X> ViewTarget<ImageView, X> buildImageViewTarget(
      @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
    return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
  }
// 负责为指定的android.view.View子类生成正确类型的Target工厂
public class ImageViewTargetFactory {
  @NonNull
  @SuppressWarnings("unchecked")
  public <Z> ViewTarget<ImageView, Z> buildTarget(
      @NonNull ImageView view, @NonNull Class<Z> clazz) {
    if (Bitmap.class.equals(clazz)) {
      return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
    } else if (Drawable.class.isAssignableFrom(clazz)) {
      return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
    } else {
      throw new IllegalArgumentException(
          "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
    }
  }
}
```

#### into()

> 核心代码加载代码，看起来简单但实现起来复杂。
首先看看 buildRequest 如何初始化 request 

```java
  private <Y extends Target<TranscodeType>> Y into(@NonNull Y target, @Nullable RequestListener<TranscodeType> targetListener, BaseRequestOptions<?> options, Executor callbackExecutor) {
    Preconditions.checkNotNull(target);
    if (!isModelSet) { throw new IllegalArgumentException("You must call #load() before calling #into()"); }
    // 现在的请求
    Request request = buildRequest(target, targetListener, options, callbackExecutor);
    // 之前的请求
    Request previous = target.getRequest();
    // 如果之前的请求完成，重新开始以重新传递结果，触发 RequestListeners 和 Targets。如果请求失败，将重新请求，
    if (request.isEquivalentTo(previous) && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
      // 如果之前的请求已经在运行，我们可以让它继续运行而不中断
      if (!Preconditions.checkNotNull(previous).isRunning()) {
        // 使用先前的请求而不是新的请求来优化，例如跳过设置占位符、跟踪和取消跟踪目标以及获取在单个请求中完成的视图维度
        previous.begin();
      }
      return target;
    }
    requestManager.clear(target);
    target.setRequest(request);
    requestManager.track(target, request);
    return target;
  }
```


#### RequestBuilder.buildRequest()

> 根据存在的场景建立不同Request

```java
  private Request buildRequest(Target<TranscodeType> target, @Nullable RequestListener<TranscodeType> targetListener, BaseRequestOptions<?> requestOptions, Executor callbackExecutor) {
    return buildRequestRecursive(/*requestLock=*/ new Object(), target, targetListener,/*parentCoordinator=*/ null, transitionOptions, requestOptions.getPriority(), requestOptions.getOverrideWidth(), requestOptions.getOverrideHeight(), requestOptions, callbackExecutor);
  }
  
  private Request buildRequestRecursive(Object requestLock, Target<TranscodeType> target, @Nullable RequestListener<TranscodeType> targetListener, @Nullable RequestCoordinator parentCoordinator, TransitionOptions<?, ? super TranscodeType> transitionOptions, Priority priority, int overrideWidth, int overrideHeight, BaseRequestOptions<?> requestOptions, Executor callbackExecutor) {
    // 如有必要首先构建 ErrorRequestCoordinator，以便我们可以更新 parentCoordinator。
    ErrorRequestCoordinator errorRequestCoordinator = null;
    if (errorBuilder != null) {
      errorRequestCoordinator = new ErrorRequestCoordinator(requestLock, parentCoordinator);
      parentCoordinator = errorRequestCoordinator;
    }

    Request mainRequest = buildThumbnailRequestRecursive(requestLock, target, targetListener, parentCoordinator, transitionOptions, priority, overrideWidth, overrideHeight, requestOptions, callbackExecutor);

    if (errorRequestCoordinator == null) return mainRequest;

    int errorOverrideWidth = errorBuilder.getOverrideWidth();
    int errorOverrideHeight = errorBuilder.getOverrideHeight();
    if (Util.isValidDimensions(overrideWidth, overrideHeight) && !errorBuilder.isValidOverride()) {
      errorOverrideWidth = requestOptions.getOverrideWidth();
      errorOverrideHeight = requestOptions.getOverrideHeight();
    }

    Request errorRequest = errorBuilder.buildRequestRecursive(requestLock, target, targetListener, errorRequestCoordinator, errorBuilder.transitionOptions, errorBuilder.getPriority(), errorOverrideWidth, errorOverrideHeight, errorBuilder, callbackExecutor);
    errorRequestCoordinator.setRequests(mainRequest, errorRequest);
    return errorRequestCoordinator;
  }
```

#### RequestBuilder.buildThumbnailRequestRecursive()

> 根据是否需要缩略图,生成各种不同的Request

这里经过一层又一层最终拿到一个`SingleRequest`。

```java
  private Request buildThumbnailRequestRecursive(Object requestLock, Target<TranscodeType> target, RequestListener<TranscodeType> targetListener, @Nullable RequestCoordinator parentCoordinator, TransitionOptions<?, ? super TranscodeType> transitionOptions, Priority priority, int overrideWidth, int overrideHeight, BaseRequestOptions<?> requestOptions, Executor callbackExecutor) {
    if (thumbnailBuilder != null) {
      // 递归案例：包含一个潜在的递归缩略图Request Builder
      if (isThumbnailBuilt) {
        throw new IllegalStateException(
            "You cannot use a request as both the main request and a "
                + "thumbnail, consider using clone() on the request(s) passed to thumbnail()");
      }

      TransitionOptions<?, ? super TranscodeType> thumbTransitionOptions = thumbnailBuilder.transitionOptions;
      // 默认情况下我们的将过渡用在缩略图，但避免覆盖可能已明确应用于缩略图请求的自定义选项。
      if (thumbnailBuilder.isDefaultTransitionOptionsSet) thumbTransitionOptions = transitionOptions;
      Priority thumbPriority = thumbnailBuilder.isPrioritySet() ? thumbnailBuilder.getPriority() : getThumbnailPriority(priority);
      int thumbOverrideWidth = thumbnailBuilder.getOverrideWidth();
      int thumbOverrideHeight = thumbnailBuilder.getOverrideHeight();
      if (Util.isValidDimensions(overrideWidth, overrideHeight) && !thumbnailBuilder.isValidOverride()) {
        thumbOverrideWidth = requestOptions.getOverrideWidth();
        thumbOverrideHeight = requestOptions.getOverrideHeight();
      }

      ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(requestLock, parentCoordinator);
      Request fullRequest = obtainRequest(requestLock, target, targetListener, requestOptions, coordinator, transitionOptions, priority, overrideWidth, overrideHeight, callbackExecutor);
      isThumbnailBuilt = true;
      // 递归生成缩略图请求
      Request thumbRequest = thumbnailBuilder.buildRequestRecursive(requestLock, target,targetListener, coordinator, thumbTransitionOptions, thumbPriority, thumbOverrideWidth, thumbOverrideHeight, thumbnailBuilder, callbackExecutor);
      isThumbnailBuilt = false;
      coordinator.setRequests(fullRequest, thumbRequest);
      return coordinator;
    } else if (thumbSizeMultiplier != null) {
      // 基本情况：缩略图Multiplier生成缩略图请求，但不能递归。
      ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(requestLock, parentCoordinator);
      Request fullRequest = obtainRequest(requestLock, target, targetListener, requestOptions, coordinator, transitionOptions, priority, overrideWidth, overrideHeight, callbackExecutor);
      BaseRequestOptions<?> thumbnailOptions = requestOptions.clone().sizeMultiplier(thumbSizeMultiplier);
      Request thumbnailRequest = obtainRequest(requestLock, target, targetListener, thumbnailOptions, coordinator, transitionOptions, getThumbnailPriority(priority), overrideWidth, overrideHeight, callbackExecutor);
      coordinator.setRequests(fullRequest, thumbnailRequest);
      return coordinator;
    } else {
      // 基本情况：没有缩略图
      return obtainRequest(requestLock, target, targetListener, requestOptions, parentCoordinator, transitionOptions, priority, overrideWidth, overrideHeight, callbackExecutor);
    }
  }
  private Request obtainRequest(Object requestLock, Target<TranscodeType> target, RequestListener<TranscodeType> targetListener, BaseRequestOptions<?> requestOptions, RequestCoordinator requestCoordinator, TransitionOptions<?, ? super TranscodeType> transitionOptions, Priority priority, int overrideWidth, int overrideHeight, Executor callbackExecutor) {
    return SingleRequest.obtain(context, glideContext, requestLock, model, transcodeClass, requestOptions, overrideWidth, overrideHeight, priority, target, targetListener, requestListeners, requestCoordinator, glideContext.getEngine(), transitionOptions.getTransitionFactory(), callbackExecutor);
  }
```

#### RequestManager.track()
这里Glide将判断请求是否需要显示，如果它需要现在显示则开始执行，否则clear( )，并将请求放入队列。这种设计更精巧，省电
```java
synchronized void track(@NonNull Target<?> target, @NonNull Request request) {  
  targetTracker.track(target);  
  requestTracker.runRequest(request);  
}
/** 开始跟踪给定的请求 */
public void runRequest(@NonNull Request request) {
  requests.add(request);
  if (!isPaused) {
    // 启动异步加载
    request.begin();
  } else {
    // 防止从以前的请求加载任何位图，释放此请求持有的任何资源，显示当前占位符（如果提供），并将请求标记为已取消
    request.clear();
    pendingRequests.add(request);
  }
}
```


#### SingleRequest.begin()

```java
  public void begin() {
    synchronized (requestLock) {
      assertNotCallingCallbacks();
      stateVerifier.throwIfRecycled();
      startTime = LogTime.getLogTime();
      if (model == null) {
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
          width = overrideWidth;
          height = overrideHeight;
        }
        // 如果用户设置可回调的Drawables,这里进行日志反馈
        int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
        onLoadFailed(new GlideException("Received null model"), logLevel);
        return;
      }
      if (status == Status.RUNNING) throw new IllegalArgumentException("Cannot restart a running request");
      // 如果我们在完成后重新启动(通常是通过notifyDataSetChanged之类的方式，向相同的目标或视图启动相同的请求)，我们可以简单地使用上次检索到的资源和大小，而跳过获取一个新的大小，开始一个新的加载等(这意味着希望重新启动加载的用户需要在开始新的加载之前显式地清除view或Target，因为他们觉得视图大小已经改变。)
      if (status == Status.COMPLETE) {
        onResourceReady(resource, DataSource.MEMORY_CACHE, /* isLoadedFromAlternateCacheKey= */ false);
        return;
      }
      // 对于既未完成也未运行的请求，重新启动可以被视为新请求，并可以从头开始运行
      cookie = GlideTrace.beginSectionAsync(TAG);
      status = Status.WAITING_FOR_SIZE;
      // 如果宽高已指定,则回调onSizeReady ,否则再获取宽高
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        onSizeReady(overrideWidth, overrideHeight);
      } else {
        target.getSize(this);
      }
      if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE) && canNotifyStatusChanged()) {
        target.onLoadStarted(getPlaceholderDrawable());
      }
    }
  }
```

- onLoadFailed : 图片或者资源为空,报错回调
- onResourceReady: 最终通过`Engine.release( )`,释放资源
- onSizeReady: View大小已明确 , 执行`Engine.load()`加载资源
- getSize: 获取View大小
- onLoadStarted: 等待或运行中,占位图处理


#### SingleRequest.onSizeReady( )

> 启动给定参数进行图片的加载, 必须在主线程上调用

```java
  @Override
  public void onSizeReady(int width, int height) {
    stateVerifier.throwIfRecycled();
    synchronized (requestLock) {
      if (status != Status.WAITING_FOR_SIZE) {
        return;
      }
      status = Status.RUNNING;
      float sizeMultiplier = requestOptions.getSizeMultiplier();
      this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
      this.height = maybeApplySizeMultiplier(height, sizeMultiplier);
      loadStatus = engine.load(glideContext, model, requestOptions.getSignature(), this.width, this.height, requestOptions.getResourceClass(), transcodeClass, priority, requestOptions.getDiskCacheStrategy(), requestOptions.getTransformations(), requestOptions.isTransformationRequired(), requestOptions.isScaleOnlyOrNoTransform(), requestOptions.getOptions(), requestOptions.isMemoryCacheable(), requestOptions.getUseUnlimitedSourceGeneratorsPool(), requestOptions.getUseAnimationPool(), requestOptions.getOnlyRetrieveFromCache(), this, callbackExecutor);
    }
  }
```


#### Engine.load( )

> 到这里想都不用想了,Engine才是真正加载图片的Class,
> Engine负责执行图片加载和管理活动资源和缓存资源。

活动资源是指那些至少提供一个请求但未释放的资源。一旦资源的所有使用者都释放了该资源，该资源就会进入缓存。如果资源从缓存中返回给新的使用者，它将被重新添加到活动资源中。  
如果从缓存中移除资源，它的资源将被回收和重用(如果可能的话)，资源将被丢弃。没有严格要求消费者释放他们的资源，所以活跃的资源被弱持有。

请求流程:
- 检查当前使用的资源集，返回活动资源（如果存在），并将任何新的非活动资源移动到内存缓存中
- 检查内存缓存并提供缓存资源（如果存在）
- 检查当前的一组正在进行的加载并将 cb 添加到进行中的加载（如果存在）
- 开始加载

```java
  public <R> LoadStatus load(GlideContext glideContext, Object model, Key signature, int width, int height, Class<?> resourceClass, Class<R> transcodeClass, Priority priority, DiskCacheStrategy diskCacheStrategy, Map<Class<?>, Transformation<?>> transformations, boolean isTransformationRequired, boolean isScaleOnlyOrNoTransform, Options options, boolean isMemoryCacheable, boolean useUnlimitedSourceExecutorPool, boolean useAnimationPool, boolean onlyRetrieveFromCache, ResourceCallback cb, Executor callbackExecutor) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,resourceClass, transcodeClass, options);
    EngineResource<?> memoryResource;
    synchronized (this) {
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);
      // 判断是否有缓存,有则直接加载
      if (memoryResource == null) {
        return waitForExistingOrStartNewJob(glideContext, model, signature, width, height, resourceClass, transcodeClass, priority, diskCacheStrategy, transformations, isTransformationRequired, isScaleOnlyOrNoTransform, options, isMemoryCacheable, useUnlimitedSourceExecutorPool, useAnimationPool, onlyRetrieveFromCache, cb, callbackExecutor, key, startTime);
      }
    }
    // 避免在保持Engine锁时回调，因为这样做会更容易死锁
    cb.onResourceReady(memoryResource, DataSource.MEMORY_CACHE, /* isLoadedFromAlternateCacheKey= */ false);
    return null;
  }
```

#### Engine.waitForExistingOrStartNewJob( )

> 等待或者执行任务

```java
  private <R> LoadStatus waitForExistingOrStartNewJob(GlideContext glideContext, Object model, Key signature, int width, int height, Class<?> resourceClass, Class<R> transcodeClass, Priority priority, DiskCacheStrategy diskCacheStrategy, Map<Class<?>, Transformation<?>> transformations, boolean isTransformationRequired, boolean isScaleOnlyOrNoTransform, Options options, boolean isMemoryCacheable, boolean useUnlimitedSourceExecutorPool, boolean useAnimationPool, boolean onlyRetrieveFromCache, ResourceCallback cb, Executor callbackExecutor, EngineKey key, long startTime) {
    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      current.addCallback(cb, callbackExecutor);
      return new LoadStatus(cb, current);
    }

    EngineJob<R> engineJob = engineJobFactory.build(key, isMemoryCacheable, useUnlimitedSourceExecutorPool, useAnimationPool, onlyRetrieveFromCache);
    DecodeJob<R> decodeJob = decodeJobFactory.build(glideContext, model, key, signature, width, height, resourceClass, transcodeClass, priority, diskCacheStrategy, transformations, isTransformationRequired, isScaleOnlyOrNoTransform, onlyRetrieveFromCache, options, engineJob);
    jobs.put(key, engineJob);
    engineJob.addCallback(cb, callbackExecutor);
    engineJob.start(decodeJob);
    if (VERBOSE_IS_LOGGABLE)  logWithTimeAndKey("Started new load", startTime, key);
    return new LoadStatus(cb, engineJob);
  }
```

#### DecodeJob

> 负责从缓存数据或原始源中解码资源并应用转换和转码的类

```java
  @Override  
  public void run() {
      if (isCancelled) {
        notifyFailed();
        return;
      }
      runWrapped();
  }

  private void runWrapped() {
    // 为了判断运行原因,这里做了三个判断
    switch (runReason) {
      case INITIALIZE:
        // 获取当前解码数据的阶段
        stage = getNextStage(Stage.INITIALIZE);
        // 获取数据生成器
        currentGenerator = getNextGenerator();
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE:
        runGenerators();
        break;
      case DECODE_DATA:
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
  }
```

##### runGenerators( )

currentGenerator实现DataFetcherGenerator接口，这个接口主要用来生成一系列的modelLoader和model

目前Glide有三种类生成器
- ResourceCacheGenerator
- DataCacheGenerator
- SourceGenerator

```java
  private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    // startNext -> 尝试单个新的DataFetcher，如果DataFetcher已启动则返回true，否则返回false
    while (!isCancelled && currentGenerator != null && !(isStarted = currentGenerator.startNext())) {
      stage = getNextStage(stage);
      currentGenerator = getNextGenerator();
      if (stage == Stage.SOURCE) {
        reschedule();
        return;
      }
    }
    // We've run out of stages and generators, give up.
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
      notifyFailed();
    }
  }
  
  private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
      case RESOURCE_CACHE:
        return new ResourceCacheGenerator(decodeHelper, this);
      case DATA_CACHE:
        return new DataCacheGenerator(decodeHelper, this);
      case SOURCE:
        return new SourceGenerator(decodeHelper, this);
      case FINISHED:
        return null;
      default:
        throw new IllegalStateException("Unrecognized stage: " + stage);
    }
  }

```

###### SourceGenerator.startNext()

将首先判断，缓存如果它不是空的，调用 cacheData; 否则，获取 loadData，然后执行 startNextLoad ()
```java
  public boolean startNext() {
    if (dataToCache != null) {
      Object data = dataToCache;
      dataToCache = null;
      try {
        boolean isDataInCache = cacheData(data);
        // 如果我们没有将数据写入缓存，cacheData方法将尝试直接解码原始数据，而不是通过磁盘缓存。因为此时cacheData已经调用了我们的回调函数，所以除了返回，没有其他事情可做了
        if (!isDataInCache) {
          return true;
        }
        // 如果我们能够成功地将数据写入缓存，那么现在需要继续调用下面的sourceCacheGenerator来从缓存加载数据
      } catch (IOException e) {
        // IOException意味着我们无法将数据写入缓存，或者在磁盘缓存写入失败后无法倒带数据。无论哪种情况，我们都可以继续尝试下面的下一个取回
        if (Log.isLoggable(TAG, Log.DEBUG)) {
          Log.d(TAG, "Failed to properly rewind or write data to cache", e);
        }
      }
    }

    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
      return true;
    }
    sourceCacheGenerator = null;

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      loadData = helper.getLoadData().get(loadDataListIndex++);
      if (loadData != null
          && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
              || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
        started = true;
        startNextLoad(loadData);
      }
    }
    return started;
  }
```

###### SourceGenerator.cacheData

```java
  /**
   * 如果我们能够缓存数据，应该尝试直接从缓存解码数据，如果我们不能缓存数据，应该尝试从源解码，则返回false
   */
  private boolean cacheData(Object dataToCache) throws IOException {
    boolean isLoadingFromSourceData = false;
    try {
      DataRewinder<Object> rewinder = helper.getRewinder(dataToCache);
      Object data = rewinder.rewindAndGet();
      Encoder<Object> encoder = helper.getSourceEncoder(data);
      DataCacheWriter<Object> writer = new DataCacheWriter<>(encoder, data, helper.getOptions());
      DataCacheKey newOriginalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());
      DiskCache diskCache = helper.getDiskCache();
      diskCache.put(newOriginalKey, writer);
      if (diskCache.get(newOriginalKey) != null) {
        originalKey = newOriginalKey;
        sourceCacheGenerator = new DataCacheGenerator(Collections.singletonList(loadData.sourceKey), helper, this);
        // 我们能够将数据写入缓存
        return true;
      } else {
        isLoadingFromSourceData = true;
        cb.onDataFetcherReady(loadData.sourceKey, rewinder.rewindAndGet(), loadData.fetcher, loadData.fetcher.getDataSource(), loadData.sourceKey);
      }
      // 写入数据到缓存失败的处理
      return false;
    } finally {
      if (!isLoadingFromSourceData) {
        loadData.fetcher.cleanup();
      }
    }
  }
```

###### DecodeHelper.getSourceEncoder()

DecodeHelper.java
```java
  <X> Encoder<X> getSourceEncoder(X data) throws Registry.NoSourceEncoderAvailableException {
    return glideContext.getRegistry().getSourceEncoder(data);
  }
```

Registry.java
```java
public <X> Encoder<X> getSourceEncoder(@NonNull X data) throws NoSourceEncoderAvailableException {  
  Encoder<X> encoder = encoderRegistry.getEncoder((Class<X>) data.getClass());  
  if(encoder != null) {  
    return encoder;  
  }  
  throw new NoSourceEncoderAvailableException(data.getClass());  
}
```

###### Register

>  每个数据类型对应一个编码器,Register就是用来记录这些的

```java
registry
        .append(ByteBuffer.class, new ByteBufferEncoder())
        .append(InputStream.class, new StreamEncoder(arrayPool))
        .append(Registry.BUCKET_BITMAP, ByteBuffer.class, Bitmap.class, byteBufferBitmapDecoder)
        .append(Registry.BUCKET_BITMAP, InputStream.class, Bitmap.class, streamBitmapDecoder);
```

##### decodeFromRetrievedData()

> 处理返回的数据

![5oU7rD.png](https://z3.ax1x.com/2021/10/26/5oU7rD.png)
```java
  private <Data> Resource<R> decodeFromData(
      DataFetcher<?> fetcher, Data data, DataSource dataSource) throws GlideException {
    try {
      if (data == null) {
        return null;
      }
      long startTime = LogTime.getLogTime();
      Resource<R> result = decodeFromFetcher(data, dataSource);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Decoded result " + result, startTime);
      }
      return result;
    } finally {
      fetcher.cleanup();
    }
  }

  @SuppressWarnings("unchecked")
  private <Data> Resource<R> decodeFromFetcher(Data data, DataSource dataSource)
      throws GlideException {
    LoadPath<Data, ?, R> path = decodeHelper.getLoadPath((Class<Data>) data.getClass());
    return runLoadPath(data, dataSource, path);
  }
```

#### EngineJob

> 通过添加和删除加载回调并在加载完成时通知回调来管理加载的类(管理加载过程中的一些回调)

GlideExecutor是一个继承的Executorservice类，它显然是一个线程池。这里通过decodeJob来确定是否从缓存解析，如果是从缓存解析，调用diskCacheExecutor，否则，调用getActiveSourceExecutor

- willDecodeFromCache
	- 如果此作业将尝试从磁盘缓存解码资源，则返回true
	- 如果始终从源解码，则返回false

```java
  public synchronized void start(DecodeJob<R> decodeJob) {
    this.decodeJob = decodeJob;
    GlideExecutor executor = decodeJob.willDecodeFromCache() ? diskCacheExecutor : getActiveSourceExecutor();
    // 执行decodeJob线程任务
    executor.execute(decodeJob);
  }
```

