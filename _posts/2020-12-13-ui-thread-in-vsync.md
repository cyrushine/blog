---
title: Android 图形栈（二）ui thread 
date: 2020-12-13 12:00:00 +0800
categories: [Android, Framework]
tags: [vsync, ui thread]
---

## 从一段 systrace 开始

![114050.png](/assets/2020-12-13-ui-thread-in-vsync/114050.png)

这是一段 systrace 记录，看得出来页面是比较流畅的，ui thread 全都在一个 VSYNC_app 内完成绘制，surfaceflinger 也在一个 VSYNC_sf 内完成各个层的合成；但有没发现在 ui thread 完成 `doFrame` 后，总是会有一个 `ReaderThread` 跟在后面，看名字像是跟渲染相关的线程，它跟 ui 绘制有关系吗？平时我们常说的，只要 ui thread 在一个刷新周期 16ms 内完成 view 的绘制，即可保证页面流畅，真的是这样吗？

## ViewRootImpl

```java
// 从 ViewRootImpl 开始
ViewRootImpl.doTraversal()
ViewRootImpl.performTraversals()
ViewRootImpl.performDraw()
ViewRootImpl.draw(boolean fullRedrawNeeded)

// 进入 ThreadedRenderer
ThreadedRenderer.draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
    // ...
    updateRootDisplayList(view, callbacks);
    // ...
    int syncResult = syncAndDrawFrame(choreographer.mFrameInfo);
    // ...
}
```

这里出现了两个很重要的函数：`updateRootDisplayList` 和 `syncAndDrawFrame`，我们一个个看

## updateRootDisplayList

```java
ThreadedRenderer.updateRootDisplayList(View view, DrawCallbacks callbacks) {
    // ...
    updateViewTreeDisplayList(view);
    // ...
    if (mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()) {
        RecordingCanvas canvas = mRootNode.beginRecording(mSurfaceWidth, mSurfaceHeight);
        try {
            // ...
            canvas.drawRenderNode(view.updateDisplayListIfDirty());
            // ...
        } finally {
            mRootNode.endRecording();
        }
    }
}
```

两段逻辑：

1. `updateViewTreeDisplayList(view)`，其中 view 是 root view 也就是 `DecorView`
2. `Canvas.drawRenderNode`，`RenderNode` 是 View 返回的

## updateViewTreeDisplayList

```java
ThreadedRenderer.updateViewTreeDisplayList(View view)
RenderNode View.updateDisplayListIfDirty() {
    final RenderNode renderNode = mRenderNode;
    // ...
    final RecordingCanvas canvas = renderNode.beginRecording(width, height);
    try {
        // ...
        draw(canvas);
        // ...
    } finally {
        renderNode.endRecording();
        setDisplayListProperties(renderNode);
    }
    // ...
    return renderNode;
}
```

最终在这里调用了 `View.draw`，里面就是常规的画布绘制操作，我们继续看看 `RecordingCanvas` 这个类

## RecordingCanvas

```java
public final class RecordingCanvas extends DisplayListCanvas
public abstract class DisplayListCanvas extends BaseRecordingCanvas
public class BaseRecordingCanvas extends Canvas {
    @Override
    public final void drawArc(float left, float top, float right, float bottom, float startAngle,
            float sweepAngle, boolean useCenter, @NonNull Paint paint) {
        nDrawArc(mNativeCanvasWrapper, left, top, right, bottom, startAngle, sweepAngle,
                useCenter, paint.getNativeInstance());
    }

    @Override
    public final void drawBitmap(@NonNull Bitmap bitmap, float left, float top,
            @Nullable Paint paint) {
        throwIfCannotDraw(bitmap);
        nDrawBitmap(mNativeCanvasWrapper, bitmap.getNativeInstance(), left, top,
                paint != null ? paint.getNativeInstance() : 0, mDensity, mScreenDensity,
                bitmap.mDensity);
    }

    @Override
    public final void drawRect(float left, float top, float right, float bottom,
            @NonNull Paint paint) {
        nDrawRect(mNativeCanvasWrapper, left, top, right, bottom, paint.getNativeInstance());
    }
}
```

`RecordingCanvas` 继承自 `DisplayListCanvas`，`DisplayListCanvas` 继承自 `BaseRecordingCanvas`

在 `BaseRecordingCanvas` 里，`View.draw(Canvas)` 所用到的 `drawBitmap`、`drawText`、`drawRect` 等绘图方法都被重定向到 `BaseCanvas.mNativeCanvasWrapper`，继续看看它指向谁

