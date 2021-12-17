# 第一行代码

[Fresco](https://github.com/facebook/fresco) 最简单和入门级的 API 是 `SimpleDraweeView.setImageURI(uri)`，那就先从这个方法走下去看看会遇到哪些概念

```java
SimpleDraweeView.setImageURI(uri)
SimpleDraweeView.setImageURI(uri, null)
DraweeView.setController
DraweeHolder.setController
DraweeHolder.attachController
AbstractDraweeController.onAttach
AbstractDraweeController.submitRequest

class AbstractDraweeController {  
  protected void submitRequest() {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("AbstractDraweeController#submitRequest");
    }

    // 检查缓存，有则直接返回缓存数据，没有则执行抓取操作；现在先不理缓存逻辑
    final T closeableImage = getCachedImage();    
    if (closeableImage != null) {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.beginSection("AbstractDraweeController#submitRequest->cache");
      }
      mDataSource = null;
      mIsRequestSubmitted = true;
      mHasFetchFailed = false;
      mEventTracker.recordEvent(Event.ON_SUBMIT_CACHE_HIT);
      reportSubmit(mDataSource, getImageInfo(closeableImage));
      onImageLoadedFromCacheImmediately(mId, closeableImage);
      onNewResultInternal(mId, mDataSource, closeableImage, 1.0f, true, true, true);
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
      return;
    }
    mEventTracker.recordEvent(Event.ON_DATASOURCE_SUBMIT);
    mSettableDraweeHierarchy.setProgress(0, true);
    mIsRequestSubmitted = true;
    mHasFetchFailed = false;
    mDataSource = getDataSource();    // 重要
    reportSubmit(mDataSource, null);
    if (FLog.isLoggable(FLog.VERBOSE)) {
      FLog.v(
          TAG,
          "controller %x %s: submitRequest: dataSource: %x",
          System.identityHashCode(this),
          mId,
          System.identityHashCode(mDataSource));
    }

    // 从数据源抓取，callback 模式，有三种情况：成功、失败和更新进度
    final String id = mId;
    final boolean wasImmediate = mDataSource.hasResult();
    final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            // isFinished must be obtained before image, otherwise we might set intermediate result
            // as final image.
            boolean isFinished = dataSource.isFinished();
            boolean hasMultipleResults = dataSource.hasMultipleResults();
            float progress = dataSource.getProgress();
            T image = dataSource.getResult();
            if (image != null) {
              onNewResultInternal(
                  id, dataSource, image, progress, isFinished, wasImmediate, hasMultipleResults);
            } else if (isFinished) {
              onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
            }
          }

          @Override
          public void onFailureImpl(DataSource<T> dataSource) {
            onFailureInternal(id, dataSource, dataSource.getFailureCause(), /* isFinished */ true);
          }

          @Override
          public void onProgressUpdate(DataSource<T> dataSource) {
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            onProgressUpdateInternal(id, dataSource, progress, isFinished);
          }
        };
    mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
  }    
}

class AbstractDataSource {
  private enum DataSourceStatus {
    // data source has not finished yet
    IN_PROGRESS,

    // data source has finished with success
    SUCCESS,

    // data source has finished with failure
    FAILURE,
  }

  // 数据源有三种状态：行进中、成功和失败，一般情况下会是还没开始抓取操作也就是【行进中】
  // callback 也就是订阅者被数据源保存起来以便后续通知
  public void subscribe(final DataSubscriber<T> dataSubscriber, final Executor executor) {
    Preconditions.checkNotNull(dataSubscriber);
    Preconditions.checkNotNull(executor);
    boolean shouldNotify;

    synchronized (this) {
      if (mIsClosed) {
        return;
      }

      if (mDataSourceStatus == DataSourceStatus.IN_PROGRESS) {
        mSubscribers.add(Pair.create(dataSubscriber, executor));
      }

      shouldNotify = hasResult() || isFinished() || wasCancelled();
    }

    if (shouldNotify) {
      notifyDataSubscriber(dataSubscriber, executor, hasFailed(), wasCancelled());
    }
  }    
}

// 到目前为止，只看到了 callback 但没看到是什么时候开始抓取数据（比如网络请求操作）
// 但看到 DataSource 的描述，隐隐感觉到网络请求操作很可能已经发出，而 DataSource 就是代表这个异步操作（类似于 Future）
// 它作为绳子的一端连接着另一端的 fetch task，从这端出发按图索骥即可找到 fetch task

/**
 * An alternative to Java Futures for the image pipeline.
 */
public interface DataSource<T>

SimpleDraweeView.setImageURI(uri)
SimpleDraweeView.setImageURI(uri, null)
AbstractDraweeControllerBuilder.build
AbstractDraweeController.buildController
PipelineDraweeControllerBuilder.obtainController
AbstractDraweeControllerBuilder.obtainDataSourceSupplier
AbstractDraweeControllerBuilder.getDataSourceSupplierForRequest(controller, controllerId, imageRequest)
AbstractDraweeControllerBuilder.getDataSourceSupplierForRequest(controller, controllerId, imageRequest, CacheLevel.FULL_FETCH)
PipelineDraweeControllerBuilder.getDataSourceForRequest

class ImagePipeline {
  public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
      ImageRequest imageRequest,
      @Nullable Object callerContext,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit,
      @Nullable RequestListener requestListener,
      @Nullable String uiComponentId) {
    try {
      Producer<CloseableReference<CloseableImage>> producerSequence =
          mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);    // Producer 生产者负责从网络抓取图片
      return submitFetchRequest(                                                     // 在这里将任务提交到任务队列等待执行
          producerSequence,
          imageRequest,
          lowestPermittedRequestLevelOnSubmit,
          callerContext,
          requestListener,
          uiComponentId);
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    }
  }

  private <T> DataSource<CloseableReference<T>> submitFetchRequest(
      Producer<CloseableReference<T>> producerSequence,
      ImageRequest imageRequest,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit,
      @Nullable Object callerContext,
      @Nullable RequestListener requestListener,
      @Nullable String uiComponentId) {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("ImagePipeline#submitFetchRequest");
    }
    final RequestListener2 requestListener2 =
        new InternalRequestListener(
            getRequestListenerForRequest(imageRequest, requestListener), mRequestListener2);

    if (mCallerContextVerifier != null) {
      mCallerContextVerifier.verifyCallerContext(callerContext, false);
    }

    try {
      ImageRequest.RequestLevel lowestPermittedRequestLevel =
          ImageRequest.RequestLevel.getMax(
              imageRequest.getLowestPermittedRequestLevel(), lowestPermittedRequestLevelOnSubmit);
      SettableProducerContext settableProducerContext =
          new SettableProducerContext(
              imageRequest,
              generateUniqueFutureId(),
              uiComponentId,
              requestListener2,
              callerContext,
              lowestPermittedRequestLevel,
              /* isPrefetch */ false,
              imageRequest.getProgressiveRenderingEnabled()
                  || !UriUtil.isNetworkUri(imageRequest.getSourceUri()),
              imageRequest.getPriority(),
              mConfig);
      return CloseableProducerToDataSourceAdapter.create(
          producerSequence, settableProducerContext, requestListener2);
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    } finally {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
    }
  }  
}

class CloseableProducerToDataSourceAdapter {
  public static <T> DataSource<CloseableReference<T>> create(
      Producer<CloseableReference<T>> producer,
      SettableProducerContext settableProducerContext,
      RequestListener2 listener) {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("CloseableProducerToDataSourceAdapter#create");
    }
    CloseableProducerToDataSourceAdapter<T> result =
        new CloseableProducerToDataSourceAdapter<T>(producer, settableProducerContext, listener);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
    return result;
  }    
}

class AbstractProducerToDataSourceAdapter {
  protected AbstractProducerToDataSourceAdapter(
      Producer<T> producer,
      SettableProducerContext settableProducerContext,
      RequestListener2 requestListener) {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("AbstractProducerToDataSourceAdapter()");
    }
    mSettableProducerContext = settableProducerContext;
    mRequestListener = requestListener;
    setInitialExtras();
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("AbstractProducerToDataSourceAdapter()->onRequestStart");
    }
    mRequestListener.onRequestStart(mSettableProducerContext);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("AbstractProducerToDataSourceAdapter()->produceResult");
    }
    producer.produceResults(createConsumer(), settableProducerContext);                     // 提交任务到任务队列的操作隐藏在这里
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
  }    
}

/**
 * Fresco 设计了生产者这么一个概念来组合/组装整个图形处理流水线（ImagePipeline），上一个生产者的输出可以作为下一个生产者的输入
 * Building block for image processing in the image pipeline
 */
public interface Producer<T> {
  /**
   * 此方法用以启动整个图像流水线，结果是通过 callback 模式（Consumer）回传的
   * Start producing results for given context. Provided consumer is notified whenever progress is
   * made (new value is ready or error occurs).
   */
  void produceResults(Consumer<T> consumer, ProducerContext context);
}

public interface Consumer<T> {
  void onNewResult(@Nullable T newResult, @Status int status);
  void onFailure(Throwable t);
  void onCancellation();
  void onProgressUpdate(float progress);
}

// 回到 ImagePipeline.fetchDecodedImage 看看整个流水线是怎么组装起来的
class ProducerSequenceFactory {
  public Producer<CloseableReference<CloseableImage>> getDecodedImageProducerSequence(
      ImageRequest imageRequest) {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("ProducerSequenceFactory#getDecodedImageProducerSequence");
    }
    Producer<CloseableReference<CloseableImage>> pipelineSequence =
        getBasicDecodedImageSequence(imageRequest);
    // 后处理器、预加载为 Bitmap 和延迟加载
    if (imageRequest.getPostprocessor() != null) {    
      pipelineSequence = getPostprocessorSequence(pipelineSequence);
    }

    if (mUseBitmapPrepareToDraw) {
      pipelineSequence = getBitmapPrepareSequence(pipelineSequence);
    }

    if (mAllowDelay && imageRequest.getDelayMs() > 0) {
      pipelineSequence = getDelaySequence(pipelineSequence);
    }

    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
    return pipelineSequence;
  }    
}

class ProducerSequenceFactory {
  private Producer<CloseableReference<CloseableImage>> getBasicDecodedImageSequence(
      ImageRequest imageRequest) {
    try {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.beginSection("ProducerSequenceFactory#getBasicDecodedImageSequence");
      }
      Preconditions.checkNotNull(imageRequest);

      Uri uri = imageRequest.getSourceUri();
      Preconditions.checkNotNull(uri, "Uri is null.");

      switch (imageRequest.getSourceUriType()) {
        case SOURCE_TYPE_NETWORK:    // 网络数据源
          return getNetworkFetchSequence();
        case SOURCE_TYPE_LOCAL_VIDEO_FILE:
          return getLocalVideoFileFetchSequence();
        case SOURCE_TYPE_LOCAL_IMAGE_FILE:
          return getLocalImageFileFetchSequence();
        case SOURCE_TYPE_LOCAL_CONTENT:
          if (imageRequest.getLoadThumbnailOnly()
              && Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            return getLocalContentUriThumbnailFetchSequence();
          } else if (MediaUtils.isVideo(mContentResolver.getType(uri))) {
            return getLocalVideoFileFetchSequence();
          }
          return getLocalContentUriFetchSequence();
        case SOURCE_TYPE_LOCAL_ASSET:
          return getLocalAssetFetchSequence();
        case SOURCE_TYPE_LOCAL_RESOURCE:
          return getLocalResourceFetchSequence();
        case SOURCE_TYPE_QUALIFIED_RESOURCE:
          return getQualifiedResourceFetchSequence();
        case SOURCE_TYPE_DATA:
          return getDataFetchSequence();
        default:
          throw new IllegalArgumentException(
              "Unsupported uri scheme! Uri is: " + getShortenedUriString(uri));
      }
    } finally {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
    }
  }  

  private synchronized Producer<CloseableReference<CloseableImage>> getNetworkFetchSequence() {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("ProducerSequenceFactory#getNetworkFetchSequence");
    }
    if (mNetworkFetchSequence == null) {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.beginSection("ProducerSequenceFactory#getNetworkFetchSequence:init");
      }
      mNetworkFetchSequence =
          newBitmapCacheGetToDecodeSequence(getCommonNetworkFetchToEncodedMemorySequence());
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
    }
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
    return mNetworkFetchSequence;
  }

  private synchronized Producer<EncodedImage> getCommonNetworkFetchToEncodedMemorySequence() {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection(
          "ProducerSequenceFactory#getCommonNetworkFetchToEncodedMemorySequence");
    }
    if (mCommonNetworkFetchToEncodedMemorySequence == null) {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.beginSection(
            "ProducerSequenceFactory#getCommonNetworkFetchToEncodedMemorySequence:init");
      }
      Producer<EncodedImage> inputProducer =
          Preconditions.checkNotNull(
              newEncodedCacheMultiplexToTranscodeSequence(                        // decode 和 cache
                  mProducerFactory.newNetworkFetchProducer(mNetworkFetcher)));    // 从网络获取 byte array
      mCommonNetworkFetchToEncodedMemorySequence =
          ProducerFactory.newAddImageTransformMetaDataProducer(inputProducer);

      mCommonNetworkFetchToEncodedMemorySequence =
          mProducerFactory.newResizeAndRotateProducer(                            // 缩放和选择
              mCommonNetworkFetchToEncodedMemorySequence,
              mResizeAndRotateEnabledForNetwork && !mDownsampleEnabled,
              mImageTranscoderFactory);
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
    }
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
    return mCommonNetworkFetchToEncodedMemorySequence;
  }      
}

class ProducerFactory {
  public Producer<EncodedImage> newNetworkFetchProducer(NetworkFetcher networkFetcher) {
    return new NetworkFetchProducer(mPooledByteBufferFactory, mByteArrayPool, networkFetcher);
  }    
}

// NetworkFetchProducer.produceResults 会将网络请求任务提交到任务队列
// 得到的数据（byte array）包装为 EncodedImage（并没有立刻 decode）交给下一个 Producer
class NetworkFetchProducer implements Producer<EncodedImage> {
  public void produceResults(Consumer<EncodedImage> consumer, ProducerContext context) {
    context.getProducerListener().onProducerStart(context, PRODUCER_NAME);
    final FetchState fetchState = mNetworkFetcher.createFetchState(consumer, context);
    mNetworkFetcher.fetch(
        fetchState,
        new NetworkFetcher.Callback() {
          @Override
          public void onResponse(InputStream response, int responseLength) throws IOException {
            if (FrescoSystrace.isTracing()) {
              FrescoSystrace.beginSection("NetworkFetcher->onResponse");
            }
            NetworkFetchProducer.this.onResponse(fetchState, response, responseLength);
            if (FrescoSystrace.isTracing()) {
              FrescoSystrace.endSection();
            }
          }

          @Override
          public void onFailure(Throwable throwable) {
            NetworkFetchProducer.this.onFailure(fetchState, throwable);
          }

          @Override
          public void onCancellation() {
            NetworkFetchProducer.this.onCancellation(fetchState);
          }
        });
  }

  protected void onResponse(
      FetchState fetchState, InputStream responseData, int responseContentLength)
      throws IOException {
    final PooledByteBufferOutputStream pooledOutputStream;
    if (responseContentLength > 0) {
      pooledOutputStream = mPooledByteBufferFactory.newOutputStream(responseContentLength);
    } else {
      pooledOutputStream = mPooledByteBufferFactory.newOutputStream();
    }
    final byte[] ioArray = mByteArrayPool.get(READ_SIZE);
    try {
      int length;
      while ((length = responseData.read(ioArray)) >= 0) {
        if (length > 0) {
          pooledOutputStream.write(ioArray, 0, length);
          maybeHandleIntermediateResult(pooledOutputStream, fetchState);
          float progress = calculateProgress(pooledOutputStream.size(), responseContentLength);
          fetchState.getConsumer().onProgressUpdate(progress);
        }
      }
      mNetworkFetcher.onFetchCompletion(fetchState, pooledOutputStream.size());
      handleFinalResult(pooledOutputStream, fetchState);
    } finally {
      mByteArrayPool.release(ioArray);
      pooledOutputStream.close();
    }
  }

  protected void handleFinalResult(
      PooledByteBufferOutputStream pooledOutputStream, FetchState fetchState) {
    Map<String, String> extraMap = getExtraMap(fetchState, pooledOutputStream.size());
    ProducerListener2 listener = fetchState.getListener();
    listener.onProducerFinishWithSuccess(fetchState.getContext(), PRODUCER_NAME, extraMap);
    listener.onUltimateProducerReached(fetchState.getContext(), PRODUCER_NAME, true);
    fetchState.getContext().putOriginExtra("network");
    notifyConsumer(
        pooledOutputStream,
        Consumer.IS_LAST | fetchState.getOnNewResultStatusFlags(),
        fetchState.getResponseBytesRange(),
        fetchState.getConsumer(),
        fetchState.getContext());
  }        
}
```

# 一些概念

Fresco 不能像 Glide 那样直接作用于 `ImageView`，必须用 `SimpleDraweeView` 替换掉布局里的 `ImageView`（感觉侵入性有点强呀？）

还好 Fresco 内部逻辑并没有写到 UI 控件 `SimpleDraweeView` 里，而是抽离放在 `DraweeHolder` 和 `DraweeController` 里

图像的加载和处理是一个流水线 `ImagePipeline`，`Producer / Consumer` 是流水线上的一个个节点，上一个节点的输出作为下一个节点的输出，各节点是一对一的，生产者和消费者是 callback 模式

`ImagePipeline` 的最终输出是一个 `DataSource`，类似于 `Future` 的概念，可以用 `DataSubscriber` 订阅它的各种事件：成功（或者是一个新的结果）、失败、取消和进度更新，数据源和订阅者是一对多的

# ImagePipeline

`ImagePipeline` 可以想象为一个栈（Stack），不断地往里边添加节点，第一个是头节点，最后一个是尾结点，先添加的节点在后添加节点的 `前面`，后添加节点在先添加节点的 `后面`

发一个请求时先触达尾结点然后流向头节点，这叫 `去程`，响应从头结点开始流向尾结点，这叫 `回程`

跟 OkHttp Interceptor Chain 和 Servlet Contianer Filter 一样，ImagePipeline 的一次工作包含 `去程` 和 `回程`，以 `[A, B, C, D]` 为例（`A` 是头节点）：

* 去程从 `Producer.produceResults` 开始，`D.produceResults` -> `C.produceResults` -> `B.produceResults` -> `A.produceResults`，`A` 也许是个 NetworkFetchProducer 执行网络请求，拿到 Response 后执行回程
* 回程从 `Consumer.onNewResult` 开始，`A.onNewResult` -> `B.onNewResult` -> `C.onNewResult` -> `D.onNewResult` -> `my.onNewResult`，当然可能会失败 `onFailure(throwable)`、取消 `onCancellation()` 等，一样是这个顺序

# NetworkFetchProducer

顾名思义，执行网络请求（HTTP）的 ImagePipeline 节点，，也是是整个流水线的第一个节点

它用接口 `NetworkFetcher` 抹平了底层各网络库的 API 差异，具体的实现有：

* `OkHttpNetworkFetcher`
* `VolleyNetworkFetcher`
* `HttpUrlConnectionNetworkFetcher`
* `PriorityNetworkFetcher`，它是一个装饰器（Decorator），具体的网络加载功能由上面的那些实现提供，它给请求提供了【优先队列】的特性，后续会说到

```java
/**
 *  Interface that specifies network fetcher used by the image pipeline.
 */
public interface NetworkFetcher<FETCH_STATE extends FetchState> {
  interface Callback {
    void onResponse(InputStream response, int responseLength) throws IOException;
    void onFailure(Throwable throwable);
    void onCancellation();
  }

  /**
   * Creates a new instance of the {@link FetchState}-derived object used to store state.
   */
  FETCH_STATE createFetchState(Consumer<EncodedImage> consumer, ProducerContext producerContext);
  /**
   * Gets a map containing extra parameters to pass to the listeners.
   */
  Map<String, String> getExtraMap(FETCH_STATE fetchState, int byteSize);  

  void fetch(FETCH_STATE fetchState, Callback callback);
  boolean shouldPropagate(FETCH_STATE fetchState);
  void onFetchCompletion(FETCH_STATE fetchState, int byteSize);
}
```

`NetworkFetchProducer` 从 Response 里读取 ByteArray 格式的内容，并包装为 `EncodedImage` 输出给下一节点；考虑到性能问题，在这一阶段 ByteArray 并没有 decode 为 `Bitmap`，而是通过解析前 N 个字节长度的数据来获取图片格式、尺寸等一些元信息

同时加载进度 `Consumer.onProgressUpdate` 也是由 `NetworkFetchProducer` 在 Response Reading 时发出

```java
class NetworkFetchProducer {
  public void produceResults(Consumer<EncodedImage> consumer, ProducerContext context) {    // 网络加载功能交由 NetworkFetcher 实现
    context.getProducerListener().onProducerStart(context, PRODUCER_NAME);
    final FetchState fetchState = mNetworkFetcher.createFetchState(consumer, context);
    mNetworkFetcher.fetch(
        fetchState,
        new NetworkFetcher.Callback() {
          @Override
          public void onResponse(InputStream response, int responseLength) throws IOException {
            if (FrescoSystrace.isTracing()) {
              FrescoSystrace.beginSection("NetworkFetcher->onResponse");
            }
            NetworkFetchProducer.this.onResponse(fetchState, response, responseLength);
            if (FrescoSystrace.isTracing()) {
              FrescoSystrace.endSection();
            }
          }

          @Override
          public void onFailure(Throwable throwable) {
            NetworkFetchProducer.this.onFailure(fetchState, throwable);
          }

          @Override
          public void onCancellation() {
            NetworkFetchProducer.this.onCancellation(fetchState);
          }
        });
  }

  protected void onResponse(
      FetchState fetchState, InputStream responseData, int responseContentLength)
      throws IOException {
    final PooledByteBufferOutputStream pooledOutputStream;
    if (responseContentLength > 0) {
      pooledOutputStream = mPooledByteBufferFactory.newOutputStream(responseContentLength);
    } else {
      pooledOutputStream = mPooledByteBufferFactory.newOutputStream();
    }
    final byte[] ioArray = mByteArrayPool.get(READ_SIZE);
    try {
      int length;
      while ((length = responseData.read(ioArray)) >= 0) {
        if (length > 0) {
          pooledOutputStream.write(ioArray, 0, length);
          maybeHandleIntermediateResult(pooledOutputStream, fetchState);
          float progress = calculateProgress(pooledOutputStream.size(), responseContentLength);
          fetchState.getConsumer().onProgressUpdate(progress);    // 更新加载进度
        }
      }
      mNetworkFetcher.onFetchCompletion(fetchState, pooledOutputStream.size());
      handleFinalResult(pooledOutputStream, fetchState);
    } finally {
      mByteArrayPool.release(ioArray);
      pooledOutputStream.close();
    }
  }    

  protected void handleFinalResult(
      PooledByteBufferOutputStream pooledOutputStream, FetchState fetchState) {
    Map<String, String> extraMap = getExtraMap(fetchState, pooledOutputStream.size());
    ProducerListener2 listener = fetchState.getListener();
    listener.onProducerFinishWithSuccess(fetchState.getContext(), PRODUCER_NAME, extraMap);
    listener.onUltimateProducerReached(fetchState.getContext(), PRODUCER_NAME, true);
    fetchState.getContext().putOriginExtra("network");
    notifyConsumer(
        pooledOutputStream,
        Consumer.IS_LAST | fetchState.getOnNewResultStatusFlags(),
        fetchState.getResponseBytesRange(),
        fetchState.getConsumer(),
        fetchState.getContext());
  }

  protected static void notifyConsumer(
      PooledByteBufferOutputStream pooledOutputStream,
      @Consumer.Status int status,
      @Nullable BytesRange responseBytesRange,
      Consumer<EncodedImage> consumer,
      ProducerContext context) {    // 将 ByteArray 包装为 EncodedImage 输出给消费者
    CloseableReference<PooledByteBuffer> result =
        CloseableReference.of(pooledOutputStream.toByteBuffer());
    EncodedImage encodedImage = null;
    try {
      encodedImage = new EncodedImage(result);
      encodedImage.setBytesRange(responseBytesRange);
      encodedImage.parseMetaData();
      context.setEncodedImageOrigin(EncodedImageOrigin.NETWORK);
      consumer.onNewResult(encodedImage, status);
    } finally {
      EncodedImage.closeSafely(encodedImage);
      CloseableReference.closeSafely(result);
    }
  }      
}

// 这里简单地看下 EncodedImage 是如何判断图片格式的，各种图片的格式和 SOI 可以深入代码细节，结合网上资料了解
class EncodedImage {
  public void parseMetaData() {
    if (!sUseCachedMetadata) {
      internalParseMetaData();
      return;
    }

    if (mHasParsedMetadata) {
      return;
    }
    internalParseMetaData();
    mHasParsedMetadata = true;
  }


  /** Sets the encoded image meta data. */
  private void internalParseMetaData() {
    final ImageFormat imageFormat =
        ImageFormatChecker.getImageFormat_WrapIOException(getInputStream());    // 解析图片格式
    mImageFormat = imageFormat;
    // BitmapUtil.decodeDimensions has a bug where it will return 100x100 for some WebPs even though
    // those are not its actual dimensions
    final Pair<Integer, Integer> dimensions;    // 解析尺寸
    if (DefaultImageFormats.isWebpFormat(imageFormat)) {
      dimensions = readWebPImageSize();
    } else {
      dimensions = readImageMetaData().getDimensions();
    }
    if (imageFormat == DefaultImageFormats.JPEG && mRotationAngle == UNKNOWN_ROTATION_ANGLE) {
      // Load the JPEG rotation angle only if we have the dimensions
      if (dimensions != null) {
        mExifOrientation = JfifUtil.getOrientation(getInputStream());
        mRotationAngle = JfifUtil.getAutoRotateAngleFromOrientation(mExifOrientation);
      }
    } else if (imageFormat == DefaultImageFormats.HEIF
        && mRotationAngle == UNKNOWN_ROTATION_ANGLE) {
      mExifOrientation = HeifExifUtil.getOrientation(getInputStream());
      mRotationAngle = JfifUtil.getAutoRotateAngleFromOrientation(mExifOrientation);
    } else if (mRotationAngle == UNKNOWN_ROTATION_ANGLE) {
      mRotationAngle = 0;
    }
  }      
}

class ImageFormatChecker {
  public static ImageFormat getImageFormat_WrapIOException(final InputStream is) {
    try {
      return getImageFormat(is);
    } catch (IOException ioe) {
      throw Throwables.propagate(ioe);
    }
  }


  /**
   * Tries to read up to MAX_HEADER_LENGTH bytes from InputStream is and use read bytes to determine
   * type of the image contained in is. 
   */
  public static ImageFormat getImageFormat(final InputStream is) throws IOException {
    return getInstance().determineImageFormat(is);
  }      

  public ImageFormat determineImageFormat(final InputStream is) throws IOException {
    Preconditions.checkNotNull(is);
    final byte[] imageHeaderBytes = new byte[mMaxHeaderLength];
    final int headerSize = readHeaderFromStream(mMaxHeaderLength, is, imageHeaderBytes);

    ImageFormat format = mDefaultFormatChecker.determineFormat(imageHeaderBytes, headerSize);
    if (format != null && format != ImageFormat.UNKNOWN) {
      return format;
    }

    if (mCustomImageFormatCheckers != null) {
      for (ImageFormat.FormatChecker formatChecker : mCustomImageFormatCheckers) {
        format = formatChecker.determineFormat(imageHeaderBytes, headerSize);
        if (format != null && format != ImageFormat.UNKNOWN) {
          return format;
        }
      }
    }
    return ImageFormat.UNKNOWN;
  }  
}

class DefaultImageFormatChecker {
  public final ImageFormat determineFormat(byte[] headerBytes, int headerSize) {
    Preconditions.checkNotNull(headerBytes);

    if (!mUseNewOrder && WebpSupportStatus.isWebpHeader(headerBytes, 0, headerSize)) {
      return getWebpFormat(headerBytes, headerSize);
    }

    if (isJpegHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.JPEG;
    }

    if (isPngHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.PNG;
    }

    if (mUseNewOrder && WebpSupportStatus.isWebpHeader(headerBytes, 0, headerSize)) {
      return getWebpFormat(headerBytes, headerSize);
    }

    if (isGifHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.GIF;
    }

    if (isBmpHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.BMP;
    }

    if (isIcoHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.ICO;
    }

    if (isHeifHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.HEIF;
    }

    if (isDngHeader(headerBytes, headerSize)) {
      return DefaultImageFormats.DNG;
    }

    return ImageFormat.UNKNOWN;
  }


  /**
   * Every JPEG image should start with SOI mark (0xFF, 0xD8) followed by beginning of another
   * segment (0xFF)
   */
  private static final byte[] JPEG_HEADER = new byte[] {(byte) 0xFF, (byte) 0xD8, (byte) 0xFF};

  /**
   * Checks if imageHeaderBytes starts with SOI (start of image) marker, followed by 0xFF. If
   * headerSize is lower than 3 false is returned. Description of jpeg format can be found here: <a
   * href="http://www.w3.org/Graphics/JPEG/itu-t81.pdf">
   * http://www.w3.org/Graphics/JPEG/itu-t81.pdf</a> Annex B deals with compressed data format
   */
  private static boolean isJpegHeader(final byte[] imageHeaderBytes, final int headerSize) {
    return headerSize >= JPEG_HEADER.length
        && ImageFormatCheckerUtils.startsWithPattern(imageHeaderBytes, JPEG_HEADER);
  }      
}
```

# 更多的资源加载器

通过资源的 URI Scheme 来选择合适的加载器，`NetworkFetchProducer` 应该是最常用加载器之一了，它从网络获取资源，此外还有 `LocalResourceFetchProducer`、`LocalFileFetchProducer` 等

| uri | producer | core api |
|-----|----------|----------|
| https://abc.com/avatar.jpg                 | NetworkFetchProducer           | OkHttp & Volley & HttpUrlConnection |
| res://com.example/75281679                 | LocalResourceFetchProducer     | `Resources.openRawResource(resId)`，authority 部分是忽略掉的，直接取 path 作为资源 ID |
| asset://com.example/avatar/default.jpg     | LocalAssetFetchProducer        | `AssetManager.open`，authority 部分是忽略掉的，直接取 path 作为 asset path |
| file:///sdcard/DCIM/Camera/avatar.jpg      | LocalFileFetchProducer         | File API |
| content://media/external/images/media/2283 | LocalContentUriFetchProducer & LocalThumbnailBitmapProducer | ContentResolver.openInputStream/loadThumbnail |
| data://base64                              | DataFetchProducer              | Base64.decode |
| android.resource://                        | QualifiedResourceFetchProducer | ContentResolver.openInputStream |

```java
class ImageRequest {
  private static @SourceUriType int getSourceUriType(final Uri uri) {
    if (uri == null) {
      return SOURCE_TYPE_UNKNOWN;
    }
    if (UriUtil.isNetworkUri(uri)) {                                      // http:// & https://
      return SOURCE_TYPE_NETWORK;
    } else if (UriUtil.isLocalFileUri(uri)) {                             // file://
      if (MediaUtils.isVideo(MediaUtils.extractMime(uri.getPath()))) {    // 通过文件后缀名判断是 image 还是 video
        return SOURCE_TYPE_LOCAL_VIDEO_FILE;
      } else {
        return SOURCE_TYPE_LOCAL_IMAGE_FILE;                              
      }
    } else if (UriUtil.isLocalContentUri(uri)) {                          // content://
      return SOURCE_TYPE_LOCAL_CONTENT;
    } else if (UriUtil.isLocalAssetUri(uri)) {                            // asset://
      return SOURCE_TYPE_LOCAL_ASSET;
    } else if (UriUtil.isLocalResourceUri(uri)) {                         // res://
      return SOURCE_TYPE_LOCAL_RESOURCE;
    } else if (UriUtil.isDataUri(uri)) {                                  // data://
      return SOURCE_TYPE_DATA;
    } else if (UriUtil.isQualifiedResourceUri(uri)) {                     // android.resource://
      return SOURCE_TYPE_QUALIFIED_RESOURCE;
    } else {
      return SOURCE_TYPE_UNKNOWN;
    }
  }    
}
```

# 磁盘缓存

磁盘缓存这一 feature 由 `ImagePipelineConfig.Builder.setDiskCacheEnabled` 开启，相关的 `ImagePipeline` 节点是紧邻着 `NetworkFetchProducer` 等头节点之后被添加的，包括三个个节点（按添加顺序）：

1. PartialDiskCacheProducer
2. DiskCacheWriteProducer
3. DiskCacheReadProducer

添加 `ImagePipeline` 节点的方法栈如下：

```java
SimpleDraweeView.setImageURI(uri)
SimpleDraweeView.setImageURI(uri, null)
AbstractDraweeControllerBuilder.build
AbstractDraweeController.buildController
PipelineDraweeControllerBuilder.obtainController
AbstractDraweeControllerBuilder.obtainDataSourceSupplier
AbstractDraweeControllerBuilder.getDataSourceSupplierForRequest(controller, controllerId, imageRequest)
AbstractDraweeControllerBuilder.getDataSourceSupplierForRequest(controller, controllerId, imageRequest, CacheLevel.FULL_FETCH)
PipelineDraweeControllerBuilder.getDataSourceForRequest
ImagePipeline.fetchDecodedImage
ProducerSequenceFactory.getDecodedImageProducerSequence
ProducerSequenceFactory.getBasicDecodedImageSequence
ProducerSequenceFactory.getNetworkFetchSequence
ProducerSequenceFactory.getCommonNetworkFetchToEncodedMemorySequence
ProducerSequenceFactory.newEncodedCacheMultiplexToTranscodeSequence

class ProducerSequenceFactory {
  private Producer<EncodedImage> newDiskCacheSequence(Producer<EncodedImage> inputProducer) {
    Producer<EncodedImage> cacheWriteProducer;
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("ProducerSequenceFactory#newDiskCacheSequence");
    }
    if (mPartialImageCachingEnabled) {
      Producer<EncodedImage> partialDiskCacheProducer =
          mProducerFactory.newPartialDiskCacheProducer(inputProducer);
      cacheWriteProducer = mProducerFactory.newDiskCacheWriteProducer(partialDiskCacheProducer);
    } else {
      cacheWriteProducer = mProducerFactory.newDiskCacheWriteProducer(inputProducer);
    }
    DiskCacheReadProducer result = mProducerFactory.newDiskCacheReadProducer(cacheWriteProducer);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
    return result;
  }    
}
```

## PartialDiskCacheProducer

由 `ImagePipelineConfig.Builder.experiment().setPartialImageCachingEnabled` 开启

继续前先了解下 [HTTP 分段下载](../../../../2021/12/16/http-range/) 的概念，可以看到分段下载是可以分多段（按需分段）的，但 `PartialDiskCacheProducer` 不太一样它只能分两段，看看它代码的逻辑：

1. 发起 ImagePipeline 请求时（去程）
    1. 先从 disk cache 里寻找 partial cache
    2. 如果请求的是 partial content 并且 partial cache 包含这一 range，就不需要执行网络请求了，直接返回缓存
    3. 否则 partial cache 作为 `IS_PARTIAL_RESULT` 返回，且执行网络请求获取剩下的内容
    4. 没有 partial cache 那执行原请求
2. 收到响应时（回程）
    1. response header 里包含 `Content-Range`，说明是 partial content，设置 `IS_PARTIAL_RESULT` 标志
    2. 存在 partial cache 且是 partial response，合并 cache 和 response 后传递给下一节点，删除 partial cache
    3. 没有 partial cache 且是 partial response，将这部分内容缓存为 partical cache 以便后续合并

那么业务逻辑就是这样的：

1. 请求一个完整的资源服务器返回部分资源（206，Content-Range），或者请求资源的部分内容（Range 头部），总之就是服务器返回 partial content
2. 发现 `IS_PARTIAL_RESULT` 标志，将 response 缓存起来
3. 下次请求此资源时（Range 头部不是必须的），先返回缓存的部分内容，再请求剩下未获取的内容
4. 收到响应后（必须包含 `Content-Range`，否则不认为是 partial content，没有 `IS_PARTIAL_RESULT` 标志），合并 partial content 和 partial cache 为完整的资源返回，删除 partial cache

问题就来了，什么场景下会只请求部分内容（Range 头部）、请求一个完整的资源服务器却只返回部分资源（206，Content-Range）？不是很能理解它的使用场景

```java
class PartialDiskCacheProducer {
  public void produceResults(
      final Consumer<EncodedImage> consumer, final ProducerContext producerContext) {
    final ImageRequest imageRequest = producerContext.getImageRequest();
    final boolean isDiskCacheEnabledForRead =
        producerContext
            .getImageRequest()
            .isCacheEnabled(ImageRequest.CachesLocationsMasks.DISK_READ);

    final ProducerListener2 listener = producerContext.getProducerListener();
    listener.onProducerStart(producerContext, PRODUCER_NAME);

    final Uri uriForPartialCacheKey = createUriForPartialCacheKey(imageRequest);
    final CacheKey partialImageCacheKey =
        mCacheKeyFactory.getEncodedCacheKey(
            imageRequest, uriForPartialCacheKey, producerContext.getCallerContext());

    if (!isDiskCacheEnabledForRead) {
      listener.onProducerFinishWithSuccess(
          producerContext, PRODUCER_NAME, getExtraMap(listener, producerContext, false, 0));
      startInputProducer(consumer, producerContext, partialImageCacheKey, null);
      return;
    }

    // 发起请求时，先从 disk cache 里找 partial cache
    final AtomicBoolean isCancelled = new AtomicBoolean(false);
    final Task<EncodedImage> diskLookupTask =
        mDefaultBufferedDiskCache.get(partialImageCacheKey, isCancelled);    
    final Continuation<EncodedImage, Void> continuation =
        onFinishDiskReads(consumer, producerContext, partialImageCacheKey);

    diskLookupTask.continueWith(continuation);
    subscribeTaskForRequestCancellation(isCancelled, producerContext);
  }

  private Continuation<EncodedImage, Void> onFinishDiskReads(
      final Consumer<EncodedImage> consumer,
      final ProducerContext producerContext,
      final CacheKey partialImageCacheKey) {
    final ProducerListener2 listener = producerContext.getProducerListener();
    return new Continuation<EncodedImage, Void>() {
      @Override
      public Void then(Task<EncodedImage> task) throws Exception {
        if (isTaskCancelled(task)) {
          listener.onProducerFinishWithCancellation(producerContext, PRODUCER_NAME, null);
          consumer.onCancellation();
        } else if (task.isFaulted()) {
          listener.onProducerFinishWithFailure(
              producerContext, PRODUCER_NAME, task.getError(), null);
          startInputProducer(consumer, producerContext, partialImageCacheKey, null);
        } else {
          EncodedImage cachedReference = task.getResult();
          if (cachedReference != null) {
            listener.onProducerFinishWithSuccess(
                producerContext,
                PRODUCER_NAME,
                getExtraMap(listener, producerContext, true, cachedReference.getSize()));
            final BytesRange cachedRange = BytesRange.toMax(cachedReference.getSize() - 1);
            cachedReference.setBytesRange(cachedRange);

            // Create a new ImageRequest for the remaining data
            final int cachedLength = cachedReference.getSize();
            final ImageRequest originalRequest = producerContext.getImageRequest();

            // 如果请求的是 partial content 并且 partial cache 包含这一 range，就不需要执行网络请求了，直接返回缓存
            if (cachedRange.contains(originalRequest.getBytesRange())) {
              producerContext.putOriginExtra("disk", "partial");
              listener.onUltimateProducerReached(producerContext, PRODUCER_NAME, true);
              consumer.onNewResult(cachedReference, Consumer.IS_LAST | Consumer.IS_PARTIAL_RESULT);
            } else {

              // 否则 partial cache 作为 IS_PARTIAL_RESULT 返回，且执行网络请求获取剩下的内容  
              consumer.onNewResult(cachedReference, Consumer.IS_PARTIAL_RESULT);

              // Pass the request on, but only for the remaining bytes
              final ImageRequest remainingRequest =
                  ImageRequestBuilder.fromRequest(originalRequest)
                      .setBytesRange(BytesRange.from(cachedLength - 1))
                      .build();
              final SettableProducerContext contextForRemainingRequest =
                  new SettableProducerContext(remainingRequest, producerContext);

              startInputProducer(
                  consumer, contextForRemainingRequest, partialImageCacheKey, cachedReference);
            }
          } else {

            // 没有 partial cache 那执行原请求  
            listener.onProducerFinishWithSuccess(
                producerContext, PRODUCER_NAME, getExtraMap(listener, producerContext, false, 0));
            startInputProducer(consumer, producerContext, partialImageCacheKey, cachedReference);
          }
        }
        return null;
      }
    };
  }      
}

class OkHttpNetworkFetcher {
  protected void fetchWithRequest(
      final OkHttpNetworkFetchState fetchState,
      final NetworkFetcher.Callback callback,
      final Request request) {
    final Call call = mCallFactory.newCall(request);

    fetchState
        .getContext()
        .addCallbacks(
            new BaseProducerContextCallbacks() {
              @Override
              public void onCancellationRequested() {
                if (Looper.myLooper() != Looper.getMainLooper()) {
                  call.cancel();
                } else {
                  mCancellationExecutor.execute(
                      new Runnable() {
                        @Override
                        public void run() {
                          call.cancel();
                        }
                      });
                }
              }
            });

    call.enqueue(
        new okhttp3.Callback() {
          @Override
          public void onResponse(Call call, Response response) throws IOException {
            fetchState.responseTime = SystemClock.elapsedRealtime();
            final ResponseBody body = response.body();
            if (body == null) {
              handleException(call, new IOException("Response body null: " + response), callback);
              return;
            }
            try {
              if (!response.isSuccessful()) {
                handleException(
                    call, new IOException("Unexpected HTTP code " + response), callback);
                return;
              }

              // response header 里包含 Content-Range，说明是 partial content，设置 IS_PARTIAL_RESULT 标志
              BytesRange responseRange =
                  BytesRange.fromContentRangeHeader(response.header("Content-Range"));
              if (responseRange != null
                  && !(responseRange.from == 0
                      && responseRange.to == BytesRange.TO_END_OF_CONTENT)) {
                // Only treat as a partial image if the range is not all of the content
                fetchState.setResponseBytesRange(responseRange);
                fetchState.setOnNewResultStatusFlags(Consumer.IS_PARTIAL_RESULT);
              }

              long contentLength = body.contentLength();
              if (contentLength < 0) {
                contentLength = 0;
              }
              callback.onResponse(body.byteStream(), (int) contentLength);
            } catch (Exception e) {
              handleException(call, e, callback);
            } finally {
              body.close();
            }
          }

          @Override
          public void onFailure(Call call, IOException e) {
            handleException(call, e, callback);
          }
        });
  }    
}

class PartialDiskCacheConsumer {
    public void onNewResultImpl(@Nullable EncodedImage newResult, @Status int status) {
      if (isNotLast(status)) {
        // TODO 19247361 Consider merging of non-final results
        return;
      }

      // 存在 partial cache 且是 partial response，合并 cache 和 response 后传递给下一节点，删除 partial cache
      if (mPartialEncodedImageFromCache != null
          && newResult != null
          && newResult.getBytesRange() != null) {
        try {
          final PooledByteBufferOutputStream pooledOutputStream =
              merge(mPartialEncodedImageFromCache, newResult);
          sendFinalResultToConsumer(pooledOutputStream);
        } catch (IOException e) {
          // TODO 19247425 Delete cached file and request full image
          FLog.e(PRODUCER_NAME, "Error while merging image data", e);
          getConsumer().onFailure(e);
        } finally {
          newResult.close();
          mPartialEncodedImageFromCache.close();
        }
        mDefaultBufferedDiskCache.remove(mPartialImageCacheKey);

        // 没有 partial cache 且是 partial response，将这部分内容缓存为 partical cache 以便后续合并
      } else if (mIsDiskCacheEnabledForWrite
          && statusHasFlag(status, IS_PARTIAL_RESULT)
          && isLast(status)
          && newResult != null
          && newResult.getImageFormat() != ImageFormat.UNKNOWN) {
        mDefaultBufferedDiskCache.put(mPartialImageCacheKey, newResult);
        getConsumer().onNewResult(newResult, status);
      } else {
        getConsumer().onNewResult(newResult, status);
      }
    }    
}
```

## read && write

包含两个节点 `DiskCacheWriteProducer` 和 `DiskCacheReadProducer`，它们按顺序被添加到 ImagePipeline，那么：

* 去程：`DiskCacheReadProducer.produceResults` -> `DiskCacheWriteProducer.produceResults`
* 回程：`DiskCacheWriteProducer.onNewResult` -> `DiskCacheReadProducer.onNewResult`

发起加载请求是，disk reader 从磁盘缓存里找是否有对应的缓存文件，有的话就无须做网络请求，加载缓存文件至内存后返回；因为是先添加 disk reader 后添加 disk writer 的，请求被 disk reader 截胡后就不经过 disk writer 和 network fetcher 了

如果没能找到磁盘缓存，请求流转到 network fetcher，拿到 response 后 disk writer 将其保存为磁盘缓存

```java
class DiskCacheReadProducer {
  public void produceResults(
      final Consumer<EncodedImage> consumer, final ProducerContext producerContext) {
    final ImageRequest imageRequest = producerContext.getImageRequest();
    final boolean isDiskCacheEnabledForRead =
        producerContext
            .getImageRequest()
            .isCacheEnabled(ImageRequest.CachesLocationsMasks.DISK_READ);
    if (!isDiskCacheEnabledForRead) {
      maybeStartInputProducer(consumer, producerContext);
      return;
    }

    producerContext.getProducerListener().onProducerStart(producerContext, PRODUCER_NAME);

    final CacheKey cacheKey =
        mCacheKeyFactory.getEncodedCacheKey(imageRequest, producerContext.getCallerContext());
    final boolean isSmallRequest = (imageRequest.getCacheChoice() == CacheChoice.SMALL);
    final BufferedDiskCache preferredCache =
        isSmallRequest ? mSmallImageBufferedDiskCache : mDefaultBufferedDiskCache;
    final AtomicBoolean isCancelled = new AtomicBoolean(false);
    final Task<EncodedImage> diskLookupTask = preferredCache.get(cacheKey, isCancelled);    // 去程，从磁盘缓存里查找
    final Continuation<EncodedImage, Void> continuation =
        onFinishDiskReads(consumer, producerContext);
    diskLookupTask.continueWith(continuation);
    subscribeTaskForRequestCancellation(isCancelled, producerContext);
  }

  private Continuation<EncodedImage, Void> onFinishDiskReads(
      final Consumer<EncodedImage> consumer, final ProducerContext producerContext) {
    final ProducerListener2 listener = producerContext.getProducerListener();
    return new Continuation<EncodedImage, Void>() {
      @Override
      public Void then(Task<EncodedImage> task) throws Exception {
        if (isTaskCancelled(task)) {
          listener.onProducerFinishWithCancellation(producerContext, PRODUCER_NAME, null);
          consumer.onCancellation();
        } else if (task.isFaulted()) {
          listener.onProducerFinishWithFailure(
              producerContext, PRODUCER_NAME, task.getError(), null);
          mInputProducer.produceResults(consumer, producerContext);
        } else {
          EncodedImage cachedReference = task.getResult();
          if (cachedReference != null) {    // 找到磁盘缓存直接将去程截了，不继续流向下一节点（网络请求）
            listener.onProducerFinishWithSuccess(
                producerContext,
                PRODUCER_NAME,
                getExtraMap(listener, producerContext, true, cachedReference.getSize()));
            listener.onUltimateProducerReached(producerContext, PRODUCER_NAME, true);
            producerContext.putOriginExtra("disk");
            consumer.onProgressUpdate(1);
            consumer.onNewResult(cachedReference, Consumer.IS_LAST);
            cachedReference.close();
          } else {                         // 否则将请求传递给下一节点
            listener.onProducerFinishWithSuccess(
                producerContext, PRODUCER_NAME, getExtraMap(listener, producerContext, false, 0));
            mInputProducer.produceResults(consumer, producerContext);
          }
        }
        return null;
      }
    };
  }      
}

class DiskCacheWriteConsumer {
    public void onNewResultImpl(@Nullable EncodedImage newResult, @Status int status) {
      mProducerContext.getProducerListener().onProducerStart(mProducerContext, PRODUCER_NAME);
      // intermediate, null or uncacheable results are not cached, so we just forward them
      // as well as the images with unknown format which could be html response from the server
      if (isNotLast(status)
          || newResult == null
          || statusHasAnyFlag(status, DO_NOT_CACHE_ENCODED | IS_PARTIAL_RESULT)
          || newResult.getImageFormat() == ImageFormat.UNKNOWN) {
        mProducerContext
            .getProducerListener()
            .onProducerFinishWithSuccess(mProducerContext, PRODUCER_NAME, null);
        getConsumer().onNewResult(newResult, status);
        return;
      }

      final ImageRequest imageRequest = mProducerContext.getImageRequest();
      final CacheKey cacheKey =
          mCacheKeyFactory.getEncodedCacheKey(imageRequest, mProducerContext.getCallerContext());

      if (imageRequest.getCacheChoice() == ImageRequest.CacheChoice.SMALL) {    // 将 response 缓存起来
        mSmallImageBufferedDiskCache.put(cacheKey, newResult);
      } else {
        mDefaultBufferedDiskCache.put(cacheKey, newResult);
      }
      mProducerContext
          .getProducerListener()
          .onProducerFinishWithSuccess(mProducerContext, PRODUCER_NAME, null);

      getConsumer().onNewResult(newResult, status);
    }    
}
```

# 内存缓存（encoded）

添加 `EncodedMemoryCacheProducer` 节点以实现内存缓存，这里缓存的对象是 `encoded`，也就是被图像压缩算法编码后的、各种图像格式的原始数据，尚未被 `decode` 为 `Bitmap`

* `去程` 时从内存缓存里找，命中直接返回内存数据，未命中则交由下个节点处理
* `回程` 时将原始数据添加到内存缓存中，然后继续往下传递原始数据

默认开启，由 `ImageRequestBuilder.disableMemoryCache` 关闭 

```java
class EncodedMemoryCacheProducer {
  public void produceResults(
      final Consumer<EncodedImage> consumer, final ProducerContext producerContext) {
    try {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.beginSection("EncodedMemoryCacheProducer#produceResults");
      }
      final ProducerListener2 listener = producerContext.getProducerListener();
      listener.onProducerStart(producerContext, PRODUCER_NAME);
      final ImageRequest imageRequest = producerContext.getImageRequest();
      final CacheKey cacheKey =
          mCacheKeyFactory.getEncodedCacheKey(imageRequest, producerContext.getCallerContext());
      final boolean isEncodedCacheEnabledForRead =
          producerContext
              .getImageRequest()
              .isCacheEnabled(ImageRequest.CachesLocationsMasks.ENCODED_READ);
      CloseableReference<PooledByteBuffer> cachedReference =
          isEncodedCacheEnabledForRead ? mMemoryCache.get(cacheKey) : null;
      try {
        if (cachedReference != null) {    // 命中 encoded 内存缓存，将请求截胡，直接返回内存中的数据
          EncodedImage cachedEncodedImage = new EncodedImage(cachedReference);
          try {
            listener.onProducerFinishWithSuccess(
                producerContext,
                PRODUCER_NAME,
                listener.requiresExtraMap(producerContext, PRODUCER_NAME)
                    ? ImmutableMap.of(EXTRA_CACHED_VALUE_FOUND, "true")
                    : null);
            listener.onUltimateProducerReached(producerContext, PRODUCER_NAME, true);
            producerContext.putOriginExtra("memory_encoded");
            consumer.onProgressUpdate(1f);
            consumer.onNewResult(cachedEncodedImage, Consumer.IS_LAST);
            return;
          } finally {
            EncodedImage.closeSafely(cachedEncodedImage);
          }
        }

        if (producerContext.getLowestPermittedRequestLevel().getValue()
            >= ImageRequest.RequestLevel.ENCODED_MEMORY_CACHE.getValue()) {
          listener.onProducerFinishWithSuccess(
              producerContext,
              PRODUCER_NAME,
              listener.requiresExtraMap(producerContext, PRODUCER_NAME)
                  ? ImmutableMap.of(EXTRA_CACHED_VALUE_FOUND, "false")
                  : null);
          listener.onUltimateProducerReached(producerContext, PRODUCER_NAME, false);
          producerContext.putOriginExtra("memory_encoded", "nil-result");
          consumer.onNewResult(null, Consumer.IS_LAST);
          return;
        }

        Consumer consumerOfInputProducer =
            new EncodedMemoryCacheConsumer(
                consumer,
                mMemoryCache,
                cacheKey,
                producerContext
                    .getImageRequest()
                    .isCacheEnabled(ImageRequest.CachesLocationsMasks.ENCODED_WRITE),
                producerContext.getImagePipelineConfig().getExperiments().isEncodedCacheEnabled());

        listener.onProducerFinishWithSuccess(
            producerContext,
            PRODUCER_NAME,
            listener.requiresExtraMap(producerContext, PRODUCER_NAME)
                ? ImmutableMap.of(EXTRA_CACHED_VALUE_FOUND, "false")
                : null);
        mInputProducer.produceResults(consumerOfInputProducer, producerContext);
      } finally {
        CloseableReference.closeSafely(cachedReference);
      }
    } finally {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
    }
  }    
}

class EncodedMemoryCacheConsumer {
  public void onNewResultImpl(@Nullable EncodedImage newResult, @Status int status) {
    try {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.beginSection("EncodedMemoryCacheProducer#onNewResultImpl");
      }
      // intermediate, null or uncacheable results are not cached, so we just forward them
      // as well as the images with unknown format which could be html response from the server
      if (isNotLast(status)
          || newResult == null
          || statusHasAnyFlag(status, DO_NOT_CACHE_ENCODED | IS_PARTIAL_RESULT)
          || newResult.getImageFormat() == ImageFormat.UNKNOWN) {
        getConsumer().onNewResult(newResult, status);
        return;
      }
      // cache and forward the last result
      CloseableReference<PooledByteBuffer> ref = newResult.getByteBufferRef();
      if (ref != null) {
        CloseableReference<PooledByteBuffer> cachedResult = null;
        try {
          if (mEncodedCacheEnabled && mIsEncodedCacheEnabledForWrite) {    // 将原始数据缓存至内存
            cachedResult = mMemoryCache.cache(mRequestedCacheKey, ref);
          }
        } finally {
          CloseableReference.closeSafely(ref);
        }
        if (cachedResult != null) {
          EncodedImage cachedEncodedImage;
          try {
            cachedEncodedImage = new EncodedImage(cachedResult);
            cachedEncodedImage.copyMetaDataFrom(newResult);
          } finally {
            CloseableReference.closeSafely(cachedResult);
          }
          try {
            getConsumer().onProgressUpdate(1f);
            getConsumer().onNewResult(cachedEncodedImage, status);
            return;
          } finally {
            EncodedImage.closeSafely(cachedEncodedImage);
          }
        }
      }
      getConsumer().onNewResult(newResult, status);
    } finally {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
    }
  }    
}
```

# 其他 ImagePipeline 节点

## MultiplexProducer

`multiplex`：多路复用，`multiplexer`：多路复用器，顾名思义这个节点的作用是合并相同的 `ImageRequest` 以节约网络和 IO 资源，因为同一时刻相同的请求只需要执行一次，多个 consumer 可以等待和接收同一个 response，看看是如何实现的：

1. 多路复用器有两个：`EncodedCacheKeyMultiplexProducer`（排在 encoded memory cache 后面） 和 `BitmapMemoryCacheKeyMultiplexProducer`（排在 bitmap memory cache 后面）
2. 一个请求对应一个 `Multiplexer`，相同请求（`MultiplexProducer.getKey`）的 consumer 都挂在同一个 `Multiplexer` 里
3. 第一个请求才会通过此节点，流向下一节点（执行真正的请求操作：网络、缓存），后续的相同的请求都被终止，它们的 consumer 被挂在同一 `Multiplexer` 里
4. 第一个（也是唯一一个）请求的 response 将被分发给各个 consumer
5. 涉及到多线程，创建 Multiplexer 和添加 consumer 的操作需要被 `synchronized` 保护

```java
class MultiplexProducer {
  public void produceResults(Consumer<T> consumer, ProducerContext context) {
    try {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.beginSection("MultiplexProducer#produceResults");
      }

      context.getProducerListener().onProducerStart(context, mProducerName);

      K key = getKey(context);
      Multiplexer multiplexer;
      boolean createdNewMultiplexer;
      // We do want to limit scope of this lock to guard only accesses to mMultiplexers map.
      // However what we would like to do here is to atomically lookup mMultiplexers, add new
      // consumer to consumers set associated with the map's entry and call consumer's callback with
      // last intermediate result. We should not do all of those things under this lock.
      do {
        createdNewMultiplexer = false;
        synchronized (this) {
          multiplexer = getExistingMultiplexer(key);
          if (multiplexer == null) {
            multiplexer = createAndPutNewMultiplexer(key);
            createdNewMultiplexer = true;
          }
        }
        // addNewConsumer may call consumer's onNewResult method immediately. For this reason
        // we release "this" lock. If multiplexer is removed from mMultiplexers in the meantime,
        // which is not very probable, then addNewConsumer will fail and we will be able to retry.
      } while (!multiplexer.addNewConsumer(consumer, context));

      if (createdNewMultiplexer) {
        multiplexer.startInputProducerIfHasAttachedConsumers(
            TriState.valueOf(context.isPrefetch()));
      }
    } finally {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
    }
  }    
}
```