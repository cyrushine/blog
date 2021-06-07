(viomi-statistics-lib)[https://gitlab.viomi.com.cn/app/Android/Libs/Statistical] 是云米科技自研的 Android 埋点库

1. `StatisticsManager.writeClickStatistics` 记录点击事件
2. `StatisticsManager.writeShowStatistics` 记录曝光事件
3. `StatisticsManager.writeStatistics` 记录通用的事件

事件是以 JSON 文本的格式持久化在文件里的，每个事件一行

```java
/**
 * 将统计数据写入到文本文件中
 * 新格式
 */
public static void writeStatisticsNew(String category, String actionType, String business, JSONArray valueArray,String actionContent) {
    if (TextUtils.isEmpty(category)) return;
    JSONObject jsonObject = new JSONObject();
    try {
        jsonObject.put("category", category);
        jsonObject.put("action_type", actionType);
        jsonObject.put("action", actionContent);
        jsonObject.put("act_time", System.currentTimeMillis());
        jsonObject.put("bu", business);
        if (valueArray != null)
            jsonObject.put("value", valueArray);
    } catch (JSONException e) {
        e.printStackTrace();
    }
    String content = jsonObject.toString();
    LogUtil.d(TAG, "the content:" + content);
    if (TextUtils.isEmpty(content))
        return;
    // 生成文件夹之后，再生成文件，不然会出错
    File dir = getLogFileDir();
    if (!dir.exists())
        dir.mkdirs();
    String strFilePath = getLogFilePath();
    Log.d(TAG, "the dir:" + dir.getAbsolutePath() + " ---strFilePath:" + strFilePath);
    // 每次写入时，都换行写
    String strContent = content + "\r\n";
    try {
        File file = new File(strFilePath);
        if (!file.exists()) {
            LogUtil.d(TAG, "Create the file: " + strFilePath);
            LogUtil.d(TAG, "Create parent file: " + file.getParentFile().mkdirs());
            LogUtil.d(TAG, "Create new file: " + file.createNewFile());
        }
        RandomAccessFile raf = new RandomAccessFile(file, "rwd");
        raf.seek(file.length());
        raf.write(strContent.getBytes());
        raf.close();
    } catch (Exception e) {
        LogUtil.e("error:", e + "");
    }
}
```

HTTP POST 上传到服务器（手动触发）

```java

public class StatisticsManager {
    /**
     * 手动触发日志上报
     * 立即上报一次日志
     */
    public void forceUpload() {
        getContext().sendBroadcast(new Intent(StatisticsConstants.BROADCAST_FORCE_UPLOAD));
    }

    public boolean startWork() {
        // ...
        if (statisticConfig.timerUploadEnable && broadcastReceiver == null) {//埋点日志上报
            broadcastReceiver = new StatisticReceiver();
            IntentFilter filter = new IntentFilter();
            filter.addAction(Intent.ACTION_TIME_TICK);
            filter.addAction(StatisticsConstants.BROADCAST_FORCE_UPLOAD);
            statisticConfig.applicationContext.registerReceiver(broadcastReceiver, filter);
        }
        // ...
        return true;
    }
}

// 通过广播触发日志上传
public class StatisticReceiver extends BroadcastReceiver {
    private void uploadStaticFile(final File file) {
        AsyncTask.execute(() -> {
            if (file.length() > StatisticsManager.getInstance().getConfig().maxFileSize) { // 文件过大,超过阀值，删除重新采集
                FileUtil.deleteChildFile(file); // 过大，删除后对删除进行记录
                mCount = upErrCount = 0;//清零
                return;
            }
            int upLoadFileRandom = new Random().nextInt(30);//随机生成0到30秒延时请求
            JSONObject temps = StatisticUtil.getStatisticJson(file);
            if (temps == null) {//日志文件解析异常
                Log.w(TAG, "日志异常");
                FileUtil.deleteChildFile(file);
                mCount = upErrCount = 0;//清零
                return;
            }
            Observable<MobBaseRes> observable = null;
            if (StatisticsManager.getInstance().getConfig().mStatisticDeviceType == StatisticDeviceType.DEVICE_FRIDGE) {
                observable = HttpManager.getInstance().getApiService()
                        .uploadBuriedPointStatistical(getToken(), HttpManager.getInstance().getRequestBody(temps))
                        .compose(RxSchedulerUtil.SchedulersTransformer2())
                        .delaySubscription(upLoadFileRandom, TimeUnit.SECONDS)
                        .onTerminateDetach();
            } else if (StatisticsManager.getInstance().getConfig().mStatisticDeviceType == StatisticDeviceType.DEVICE_TV) {
                observable = HttpManager.getInstance().getApiService()
                        .uploadBuriedPointStatisticalTv(getToken(), HttpManager.getInstance().getRequestBody(temps))
                        .compose(RxSchedulerUtil.SchedulersTransformer2())
                        .delaySubscription(upLoadFileRandom, TimeUnit.SECONDS)
                        .onTerminateDetach();
            } else if (StatisticsManager.getInstance().getConfig().mStatisticDeviceType == StatisticDeviceType.DEVICE_XV) {
                observable = HttpManager.getInstance().getApiService()
                        .uploadBuriedPointStatisticalXv(getToken(), HttpManager.getInstance().getRequestBody(temps))
                        .compose(RxSchedulerUtil.SchedulersTransformer2())
                        .delaySubscription(upLoadFileRandom, TimeUnit.SECONDS)
                        .onTerminateDetach();
            } else {
                String token = StatisticsManager.getInstance().getConfig().token;
                if (TextUtils.isEmpty(token))
                    token = StatisticsManager.getInstance().getConfig().httpDebug ? HttpManager.TOKEN_OTHER_DEBUG : HttpManager.TOKEN_OTHER_RELEASE;
                String url = StatisticsManager.getInstance().getConfig().url;
                if (TextUtils.isEmpty(url))
                    url = StatisticsManager.getInstance().getConfig().httpDebug ? HttpManager.URL_OTHER_DEBUG : HttpManager.URL_OTHER_RELEASE;
                observable = HttpManager.getInstance().getApiService()
                        .uploadStatisticalOther(url, token, HttpManager.getInstance().getRequestBody(temps))
                        .compose(RxSchedulerUtil.SchedulersTransformer2())
                        .delaySubscription(upLoadFileRandom, TimeUnit.SECONDS)
                        .onTerminateDetach();
            }
            ;
            if (mUploadDelayDisposable != null && !mUploadDelayDisposable.isDisposed()) {
                mUploadDelayDisposable.dispose();
                mUploadDelayDisposable = null;
            }
            mUploadDelayDisposable = observable.subscribe(s -> {
                        Log.e(TAG, "uploadStaticFile success：" + s.toString() + " code is:" + s.getCode());
                        if (s.getCode() == 200) {//上传成功
                            if (!StatisticsManager.getInstance().getConfig().test)//测试模式下不删除本地日志
                                FileUtil.deleteChildFile(file);//
                            mCount = upErrCount = 0;
                        } else {
                            mCount = 0;//重新计数
                            upErrCount = (upErrCount == 0) ? 2 : (upErrCount * 2);
                            if (upErrCount > MAX_INTERVAL)
                                upErrCount = MAX_INTERVAL;
                        }
                    },
                    throwable -> {
                        Log.e(TAG, "uploadStaticFile fail!msg=" + throwable.toString());
                        mCount = 0;//重新计数
                        upErrCount = (upErrCount == 0) ? 2 : (upErrCount * 2);
                        if (upErrCount > MAX_INTERVAL)
                            upErrCount = MAX_INTERVAL;
                    });
        });
    }

    public String getToken() {
        if (StatisticsManager.getInstance().getConfig().mStatisticDeviceType == StatisticDeviceType.DEVICE_FRIDGE) {//冰箱
            return StatisticsManager.getInstance().getConfig().httpDebug ? HttpManager.TOKEN_FRIDGE_DEBUG :
                    HttpManager.TOKEN_FRIDGE_RELEASE;
        } else if (StatisticsManager.getInstance().getConfig().mStatisticDeviceType == StatisticDeviceType.DEVICE_XV) {//小V
            return StatisticsManager.getInstance().getConfig().httpDebug ? HttpManager.TOKEN_XW_DEBUG :
                    HttpManager.TOKEN_XW_RELEASE;
        } else if (StatisticsManager.getInstance().getConfig().mStatisticDeviceType == StatisticDeviceType.DEVICE_TV) {//电视
            return StatisticsManager.getInstance().getConfig().httpDebug ? HttpManager.TOKEN_TV_DEBUG :
                    HttpManager.TOKEN_TV_RELEASE;
        }
        return HttpManager.TOKEN_FRIDGE_RELEASE;
    }
}
```