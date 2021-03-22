---
title: Android 图形栈（三）render thread
date: 2020-12-27 12:00:00 +0800
categories: [Android, Framework]
tags: [vsync, render thread]
---

接着[上一篇文章](../ui-thread-in-vsync/)，在上篇文章里我们知道了 ui thread 在 view drawing 阶段产生了 `DisplayList`，而 render thread 会根据 `DisplayList` 执行真正的渲染工作，主要是 `DrawFrameTask.syncFrameState` 和 `CanvasContext.draw` 这两个方法

## syncFrameState

重要的方法有三个：`makeCurrent`，`unpinImages` 和 `prepareTree`

`TreeInfo` 用来在后续的一系列操作中收集信息，你会看到它在各个方法中作为参数传递

```cpp
bool DrawFrameTask::syncFrameState(TreeInfo& info) {
    // ...
    bool canDraw = mContext->makeCurrent();
    mContext->unpinImages();
    // ...
    mContext->setContentDrawBounds(mContentDrawBounds);
    mContext->prepareTree(info, mFrameInfo, mSyncQueued, mTargetNode);
    // ... If prepareTextures is false, we ran out of texture cache space
    return info.prepareTextures;
}
```

## makeCurrent

```cpp
bool CanvasContext::makeCurrent()
MakeCurrentResult SkiaOpenGLPipeline::makeCurrent() {
    // ...
    if (!mEglManager.makeCurrent(mEglSurface, &error)) {
        return MakeCurrentResult::AlreadyCurrent;
    }
    // ...
}

bool EglManager::makeCurrent(EGLSurface surface, EGLint* errOut, bool force) {
    // ...
    if (!eglMakeCurrent(mEglDisplay, surface, surface, mEglContext)) {
    // ...
}
```

最终是调用了 opengl 的 `[eglMakeCurrent](https://www.khronos.org/registry/EGL/sdk/docs/man/html/eglMakeCurrent.xhtml)` 方法准备 opengl 环境；现在是 ui thread，并不会在这里进行渲染，而是为了待会将 mutable images 上传到 gpu

opengl api 都是像 `glDrawArrays`、`glDrawElements`、`glBindTexture` 这样只有方法名和参数的，它的上下文是绑定在 thread 上的，在调用 opengl api 前 `eglMakeCurrent` 就是确保当前线程有 opengl 上下文；mEglDisplay 可以理解为设备的屏幕；opengl 有双缓冲，一个被主线程读取，一个被渲染线程写入，就是第二和第三个参数，渲染完交换一下，读变写，写变读，当前都是用得同一个 surface；第四个就是 opengl 的上下文，保存了 opengl 状态机

## unpinImages

```cpp
/** \class SkImage
    SkImage describes a two dimensional array of pixels to draw. The pixels may be
    decoded in a raster bitmap, encoded in a SkPicture or compressed data stream,
    or located in GPU memory as a GPU texture.

    SkImage cannot be modified after it is created. SkImage may allocate additional
    storage as needed; for instance, an encoded SkImage may decode when drawn.

    SkImage width and height are greater than zero. Creating an SkImage with zero width
    or height returns SkImage equal to nullptr.

    SkImage may be created from SkBitmap, SkPixmap, SkSurface, SkPicture, encoded streams,
    GPU texture, YUV_ColorSpace data, or hardware buffer. Encoded streams supported
    include BMP, GIF, HEIF, ICO, JPEG, PNG, WBMP, WebP. Supported encoding details
    vary with platform.
*/
class SK_API SkImage	

    /**
     * Pin any mutable images to the GPU cache. A pinned images is guaranteed to
     * remain in the cache until it has been unpinned. We leverage this feature
     * to avoid making a CPU copy of the pixels.
     *
     * @return true if all images have been successfully pinned to the GPU cache
     *         and false otherwise (e.g. cache limits have been exceeded).
     */
    bool pinImages(std::vector<SkImage*>& mutableImages) {
        return mRenderPipeline->pinImages(mutableImages);
    }

    /**
     * Unpin any image that had be previously pinned to the GPU cache
     */
    void unpinImages() { mRenderPipeline->unpinImages(); }
```

`SkImage` 对一切图像的抽象，包括 jpg、webp 等压缩格式、Bitmap 位图、流、甚至 gpu 上的纹理

