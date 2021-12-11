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

## NetworkFetchProducer

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

### 内存池

`NetworkFetchProducer` 要经常进行 IO 操作，如果使用局部 ByteArray 会导致频繁申请/回收内存，引起内存波动和内存碎片

Fresco 使用内存池（`Pool`）或者说池化（`Pooled`）来解决这一问题：

* 小块的内存用 `GenericByteArrayPool`，比如下面从 Response 里循环读数据用的临时内存块 `ioArray` 就是从 `mByteArrayPool` 里拿的
* 大块的、不确定长度的内存用 `PooledByteBufferFactory`，比如将 Response 里的数据读到 `pooledOutputStream` 里去

```java
class NetworkFetchProducer {
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
}
```



## DiskCacheWriteProducer