```java
// 从构造开始
RecordingCanvas RenderNode.beginRecording(int width, int height) {
    // ...
    mCurrentRecordingCanvas = RecordingCanvas.obtain(this, width, height);
    return mCurrentRecordingCanvas;
}
RecordingCanvas RecordingCanvas.obtain(RenderNode node, int width, int height) {
    if (node == null) throw new IllegalArgumentException("node cannot be null");
    RecordingCanvas canvas = sPool.acquire();
    if (canvas == null) {
        canvas = new RecordingCanvas(node, width, height);
    } else {
        nResetDisplayListCanvas(canvas.mNativeCanvasWrapper, node.mNativeRenderNode, width, height);
    }
    canvas.mNode = node;
    canvas.mWidth = width;
    canvas.mHeight = height;
    return canvas;
}
```

`RecordingCanvas` 会被频繁地创建和销毁，所以用了池化，池子大小是 25，从池子里拿出的对象用 `nResetDisplayListCanvas` 重置；我们走创建新实例这条路继续看下去

```java
protected RecordingCanvas(RenderNode node, int width, int height) {
    super(nCreateDisplayListCanvas(node.mNativeRenderNode, width, height));
}

// /frameworks/base/core/jni/android_view_DisplayListCanvas.cpp
static jlong android_view_DisplayListCanvas_createDisplayListCanvas(jlong renderNodePtr, jint width, jint height) {
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    return reinterpret_cast<jlong>(Canvas::create_recording_canvas(width, height, renderNode));
}

Canvas* Canvas::create_recording_canvas(int width, int height, uirenderer::RenderNode* renderNode) {
    return new uirenderer::skiapipeline::SkiaRecordingCanvas(renderNode, width, height);
}
```

可以看到 `RecordingCanvas.mNativeCanvasWrapper` 是 native `SkiaRecordingCanvas`

`SkiaRecordingCanvas` 继承自 `SkiaCanvas`，大部分 2D 绘图方法都是由 `SkiaCanvas` 实现的，而 `SkiaCanvas` 又把绘图操作交由 `SkiaCanvas.mCanvas` 执行，我们看下它指向谁

```cpp
explicit SkiaRecordingCanvas(uirenderer::RenderNode* renderNode, int width, int height) {
    initDisplayList(renderNode, width, height);
}

void SkiaRecordingCanvas::initDisplayList(uirenderer::RenderNode* renderNode, int width, int height) {
    mCurrentBarrier = nullptr;
    SkASSERT(mDisplayList.get() == nullptr);

    if (renderNode) {
        mDisplayList = renderNode->detachAvailableList();
    }
    if (!mDisplayList) {
        mDisplayList.reset(new SkiaDisplayList());
    }

    mDisplayList->attachRecorder(&mRecorder, SkIRect::MakeWH(width, height));
    SkiaCanvas::reset(&mRecorder);
}

void SkiaCanvas::reset(SkCanvas* skiaCanvas) {
    if (mCanvas != skiaCanvas) {
        mCanvas = skiaCanvas;
        mCanvasOwned.reset();
    }
    mSaveStack.reset(nullptr);
}
```

`SkiaRecordingCanvas.mRecorder` 被赋值给了 `SkiaCanvas.mCanvas`，它是 `RecordingCanvaas`，而它又把 draw 交由 `RecordingCanvas.fDL` 执行

```cpp
void DisplayListData::drawRect(const SkRect& rect, const SkPaint& paint) {
    this->push<DrawRect>(0, rect, paint);
}

struct DrawRect final : Op {
    static const auto kType = Type::DrawRect;
    DrawRect(const SkRect& rect, const SkPaint& paint) : rect(rect), paint(paint) {}
    SkRect rect;
    SkPaint paint;
    void draw(SkCanvas* c, const SkMatrix&) const { c->drawRect(rect, paint); }
};

void DisplayListData::drawImage(sk_sp<const SkImage> image, SkScalar x, SkScalar y, onst SkPaint* paint, BitmapPalette palette) {
    this->push<DrawImage>(0, std::move(image), x, y, paint, palette);
}

struct DrawImage final : Op {
    static const auto kType = Type::DrawImage;
    DrawImage(sk_sp<const SkImage>&& image, SkScalar x, SkScalar y, const SkPaint* paint, BitmapPalette palette)
            : image(std::move(image)), x(x), y(y), palette(palette) {
        if (paint) {
            this->paint = *paint;
        }
    }
    sk_sp<const SkImage> image;
    SkScalar x, y;
    SkPaint paint;
    BitmapPalette palette;
    void draw(SkCanvas* c, const SkMatrix&) const { c->drawImage(image.get(), x, y, &paint); }
};
```

`RecordingCanvas.fDL` 是个 `DisplayListData`，它把绘图操作的所有参数记录为一个结构体 Op 并记录起来