`pinImages` 把在内存的 SkImage 作为纹理上传到 gpu 内存，然后可以通过纹理 id 引用，从而避免在内存里操作（复制）像素

`unpinImages` 从 gpu 内存里移除纹理

## DamageAccumulator

`DamageAccumulator` 是 `DirtyStack` stack（FIFO，用双向链表实现），用来累计脏区，它的一般用法是这样的

```cpp
info.damageAccumulator->pushTransform(this);
damageSelf(info);
info.damageAccumulator->popTransform();

// 将 node 压入栈顶
void DamageAccumulator::pushTransform(const RenderNode* transform)

// 更新栈顶元素的脏区 = 已有脏区 + node 大小，也就是并集
void RenderNode::damageSelf(TreeInfo& info) {
    if (isRenderable()) {
        mDamageGenerationId = info.damageGenerationId;
        if (properties().getClipDamageToBounds()) {
            info.damageAccumulator->dirty(0, 0, properties().getWidth(), properties().getHeight());
        } else {
            // Hope this is big enough?
            // TODO: Get this from the display list ops or something
            info.damageAccumulator->dirty(DIRTY_MIN, DIRTY_MIN, DIRTY_MAX, DIRTY_MAX);
        }
    }
}
void DamageAccumulator::dirty(float left, float top, float right, float bottom) {
    mHead->pendingDirty.join({left, top, right, bottom});
}

// 弹出栈顶元素 head，并将栈顶元素的脏区合并到当前栈顶元素 prev 的脏区
// 可见脏区是累加的
void DamageAccumulator::popTransform() {
    LOG_ALWAYS_FATAL_IF(mHead->prev == mHead, "Cannot pop the root frame!");
    DirtyStack* dirtyFrame = mHead;
    mHead = mHead->prev;
    switch (dirtyFrame->type) {
        case TransformRenderNode:
            applyRenderNodeTransform(dirtyFrame);
            break;
        case TransformMatrix4:
            applyMatrix4Transform(dirtyFrame);
            break;
        case TransformNone:
            mHead->pendingDirty.join(dirtyFrame->pendingDirty);
            break;
        default:
            LOG_ALWAYS_FATAL("Tried to pop an invalid type: %d", dirtyFrame->type);
    }
}
```

## prepareTree

```cpp
void CanvasContext::prepareTree(TreeInfo& info, int64_t* uiFrameInfo, int64_t syncQueued, RenderNode* target) {
    // ...
    for (const sp<RenderNode>& node : mRenderNodes) {
        // Only the primary target node will be drawn full - all other nodes would get drawn in
        // real time mode. In case of a window, the primary node is the window content and the other
        // node(s) are non client / filler nodes.
        info.mode = (node.get() == target ? TreeInfo::MODE_FULL : TreeInfo::MODE_RT_ONLY);
        node->prepareTree(info);
    }
    // ...
}
```

target 是 `HardwareRenderer.mRootNode` 对应的 native `RootRenderNode`，`mRenderNodes` 正常情况下应该只有一个元素 target，所以这里应该总是 `TreeInfo::MODE_FULL`

```cpp
void RenderNode::prepareTree(TreeInfo& info)
void RenderNode::prepareTreeImpl(TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer) {    
    // ...
    if (info.mode == TreeInfo::MODE_FULL) {
        pushStagingDisplayListChanges(observer, info);
    }
    // ...
    if (mDisplayList) {
        info.out.hasFunctors |= mDisplayList->hasFunctor();
        bool isDirty = mDisplayList->prepareListAndChildren(observer, info, childFunctorsNeedLayer,
            [](RenderNode* child, TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer) {
                child->prepareTreeImpl(observer, info, functorsNeedLayer);
            });
        // ...
    }
    pushLayerUpdate(info);
    // ...
}

void RenderNode::pushStagingDisplayListChanges(TreeObserver& observer, TreeInfo& info) {
    if (mNeedsDisplayListSync) {
        mNeedsDisplayListSync = false;
        damageSelf(info);
        syncDisplayList(observer, &info);
        damageSelf(info);
    }
}

void RenderNode::syncDisplayList(TreeObserver& observer, TreeInfo* info) {
    if (mStagingDisplayList) {
        mStagingDisplayList->updateChildren([](RenderNode* child) { child->incParentRefCount(); });
    }
    deleteDisplayList(observer, info);
    mDisplayList = mStagingDisplayList;
    mStagingDisplayList = nullptr;
    if (mDisplayList) {
        WebViewSyncData syncData {
            .applyForceDark = info && !info->disableForceDark
        };
        mDisplayList->syncContents(syncData);
        handleForceDark(info);
    }
}
```

