1. 在 `Activity` 里执行 callback（`Handler`，io callback），导致内存泄漏，通过 `LeakCanary` 发现内存泄漏，引入 `LiveData` 和 `ViewModel` 解决，`LiveData` 会主动断开 `Activity` 之前的连接，这里 `Activity` 与 GC Root 就没有路径了

2. 通过 `Glide.with(activity)` 下载图片，`Activity.onDestroy` 导致下载请求被取消

3. `ThreadPoolExecuterService` 参数不对导致任务提交失败（max 太小，`ArrayBlockingQueue` 小了）

4. 错误使用 `CopyOnWriteArrayList`，有个业务会批量生成众测活动的分享卡片，开多个线程去生成然后放在 `CopyOnWriteArrayList` 里，当时只考虑到它是线程安全的，没考虑到它的写性能代价很高

5. 单个 deocde 线程，降低内存抖动，就像 `Glide` 那样，依然是「有个业务会批量生成众测活动的分享卡片」