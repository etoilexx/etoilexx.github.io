---
date: 2022-11-25 08:00:00
layout: post
title: Android Screen Recording Practice(Part.3) - Record Video
subtitle: Andorid屏幕录制实践
description: Android屏幕录制实践
image: https://res.cloudinary.com/dm7h7e8xj/image/upload/v1559824306/theme13_dshbqx.jpg
optimized_image: https://res.cloudinary.com/dm7h7e8xj/image/upload/c_scale,w_380/v1559824306/theme13_dshbqx.jpg
category: Andorid System
tags:
  - Android System
  - Andorid屏幕录制实践
author: etoile_xx
---

# Android Screen Recording Practice(Part.3) - Record Video

## Sec.1 How to capture video on Android device

在上篇Record Audio中提到，Google针对媒体操作设计了libmeida的库，其中提供了可同时捕获音视频的方法 - MediaRecorder，而说到MediaRecorder那就离不开其对应的状态机。

![](https://cdn.jsdelivr.net/gh/jonewan/MD_PNG@master/ScreenRecord/media_recorder_status.png)

* Initial：初始态，当通过MeidaRecorder构造方法创建对象后进入该状态；

* Initialized：已初始化态，可以通过在Initial状态调用setAudioSource()/setVideoSource()进入该状态，此时可以通过调用setOutputFormat()方法配置输出格式，随后进入DataSourceConfigured状态；

* DataSourceConfigured：数据源配置态，在该状态下可以通过配置编码器、输出文件、帧率、
预览显示等等。当调用prepare()方法后到达Prepared状态。

* Prepared：就绪态，此时MediaRecorder所有配置已经就绪，再进行配置修改则不生效，可以通过start()进入录制状态，

* Recording：录制态，此时，MediaRecorder已经开启了音/视频录制，并持续将数据输入到配置的输出文件中，此时可以通过stop()方法或reset()方法结束录制并回到Initial状态。

* Released：释放态，通过在Initial状态调用release()方法来进入这个状态，这时将会释放所有和MediaRecorder对象绑定的资源。

* Error：错误状态，当录制发生错误或者调用出错时会进入该状态。

* 除了Released状态外的所有状态均可以通过调用reset()方法回到初始态进行重新配置；

MediaRecorder在使用时需要严格遵守状态图说明中的函数调用先后顺序，在不同的状态调用不同的函数，否则会出现异常。

MeidaRecorder既能录制音频文件也能录制视频文件，当单独进行音频文件的录制时通常采用如下的接口方法调用顺序：

```java
MediaRecorder recorder = new MediaRecorder();
recorder.setAudioSource(MediaRecorder.AudioSource.MIC);
recorder.setOutputFormat(MediaRecorder.OutputFormat.THREE_GPP);
recorder.setAudioEncoder(MediaRecorder.AudioEncoder.AMR_NB);
recorder.setOutputFile(PATH_NAME);
recorder.prepare();
recorder.start();   // 录制开始
...
recorder.stop();
recorder.reset();   // 可以通过setAudioSource()重新使用当前申请的MediaRecorder对象
recorder.release(); // MediaRecorder对象已被释放，无法在进行使用
```

## Sec.2 How to record screen on Android device

那如何进行屏幕视频的录制呢？根据MeidaRecorder的接口定义，首先需要在Initial状态下，通过调用setVideoSource()方法配置一个视频源，其中VideoSource的定义如下：

```java
public static final int DEFAULT = 0x00000000;
public static final int CAMERA = 0x00000001;
public static final int SURFACE = 0x00000002;
```

当录制摄像头时需要选用CAMERA作为视频源，而在录制屏幕时则采用Surface作为视频源。在第一篇中说到过，屏幕视频的录制实际上是通过对VirtualDisplay内容的捕获来实现的，因此，屏幕视频的录制框架如下图所示：

![media_projection_dialog](https://cdn.jsdelivr.net/gh/jonewan/MD_PNG@master/ScreenRecord/video_record_process_v1.png)

具体的实现方法如下：

* 创建VirtualDisplay用于提供屏幕内容的数据源：

```java
private VirtualDisplay createVirtualDisplay() {
    if (Objects.isNull(mInputSurface))
        throw new RuntimeException("MediaRecord must not be null");
    DisplayManager dm = (DisplayManager) mContext.getSystemService(Context.DISPLAY_SERVICE);
    return dm.createVirtualDisplay(
            RECORD_VIRTUAL_DISPLAY_NAME,
            defaultDisplayWidth,
            defaultDisplayHeight,
            defaultDisplayDensity,
            mInputSurface,
            DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR,
            null,
            null
    );
}
```

* 创建一个视频缓存文件用作MediaRecorder的结果输出：

```java
private File createVideoCache() {
    try {
        File cacheDir = mContext.getCacheDir();
        if (cacheDir.mkdirs())
            throw new IOException("create cache directory failed, not create media recorder.");
        return File.createTempFile(TEMP_VIDEO_FILE_PREFIX, TEMP_VIDEO_FILE_SUFFIX, cacheDir);
    } catch (IOException e) {
        e.printStackTrace();
        Log.e(TAG, Objects.requireNonNull(e.getMessage()));
        return null;
    }
}
```

* 屏幕录制管理方法的构造函数：

```java
public ScreenRecordManager(@NonNull Context context,
                           ScreenRecordParams params,
                           MediaRecorder.OnInfoListener infoListener,
                           MediaRecorder.OnErrorListener errorListener,
                           Handler handler) {
    this.mContext = context;
    this.mHandler = handler;
    this.mInfoListener = infoListener;
    this.mErrorListener = errorListener;
    this.mParams = params;
    DisplayMetrics metrics = new DisplayMetrics();
    WindowManager wm = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
    wm.getDefaultDisplay().getRealMetrics(metrics);
    defaultDisplayWidth = metrics.widthPixels;
    defaultDisplayHeight = metrics.heightPixels;
    defaultDisplayDensity = metrics.densityDpi;
    mVideoCache = createVideoCache();
    mMediaRecorder = createMediaRecorder();
    mInputSurface = Objects.requireNonNull(mMediaRecorder).getSurface();
    mVirtualDisplay = createVirtualDisplay();
}
```

* 创建并配置MediaRecorder的相关参数：

```java
private MediaRecorder createMediaRecorder() {
    MediaRecorder mr = new MediaRecorder();
    mr.reset();
    mr.setVideoSource(MediaRecorder.VideoSource.SURFACE);
    mr.setOutputFormat(MediaRecorder.OutputFormat.MPEG_4);
    mr.setVideoEncoder(MediaRecorder.VideoEncoder.H264); // 采用H264编码
    mr.setVideoEncodingProfileLevel(
            MediaCodecInfo.CodecProfileLevel.AVCProfileHigh,
            MediaCodecInfo.CodecProfileLevel.AVCLevel3
    );
    mr.setVideoEncodingBitRate(defaultDisplayWidth * defaultDisplayHeight); // 编码速率采用视频帧大小
    if (Objects.nonNull(mInfoListener))
        mr.setOnInfoListener(mInfoListener);
    if (Objects.nonNull(mErrorListener))
        mr.setOnErrorListener(mErrorListener);
    mr.setVideoSize(defaultDisplayWidth, defaultDisplayHeight);
    mr.setVideoFrameRate(mParams.videoFrameRate);
    mr.setOutputFile(mVideoCache);
    try {
        mr.prepare(); // 调用prepare进入就绪态
    } catch (IOException e) {
        e.printStackTrace();
        Log.e(TAG, Objects.requireNonNull(e.getMessage()));
        return null;
    }

    return mr;
}
```

* 配置是否开启屏幕原点

```java
private void setTapsVisible(boolean turnOn) {
    Settings.System.putInt(mContext.getContentResolver(),
            KEY_SETTINGS_TOUCH_DOT, turnOn ? 1 : 0);
    Log.d(TAG, "<setTapsVisible> set tap visible to " + turnOn);
}
```

* 调用start()开始录制

```java
void start() throws IOException, RemoteException, RuntimeException {
    if (mParams.useTap) setTapsVisible(true);
    mMediaRecorder.start();
    Log.d(TAG, "<start> media recorder has been stared.");
}
```

* 结束屏幕视频录制

```java
void stop() {
    if (mParams.useTap) setTapsVisible(false);
    if (Objects.nonNull(mMediaRecorder)) {
        try {
            mMediaRecorder.setOnErrorListener(null);
            mMediaRecorder.setOnInfoListener(null);
            mMediaRecorder.stop();
        } catch (Exception e) {
            e.printStackTrace();
            mHandler.sendEmptyMessage(MSG_RECORD_ERROR_INTERNAL);
            mVideoCache.delete();
        }
    }

    if (Objects.nonNull(mInputSurface)) {
        mInputSurface.release();
        mInputSurface = null;
        Log.d(TAG, "<release> input surface has been released.");
    }

    if (Objects.nonNull(mVirtualDisplay)) {
        mVirtualDisplay.release();
        mVirtualDisplay = null;
        Log.d(TAG, "<release> virtual display has been released.");
    }

    if (Objects.nonNull(mMediaRecorder)) {
        mMediaRecorder.release();
        mMediaRecorder = null;
        Log.d(TAG, "<release> media recorder has been released.");
    }

    if (Objects.nonNull(mHandler))
        mHandler.sendEmptyMessage(MSG_PROCESSING_SCREEN_RECORD);
}
```

* 视频文件的存储：由于在AndroidQ上开启了分区存储，普通应用不在具备通过File方法操作storage目录下的公用目录权限，因此需要通过MeidaStore的方式进行存储：

```java
RecordLegacy save() throws IOException {
    String fileName = new SimpleDateFormat(RECORD_VIDEO_FILE_NAME_PATTERN,
            Locale.getDefault()).format(new Date());
    ContentValues values = new ContentValues();
    values.put(MediaStore.Video.Media.DISPLAY_NAME, fileName);
    values.put(MediaStore.Video.Media.MIME_TYPE, "video/mp4");
    values.put(MediaStore.Video.Media.DATE_ADDED, System.currentTimeMillis());
    values.put(MediaStore.Video.Media.DATE_TAKEN, System.currentTimeMillis());
    values.put(MediaStore.Video.Media.RELATIVE_PATH, MEDIA_STORE_RELATIVE_PATH);

    ContentResolver cr = mContext.getContentResolver();
    Uri collectionUri = MediaStore.Video.Media.getContentUri(
            MediaStore.VOLUME_EXTERNAL_PRIMARY);
    Uri itemUri = cr.insert(collectionUri, values);
    Log.d(TAG, itemUri.toString());
    OutputStream os = cr.openOutputStream(itemUri, "w");
    Files.copy(mVideoCache.toPath(), os);
    os.close();
    if (mAudioCache != null) mAudioCache.delete();
    Size size = new Size(defaultDisplayWidth, defaultDisplayHeight);
    RecordLegacy recording = new RecordLegacy(itemUri, mVideoCache, size);
    mVideoCache.delete();
    return recording;
}
```