还记得在[上篇文章](../ui-thread-in-vsync/)里提到， DisplayList 在 endRecording 阶段被放在 `RenderNode.mStagingDisplayList`，这时候转移到 `mDisplayList`

```cpp
bool SkiaDisplayList::prepareListAndChildren(TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer,
        std::function<void(RenderNode*, TreeObserver&, TreeInfo&, bool)> childFn) {
    // ...
    if (info.prepareTextures && !info.canvasContext.pinImages(mMutableImages)) {
        info.prepareTextures = false;
        info.canvasContext.unpinImages();
    }

    bool hasBackwardProjectedNodesHere = false;
    bool hasBackwardProjectedNodesSubtree = false;

    for (auto& child : mChildNodes) {
        RenderNode* childNode = child.getRenderNode();
        // ...
        childFn(childNode, observer, info, functorsNeedLayer);
        // ...
    }
    // ...
}
```

`SkiaRecordingCanvas.drawBitmap` 等方法会将 mutable image 放入 `mMutableImages`，然后在 prepare 阶段上传到 gpu；为什么这么做呢，我猜想是在渲染之前需要确保所有内容都计算完毕并确保不变，mutable image 可以被修改，所以放到 gpu 里确保不被改动，immutable image 因为本身就不可修改所以无需这样处理

`TreeInfo.prepareTextures` 标识 mutable images 有没上传成功；如果上传成功，ui thread 在 `DrawFrameTask::syncFrameState` 之后就会被唤醒，否则会一直阻塞直到 `CanvasContext.draw` 完成；这对 ui thread 有着很大的影响

`SkiaDisplayList.mChildNodes` 应该只有一个 `DecorView.mRenderNode`，在这里它的 `prepareTreeImpl` 被调用

```cpp
void RenderNode::pushLayerUpdate(TreeInfo& info) {
    // ...
    if (info.canvasContext.createOrUpdateLayer(this, *info.damageAccumulator, info.errorHandler)) {
        damageSelf(info);
    }
    // ... 将有 layer 的 RenderNode 和它的脏区加入 TreeInfo.layerUpdateQueue
    SkRect dirty;
    info.damageAccumulator->peekAtDirty(&dirty);
    info.layerUpdateQueue->enqueueLayerWithDamage(this, dirty);
    // ...
}

// 如果 node 没有 layer，或者 node 的大小发送了改变，则新建 layer
bool SkiaPipeline::createOrUpdateLayer(RenderNode* node, const DamageAccumulator& damageAccumulator,
                                       ErrorHandler* errorHandler) {
    // ...
    SkSurface* layer = node->getLayerSurface();
    if (!layer || layer->width() != surfaceWidth || layer->height() != surfaceHeight) {
        SkImageInfo info;
        info = SkImageInfo::Make(surfaceWidth, surfaceHeight, getSurfaceColorType(),
                                 kPremul_SkAlphaType, getSurfaceColorSpace());
        SkSurfaceProps props(0, kUnknown_SkPixelGeometry);
        node->setLayerSurface(SkSurface::MakeRenderTarget(mRenderThread.getGrContext(),
                                                          SkBudgeted::kYes, info, 0,
                                                          this->getSurfaceOrigin(), &props));
    // ...
    }
}
```

我们知道 `View` 是 framework ui api，`RenderNode` 相当于绘制这块 `View` 所需的配置文件；这里新增了一个新的概念 layer，它是这块 `View` 对应的 surface，它所呈现的内容将绘制在这个 surface 上，同 opengl 里 surface 的概念

`RenderNode` 和它的脏区被添加到 `TreeInfo.layerUpdateQueue`；queue 里应该有两个元素，一个是 `HardwareRenderer.mRootNode` 对应的 native `RootRenderNode`，一个是 `DecorView.mRenderNode`

## CanvasContext.draw

