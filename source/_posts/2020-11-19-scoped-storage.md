---
title: Scoped Storage（沙盒）
date: 2020-11-19 12:00:00 +0800
tags: [Scoped Storage]
---

## 以前的存储访问权限

先看看以前（ < 10/Q ）的存储访问权限是怎样的

从 APP 目录（app-specific directories）的角度看，可以分为：

1. 内部存储（app-specific directories in internal storage）
    1. 对于 app 自己，无需申请任何读/写权限
    2. 对于其他 app，没有访问权限
2. 外部存储（app-specific directories in external storage）
    1. 对于 app 自己，无需申请任何读/写权限
    2. 对于其他 app，可以访问但需要申请读/写权限

从存储设备的角度看，可以分为：

1. 内部存储，除了自己的目录，没有访问权限
2. 外部存储，可以访问但需申请读/写权限（除了自己的目录）

## 启用 Scoped Storage 后

而启用 Scoped Storage（ ≥ 10/Q）后，APP 只能通过下述三种方法访问文件：

1. 对于 APP 目录（ app-specific directories ），无论是内部存储还是外部存储，无需申请任何读/写权限即可访问（ File API ），APP 卸载时被删除
2. 通过 MediaStore API 操作媒体文件，读/写 APP 自己的文件时无需任何权限，访问其他 APP 的内容时需要申请 READ_EXTERNAL_STORAGE；开启 Scoped Storage 后，只能访问 `MediaStore.Images`，`MediaStore.Video`，`MediaStore.Audio` 和自己的 `MediaStore.Downloads`
3. 通过 SAF（ `Storage Access Framework`，也就是文件选择器 ）读/写其他的文件，参考 [Open files using storage access framework](https://developer.android.google.cn/guide/topics/providers/document-provider)
4. 其他方式即使拥有读写权限也会抛出 `java.io.IOException: Permission denied`

兼容性（ targetSDK ）

1. < 29，默认关闭，可以通过 `requestLegacyExternalStorage = false` 打开
2. = 29，默认开启，可以通过 `requestLegacyExternalStorage = true` 关闭
3. &gt; 29，强制开启，`requestLegacyExternalStorage` 被忽略

常用的 APP 目录

| API	                                           | Storage            | 权限   | 返回值
|:-------------------------------------------------|:-------------------|:------|:---------------------------------------------------|
| getDataDir()	                                   | internal storage	| NO	| /data/user/0/com.example.myapplication
| getFilesDir()	                                   | internal storage	| NO	| /data/user/0/com.example.myapplication/files
| getCacheDir()	                                   | internal storage	| NO	| /data/user/0/com.example.myapplication/cache
| getDir("apple", Context.MODE_PRIVATE)	           | internal storage	| NO	| /data/user/0/com.example.myapplication/app_apple
| getExternalCacheDir()	                           | external storage	| NO	| /storage/emulated/0/Android/data/com.example.myapplication/cache
| getExternalFilesDir("apple")	                   | external storage	| NO	| /storage/emulated/0/Android/data/com.example.myapplication/files/apple
| Environment.getExternalStoragePublicDirectory()  | external storage	| YES	| /storage/emulated/0/Pictures, /storage/emulated/0/Alarms, ...

## 学会使用 MediaStore API

当 APP 卸载时，APP 目录也随之被删除；如果需要将一些文件，比如图片，持久地保存下来， 媒体文件如图片、视频和音频 CURD 操作可以参考 [Access media files from shared storage](https://developer.android.com/training/data-storage/shared/media)

APP 访问自己创建的媒体文件时，无需额外的权限，因为这些媒体文件的属性「owner app」被设置为此 APP；当 APP 被卸载，这些媒体文件的「owner app」被清空；当 APP 被再次安装并访问这些媒体文件时，需要 `READ_EXTERNAL_STORAGE`，参考 [App attribution of media files](https://developer.android.com/training/data-storage/shared/media#app-attribution)

创建媒体文件时，可以通过 `RELATIVE_PATH` 指定路径，例如图片可以放在 `"${Environment.DIRECTORY_PICTURES}/tangzhi"` 下（/storage/emulated/0/Pictures/tangzhi）

1. `MediaStore.Images` 可选择 `Environment.DIRECTORY_PICTURES` 和 `Environment.DIRECTORY_DCIM`
2. `MediaStore.Video` 可选择 `Environment.DIRECTORY_MOVIES` 和 `Environment.DIRECTORY_DCIM`
3. `MediaStore.Audio` 可选择 `Environment.DIRECTORY_MUSIC`，`Environment.DIRECTORY_RINGTONES`，`Environment.DIRECTORY_PODCASTS` ...
4. `MediaStore.Downloads` 可选择 `Environment.DIRECTORY_DOWNLOADS`

`MediaStore` 一些重要的列

| 字段                     | 解释                                                                         |
|:------------------------|:-----------------------------------------------------------------------------|
| BUCKET_DISPLAY_NAME     | 媒体文件的分类，例如：Camera，Screenshot，WeiXin                                 |
| BUCKET_ID               | 媒体的分类 ID                                                                  |
| DISPLAY_NAME            | 媒体文件名，例如：Screenshot_20201102_123620.jpg                                |
| RELATIVE_PATH           | 在 external storage 的相对路径，例如：Pictures/WeiXin/，DCIM/Camera/，一般是正确的 |
| DATA                    | 一般会将媒体文件的真实路径保存在这一列，但不保证它是正确的                             |