```cpp
void SkiaRecordingCanvas::initDisplayList(uirenderer::RenderNode* renderNode, int width, int height)
void SkiaDisplayList::attachRecorder(RecordingCanvas* recorder, const SkIRect& bounds)
void RecordingCanvas::reset(DisplayListData* dl, const SkIRect& bounds) {
    this->resetCanvas(bounds.right(), bounds.bottom());
    fDL = dl;
    mClipMayBeComplex = false;
    mSaveCount = mComplexSaveCount = 0;
}
```

总结下：java `RecordingCanvas.mNativeCanvasWrapper` 持有 native `SkiaRecordingCanvas`，`SkiaRecordingCanvas→mDisplayList→mDisplayList` 里记录所有的绘图操作

## endRecording

```cpp
RenderNode View.updateDisplayListIfDirty() {
    final RenderNode renderNode = mRenderNode;
    // ...
    final RecordingCanvas canvas = renderNode.beginRecording(width, height);
    try {
        // ...
        draw(canvas);
        // ...
    } finally {
        renderNode.endRecording();
        // ...
    }
    // ...
    return renderNode;
}

RecordingCanvas.endRecording() {
    // ...
    RecordingCanvas canvas = mCurrentRecordingCanvas;
    mCurrentRecordingCanvas = null;
    long displayList = canvas.finishRecording();
    nSetDisplayList(mNativeRenderNode, displayList);
    canvas.recycle();
}

uirenderer::DisplayList* SkiaRecordingCanvas::finishRecording() {
    // ...
    return mDisplayList.release();
}

void RenderNode::setStagingDisplayList(DisplayList* displayList) {
    mValid = (displayList != nullptr);
    mNeedsDisplayListSync = true;
    delete mStagingDisplayList;
    mStagingDisplayList = displayList;
}

RecordingCanvas.recycle() {
    mNode = null;
    sPool.release(this);
}
```

最终，`RecordingCanvas` 被回收到池里，保存了绘制 Op 的 `SkiaDisplayList` 被转移到 native `RenderNode.mStagingDisplayList`

## drawRenderNode

```java
ThreadedRenderer.updateRootDisplayList(View view, DrawCallbacks callbacks) {
    // ...
    updateViewTreeDisplayList(view);
    // ...
    if (mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()) {
        RecordingCanvas canvas = mRootNode.beginRecording(mSurfaceWidth, mSurfaceHeight);
        try {
            // ...
            canvas.drawRenderNode(view.updateDisplayListIfDirty());
            // ...
        } finally {
            mRootNode.endRecording();
        }
    }
}

void SkiaRecordingCanvas::drawRenderNode(uirenderer::RenderNode* renderNode) {
    // Record the child node. Drawable dtor will be invoked when mChildNodes deque is cleared.
    mDisplayList->mChildNodes.emplace_back(renderNode, asSkCanvas(), true, mCurrentBarrier);
    auto& renderNodeDrawable = mDisplayList->mChildNodes.back();
    if (Properties::getRenderPipelineType() == RenderPipelineType::SkiaVulkan) {
        // Put Vulkan WebViews with non-rectangular clips in a HW layer
        renderNode->mutateStagingProperties().setClipMayBeComplex(mRecorder.isClipMayBeComplex());
    }
    drawDrawable(&renderNodeDrawable);

    // use staging property, since recording on UI thread
    if (renderNode->stagingProperties().isProjectionReceiver()) {
        mDisplayList->mProjectionReceiver = &renderNodeDrawable;
    }
}
```

在上面，`DecorView` 的 DisplayList 已经被更新过了，所以 `view.updateDisplayListIfDirty()` 直接返回它的 `RenderNode`

beginRecording → draw → endRecording 三步走跟上面的是一样的，`HardwareRenderer.mRootNode` 对应的是 native `RootRenderNode`；它的 `mStagingDisplayList` 只有一个 `RenderNodeDrawable`

## syncAndDrawFrame

```java
// 现在从新回到开头的地方
void ThreadedRenderer.draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
    // ... 这里会调用 View.draw(Canvas)，并把绘制 op 保存起来
    updateRootDisplayList(view, callbacks);
    // ... 这个方法看起来会执行真正的绘制操作，进去看下
    int syncResult = syncAndDrawFrame(choreographer.mFrameInfo);
    // ...
}

// 进入 native
public int HardwareRenderer.syncAndDrawFrame(@NonNull FrameInfo frameInfo) {
    return nSyncAndDrawFrame(mNativeProxy, frameInfo.frameInfo, frameInfo.frameInfo.length);
}
```