```cpp
void CanvasContext::draw() {
    // 还记得上面说的吗，脏区是累加的，这里是总的脏区
    SkRect dirty;
    mDamageAccumulator.finish(&dirty);
    // ...
    Frame frame = mRenderPipeline->getFrame();
    setPresentTime();
    // 再次计算脏区
    SkRect windowDirty = computeDirtyRect(frame, &dirty);
    bool drew = mRenderPipeline->draw(frame, windowDirty, dirty, mLightGeometry, &mLayerUpdateQueue,
                                      mContentDrawBounds, mOpaque, mLightInfo, mRenderNodes,
                                      &(profiler()));
    int64_t frameCompleteNr = getFrameNumber();
    waitOnFences();
    bool requireSwap = false;
    int error = OK;
    bool didSwap = mRenderPipeline->swapBuffers(frame, drew, windowDirty, mCurrentFrameInfo, &requireSwap);
    // ...
}
```

## getFrame

看下 `Frame`，它包含 `EGLSurface` 和 surface 宽高

```cpp
Frame SkiaOpenGLPipeline::getFrame()
Frame EglManager::beginFrame(EGLSurface surface) {
    // ... 
    makeCurrent(surface);
    Frame frame;
    frame.mSurface = surface;
    eglQuerySurface(mEglDisplay, surface, EGL_WIDTH, &frame.mWidth);
    eglQuerySurface(mEglDisplay, surface, EGL_HEIGHT, &frame.mHeight);
    frame.mBufferAge = queryBufferAge(surface);
    eglBeginFrame(mEglDisplay, surface);
    return frame;
}
```

## IRenderPipeline::draw

```cpp
bool SkiaOpenGLPipeline::draw(...)
void SkiaPipeline::renderFrame(...) {
    // ...
    SkCanvas* canvas = tryCapture(surface.get(), nodes[0].get(), layers);
    // draw all layers up front
    renderLayersImpl(layers, opaque);
    renderFrameImpl(clip, nodes, opaque, contentDrawBounds, canvas, preTransform);
    endCapture(surface.get());
    // 绘制「布局边界」、「渲染分析」等 debug 信息，这里略过
    if (CC_UNLIKELY(Properties::debugOverdraw)) {
        renderOverdraw(clip, nodes, contentDrawBounds, surface, preTransform);
    }
    surface->getCanvas()->flush();
    // ...
}
```

`SkiaPipeline::renderLayersImpl`

```cpp
// 看上面，layers 是在 RenderNode::prepareTree 阶段加入的，包括 
// HardwareRenderer.mRootNode 对应的 native RootRenderNode 和 DecorView.mRenderNode
void SkiaPipeline::renderLayersImpl(const LayerUpdateQueue& layers, bool opaque) {
    // ...
    for (size_t i = 0; i < layers.entries().size(); i++) {
        RenderNode* layerNode = layers.entries()[i].renderNode.get();
        // ...
        SkCanvas* layerCanvas = layerNode->getLayerSurface()->getCanvas();
        // ...
        RenderNodeDrawable root(layerNode, layerCanvas, false);
        root.forceDraw(layerCanvas);
        // ...
        // cache the current context so that we can defer flushing it until
        // either all the layers have been rendered or the context changes
        GrContext* currentContext = layerNode->getLayerSurface()->getCanvas()->getGrContext();
        if (cachedContext.get() != currentContext) {
            if (cachedContext.get()) {
                ATRACE_NAME("flush layers (context changed)");
                cachedContext->flush();
            }
            cachedContext.reset(SkSafeRef(currentContext));
        }
    }
    // ...
    cachedContext->flush();
}

void RenderNodeDrawable::forceDraw(SkCanvas* canvas)
void RenderNodeDrawable::drawContent(SkCanvas* canvas) const {
    // displayList 是 layerNode 的，canvas 是 layerNode 对应的 layer surface 的
    // 下面看看这个由 layer surface 作为 backend 的 canvas 做了什么
    displayList->draw(canvas);
    // ...
}
```

RenderNode → mSkiaLayer → layerSurface，这里我们接触到 `RenderNode` 的一个属性/概念 layer

它是 `SkiaLayer` 结构体，主要包含一个 `layerSurface`，它是一个 offscreen render target，backend 可以是纹理、pixels buffer 等

它在 `SkiaPipeline::createOrUpdateLayer` 里通过 `RenderNode::setLayerSurface` 被赋予 `SkSurface_Gpu`；那么上面的 canvas 则是以 `SkGpuDevice` 为 backend 的 `SkCanvas`，所有的 draw 操作（onDrawXXX）都被重定向到 `SkGpuDevice`（drawXXX）；而在 `SkGpuDevice` 里，draw 操作又被重定向到 `GrRenderTargetContext`；在 `GrRenderTargetContext` 里，draw 操作被封装为 `GrDrawOp`，通过 `GrOpsTask::addDrawOp` 加入到 `GrRenderTargetContext::fOpsTask`；而 `GrRenderTargetContext::fOpsTask` 会被 `GrDrawingManager::fDAG` 持有

drawing op 在这里被再次包装，由 `DisplayList` 包装为 `GrDrawOp`

`RenderNode` 的 `DisplayList` 会被渲染到 layer 上

```cpp
/**
 * Call to ensure all drawing to the context has been issued to the underlying 3D API.
 */
void GrContext::flush()

GrSemaphoresSubmitted GrContext::flush(const GrFlushInfo& info, const GrPrepareForExternalIORequests& externalRequests)

GrSemaphoresSubmitted GrDrawingManager::flush(...) {
    // ...
    auto direct = fContext->priv().asDirectContext();
    GrGpu* gpu = direct->priv().getGpu();
    fFlushing = true;
    auto resourceProvider = direct->priv().resourceProvider();
    auto resourceCache = direct->priv().getResourceCache();
    GrOpFlushState flushState(gpu, resourceProvider, &fTokenTracker, fCpuBufferCache);
    GrOnFlushResourceProvider onFlushProvider(this);
    // ...
    int startIndex, stopIndex;
    bool flushed = false;
    {
        while (alloc.assign(&startIndex, &stopIndex, &error)) {
            if (this->executeRenderTasks(startIndex, stopIndex, &flushState, &numRenderTasksExecuted)) {
                flushed = true;
            }
        }
    }

    fDAG.reset();
    this->clearDDLTargets();
    GrSemaphoresSubmitted result = gpu->finishFlush(proxies, numProxies, access, info, externalRequests);
    // Give the cache a chance to purge resources that become purgeable due to flushing.
    if (flushed) {
        resourceCache->purgeAsNeeded();
        flushed = false;
    }
    for (GrOnFlushCallbackObject* onFlushCBObject : fOnFlushCBObjects) {
        onFlushCBObject->postFlush(fTokenTracker.nextTokenToFlush(), fFlushingRenderTaskIDs.begin(),
                                   fFlushingRenderTaskIDs.count());
        flushed = true;
    }
    if (flushed) {
        resourceCache->purgeAsNeeded();
    }
    fFlushingRenderTaskIDs.reset();
    fFlushing = false;
    return result;
}

bool GrDrawingManager::executeRenderTasks(int startIndex, int stopIndex, GrOpFlushState* flushState,
                                          int* numRenderTasksExecuted) {
    SkASSERT(startIndex <= stopIndex && stopIndex <= fDAG.numRenderTasks());

#if GR_FLUSH_TIME_OP_SPEW
    SkDebugf("Flushing opsTask: %d to %d out of [%d, %d]\n", startIndex, stopIndex, 0, fDAG.numRenderTasks());
    for (int i = startIndex; i < stopIndex; ++i) {
        if (fDAG.renderTask(i)) {
            fDAG.renderTask(i)->dump(true);
        }
    }
#endif

    bool anyRenderTasksExecuted = false;
    for (int i = startIndex; i < stopIndex; ++i) {
        GrRenderTask* renderTask = fDAG.renderTask(i);
        if (!renderTask || !renderTask->isInstantiated()) {
             continue;
        }
        SkASSERT(renderTask->deferredProxiesAreInstantiated());
        renderTask->prepare(flushState);
    }

    // Upload all data to the GPU
    flushState->preExecuteDraws();

    // For Vulkan, if we have too many oplists to be flushed we end up allocating a lot of resources
    // for each command buffer associated with the oplists. If this gets too large we can cause the
    // devices to go OOM. In practice we usually only hit this case in our tests, but to be safe we
    // put a cap on the number of oplists we will execute before flushing to the GPU to relieve some
    // memory pressure.
    static constexpr int kMaxRenderTasksBeforeFlush = 100;

    // Execute the onFlush renderTasks first, if any.
    for (sk_sp<GrRenderTask>& onFlushRenderTask : fOnFlushRenderTasks) {
        if (!onFlushRenderTask->execute(flushState)) {
            SkDebugf("WARNING: onFlushRenderTask failed to execute.\n");
        }
        SkASSERT(onFlushRenderTask->unique());
        onFlushRenderTask = nullptr;
        (*numRenderTasksExecuted)++;
        if (*numRenderTasksExecuted >= kMaxRenderTasksBeforeFlush) {
            flushState->gpu()->finishFlush(nullptr, 0, SkSurface::BackendSurfaceAccess::kNoAccess,
                                           GrFlushInfo(), GrPrepareForExternalIORequests());
            *numRenderTasksExecuted = 0;
        }
    }
    fOnFlushRenderTasks.reset();

    // Execute the normal op lists.
    for (int i = startIndex; i < stopIndex; ++i) {
        GrRenderTask* renderTask = fDAG.renderTask(i);
        if (!renderTask || !renderTask->isInstantiated()) {
            continue;
        }

        if (renderTask->execute(flushState)) {
            anyRenderTasksExecuted = true;
        }
        (*numRenderTasksExecuted)++;
        if (*numRenderTasksExecuted >= kMaxRenderTasksBeforeFlush) {
            flushState->gpu()->finishFlush(nullptr, 0, SkSurface::BackendSurfaceAccess::kNoAccess,
                                           GrFlushInfo(), GrPrepareForExternalIORequests());
            *numRenderTasksExecuted = 0;
        }
    }

    SkASSERT(!flushState->opsRenderPass());
    SkASSERT(fTokenTracker.nextDrawToken() == fTokenTracker.nextTokenToFlush());

    // We reset the flush state before the RenderTasks so that the last resources to be freed are
    // those that are written to in the RenderTasks. This helps to make sure the most recently used
    // resources are the last to be purged by the resource cache.
    flushState->reset();
    fDAG.removeRenderTasks(startIndex, stopIndex);
    return anyRenderTasksExecuted;
}
```