```cpp
// frameworks/base/core/jni/android_view_ThreadedRenderer.cpp
static int android_view_ThreadedRenderer_syncAndDrawFrame(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jlongArray frameInfo, jint frameInfoSize) {
    // ...
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    env->GetLongArrayRegion(frameInfo, 0, frameInfoSize, proxy->frameInfo());
    return proxy->syncAndDrawFrame();
}

int RenderProxy::syncAndDrawFrame() {
    return mDrawFrameTask.drawFrame();
}

int DrawFrameTask::drawFrame()
// 到此都还是在 ui thread
void DrawFrameTask::postAndWait() {
    AutoMutex _lock(mLock);
    // RenderThread 继承自 Thread，DrawFrameTask::run 被提交至 RenderThread 的 WorkQueue 等待执行
    mRenderThread->queue().post([this]() { run(); });
    // 这里会导致 ui thread 阻塞，直到 DrawFrameTask::run() 里把相关数据同步过来后才恢复 ui thread
    mSignal.wait(mLock);
}

// 记住此时已经是在 RenderThread 而不是 ui thread，ui thread 此时被阻塞了
void DrawFrameTask::run() {
    ATRACE_NAME("DrawFrame");

    bool canUnblockUiThread;
    bool canDrawThisFrame;
    {
        TreeInfo info(TreeInfo::MODE_FULL, *mContext);
        canUnblockUiThread = syncFrameState(info);
        canDrawThisFrame = info.out.canDrawThisFrame;

        if (mFrameCompleteCallback) {
            mContext->addFrameCompleteListener(std::move(mFrameCompleteCallback));
            mFrameCompleteCallback = nullptr;
        }
    }

    // Grab a copy of everything we need
    CanvasContext* context = mContext;
    std::function<void(int64_t)> callback = std::move(mFrameCallback);
    mFrameCallback = nullptr;

    // From this point on anything in "this" is *UNSAFE TO ACCESS*
    if (canUnblockUiThread) {
        unblockUiThread();
    }

    // Even if we aren't drawing this vsync pulse the next frame number will still be accurate
    if (CC_UNLIKELY(callback)) {
        context->enqueueFrameWork(
                [callback, frameNr = context->getFrameNumber()]() { callback(frameNr); });
    }

    if (CC_LIKELY(canDrawThisFrame)) {
        context->draw();
    } else {
        // wait on fences so tasks don't overlap next frame
        context->waitOnFences();
    }

    if (!canUnblockUiThread) {
        unblockUiThread();
    }
}
```

这里有两个函数是需要关注的：`syncFrameState(info)` 和 `context->draw()`，但不打算深入下去，因为后续涉及到很多状态判断和方法调用，而且会进入 skia 引擎的相关方法，不是代码跟踪就可以理解的（凡是跟 opengl 相关的代码，经过层层封装都不太好理解）

搜索 `DrawFrameTask` 或者 `DisplayList` 等关键字可以找到如何把 `DisplayList` 处理为 `egl` 相关指令的文章，这里根据[《Android N中UI硬件渲染（hwui）的HWUI_NEW_OPS(基于Android 7.1)》](https://blog.csdn.net/jinzhuojun/article/details/54234354) 的描述总结下此阶段的工作：

1. `syncFrameState(info)` 上传纹理
2. `unblockUiThread()` 然后恢复 ui thread
3. `context->draw()` 输出 egl 指令

## 总结

![52054.png](/assets/2020-12-13-ui-thread-in-vsync/52054.png)

这是一个 sync_app 信号内的 systrace，结合这张图片，尝试解答开头提出的问题

1. app 内的 ui 渲染至少包含两个线程：ui thread（执行 `View.draw()`）和 render thread（执行 egl 指令）
2. ui thread 和 render thread 有一段重合的地方，也就是在 ui thread 完成「Record `View#draw()`」后，ui thread 被阻塞了；而 render thread 开始执行 `syncFrameState`，完成后恢复 ui thread，此时 ui thread 的任务已完成，后续的都是 render thread 的任务了
3. 从时长看，ui thread 执行 `Choreographer#doFrame` 用时 4ms，render thread 执行 `DrawFrame` 用时 8ms；render thread 的存在大大地释放了 ui thread 的压力
4. 要在 16ms 内完成一帧的绘制，不能都让 ui thread 给消耗掉了，还得留出一段时间给 render thread，也就是说 ui thread 在一帧内的任务 `Choreographer#doFrame` 耗时要小于 16ms 才行；而且从上图看 render thread 的耗时远大于 ui thread，留给 ui thread 的时间应该是远小于 16ms 的

## 一些概念

1. `DisplayList`：对 `Canvas` 的所有绘制操作并不会立刻执行，而是保存为 `DisplayList`；这一过程称为“录制”，后续可以“回放”进行渲染
2. `View` 是表，是 framework 暴露出来的 ui api，就像 DOM 是浏览器暴露出来的 ui api；`RenderNode` 是里