最终 `flush` 将 `GrDrawOp` 转换为 opengl 命令并提交给 gpu 执行，下面看看绘制一个矩形 `GrFillRectOp` 是怎么转换为 opengl 命令的

```cpp
void SkCanvas::drawRect(const SkRect& r, const SkPaint& paint)

void SkCanvas::onDrawRect(const SkRect& r, const SkPaint& paint) {
    // ...
    if (needs_autodrawlooper(this, paint)) {
        LOOPER_BEGIN_CHECK_COMPLETE_OVERWRITE(paint, &r, false)
        while (iter.next()) {
            iter.fDevice->drawRect(r, looper.paint());
        }
        LOOPER_END
    }
    // ...
}

void SkGpuDevice::drawRect(const SkRect& rect, const SkPaint& paint)

void GrRenderTargetContext::drawRect(...) {
    // ...
    const SkStrokeRec& stroke = style->strokeRec();
    if (stroke.getStyle() == SkStrokeRec::kFill_Style) {
        this->drawFilledRect(clip, std::move(paint), aa, viewMatrix, rect);
        return;
    }
    // ...
}

void GrRenderTargetContext::drawFilledRect(...) {
    // ...
    GrAAType aaType = this->chooseAAType(aa, GrAllowMixedSamples::kNo);
    this->addDrawOp(clip, GrFillRectOp::Make(fContext, std::move(paint), aaType, viewMatrix,
                                             croppedRect, ss));
}
```

看看 `GrFillRectOp` 的定义

```cpp
std::unique_ptr<GrDrawOp> GrFillRectOp::Make(...)
static std::unique_ptr<GrDrawOp> Make(...)

template <typename Op, typename... OpArgs>
static std::unique_ptr<GrDrawOp> FactoryHelper(GrRecordingContext* context, GrPaint&& paint, OpArgs... opArgs) {
    return GrSimpleMeshDrawOpHelper::FactoryHelper<Op, OpArgs...>(context, std::move(paint), std::forward<OpArgs>(opArgs)...);
}

template <typename Op, typename... OpArgs>
std::unique_ptr<GrDrawOp> GrSimpleMeshDrawOpHelper::FactoryHelper(GrRecordingContext* context,
                                                                  GrPaint&& paint,
                                                                  OpArgs... opArgs) {
    GrOpMemoryPool* pool = context->priv().opMemoryPool();
    MakeArgs makeArgs;
    if (paint.isTrivial()) {
        makeArgs.fProcessorSet = nullptr;
        return pool->allocate<Op>(makeArgs, paint.getColor4f(), std::forward<OpArgs>(opArgs)...);
    }
    // ...
}

FillRectOp(Helper::MakeArgs args, SkPMColor4f paintColor, GrAAType aaType,
               DrawQuad* quad, const GrUserStencilSettings* stencil, Helper::InputFlags inputFlags)
            : INHERITED(ClassID())
            , fHelper(args, aaType, stencil, inputFlags)
            , fQuads(1, !fHelper.isTrivial()) {
    // ...
}
```

```cpp
void GrRenderTargetContext::addDrawOp(const GrClip& clip, std::unique_ptr<GrDrawOp> op,
                                      const std::function<WillAddOpFn>& willAddFn) {
    auto opList = this->getRTOpList();
    // ...
    opList->addDrawOp(std::move(op), analysis, std::move(appliedClip), dstProxy, *this->caps());
}

GrRenderTargetOpList* GrRenderTargetContext::getRTOpList() {
    // ...
    if (!fOpList || fOpList->isClosed()) {
        fOpList = this->drawingManager()->newRTOpList(fRenderTargetProxy.get(), fManagedOpList);
    }
    return fOpList.get();
}

sk_sp<GrRenderTargetOpList> GrDrawingManager::newRTOpList(GrRenderTargetProxy* rtp, bool managedOpList) {
    // ...
    auto resourceProvider = fContext->contextPriv().resourceProvider();
    sk_sp<GrRenderTargetOpList> opList(new GrRenderTargetOpList(resourceProvider, fContext->contextPriv().refOpMemoryPool(), rtp, fContext->contextPriv().getAuditTrail()));
    if (managedOpList) {
        fDAG.add(opList);
    }
    // ...
    return opList;
}

void GrRenderTargetOpList::addDrawOp(...) {
    auto addDependency = [ &caps, this ] (GrSurfaceProxy* p) {
        this->addDependency(p, caps);
    };
    op->visitProxies(addDependency);
    clip.visitProxies(addDependency);
    if (dstProxy.proxy()) {
        addDependency(dstProxy.proxy());
    }
    this->recordOp(std::move(op), processorAnalysis, clip.doesClip() ? &clip : nullptr, &dstProxy, caps);
}
```

```cpp
class FillRectOp final : public GrMeshDrawOp {
    VertexSpec vertexSpec() const {
        auto indexBufferOption = GrQuadPerEdgeAA::CalcIndexBufferOption(fHelper.aaType(), fQuads.count());
        return VertexSpec(fQuads.deviceQuadType(), fColorType, fQuads.localQuadType(),
            fHelper.usesLocalCoords(), GrQuadPerEdgeAA::Domain::kNo,
            fHelper.aaType(),
            fHelper.compatibleWithCoverageAsAlpha(), indexBufferOption);
    }   

    void tessellate(const VertexSpec& vertexSpec, char* dst) const {
        static constexpr SkRect kEmptyDomain = SkRect::MakeEmpty();
        GrQuadPerEdgeAA::Tessellator tessellator(vertexSpec, dst);
        auto iter = fQuads.iterator();
        while (iter.next()) {
            auto info = iter.metadata();
            tessellator.append(iter.deviceQuad(), iter.localQuad(), info.fColor, kEmptyDomain, info.fAAFlags);
        }
    }   

    void onPrePrepareDraws(...) override {
        SkArenaAlloc* arena = context->priv().recordTimeAllocator();
        const VertexSpec vertexSpec = this->vertexSpec();
        const int totalNumVertices = fQuads.count() * vertexSpec.verticesPerQuad();
        const size_t totalVertexSizeInBytes = vertexSpec.vertexSize() * totalNumVertices;
        fPrePreparedVertices = arena->makeArrayDefault<char>(totalVertexSizeInBytes);
        this->tessellate(vertexSpec, fPrePreparedVertices);
    }
}
```