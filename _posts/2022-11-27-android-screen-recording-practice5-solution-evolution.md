---
date: 2022-11-27 08:00:00
layout: post
title: Android Screen Recording Practice(Part.5) - Solution Evolution
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
# Android Screen Recording Practice(Part.5) - Solution Evolution

## Preface

通用屏幕录制功能拆解开来无非就是系统、MIC音频录制 + 屏幕视频录制，之后将录制的音频结果和视频结果合成为一个成果文件，那我们根据这个思路来进行设计。

## Sec 1. Solution 1 - AudioRecord + MediaCodec + MediaRecorder + MediaMuxer

首先，录制音频则需要前面提到过的AudioRecord来实现，音频的录制架构也已经在前问中提到过，这里不再进行赘述，最终的录制结果生成一个AAC编码的音频文件；视频的录制同样采用前文提到过的MediaRecorder，利用H264编码成为AVC视频文件；之后将音视频文件通过MediaMuxer进行合成编码，最终生成我们需要的MP4屏幕录制视频结果文件；整体架构图如下图所示：

![media_projection_dialog](https://cdn.jsdelivr.net/gh/jonewan/MD_PNG@master/ScreenRecord/screen_recorder_v1.png)

### Problem of Sulution Design

看似顺利成章的方案设计，那整个方案存在哪些问题呢？最大的问题就在于 - 耗时。

从方案框架来看，实际上整个过程对视频和音频的数据处理了两次（在音视频录制时分别处理了一次，在合成时又处理了一次），若录制时间较短或者数据量较小（音频通道静音或屏幕长时间无变化）时感受不明显，当录制时间较长或者录制数据量较大时，在用户点击停止录制后的等待时间可能都已经超过了录制的原本时间。

这种交互体验当然是无法接受的，那要怎么样优化才能提升录制效率呢？

## Sec 2. Solution 2 - AudioRecord + MediaCodec + MediaMuxer

很明显，是否能够找到一种方案只让音频数据和视频数据在录制过程中被处理一次，不要产生过多的中间文件就会直接优化50%+的录制效率。

因此，对整个录制方案重新进行整理，设计了如下的方案架构：

![media_projection_dialog](https://cdn.jsdelivr.net/gh/jonewan/MD_PNG@master/ScreenRecord/screen_recorder_v2.png)

可以看到，视频数据的录制不再采用原本的MediaRecorder了，由于MediaRecorder提供的API方法的自由度较小，无法对数据进行实时控制，因此对视频的录制也采用了MediaCodec的方式；

通过DisplayManager创建的VirtualDisplay获取到Surface数据后提供给MediaCodec编码器的输入缓冲区；同时两路AudioRecord将数据放入MediaCodec编码器的输入缓冲区；通过两路编码器分别将编码后的视频数据和音频数据提供给MediaMuxer，通过MP4编码后实时写入结果视频文件中。

如此以来，便达到了仅处理一次音视频数据的目的，当用户点击停止录制时仅需要将音视频两路编码器的输出缓冲区中的未处理完的数据完成合成即可，无需再读取中间文件，减少了文件操作的出错概率，同时将整个录制视频的处理效率从`O(N)`变为了`O(1)`。

## Sec 3. Source Code Architecture Optimization

在初版设计时并未过多考虑到代码架构的设计，整体程序的运行效率较低，在进行方案优化时对代码架构进行了调整，参考了MVVM设计方法，将用户交互悬浮窗View的相关设计代码进行独立，使用ViewManager进行统一UI逻辑的处理，MediaRecordManager用于处理屏幕录制的相关逻辑，二者相互间通过回掉方法进行传递数据；其中基于对音视频录制逻辑的抽象，利用工厂模式的设计思想对编码与混叠的相关代码逻辑进行了优化，整体的优化后方案类关系图如下图所示：

![](//www.plantuml.com/plantuml/png/hLXVRnmr47-_Js6fBvSsd0IlhQesVofEIjgGIzi7n8DlT_REYlySjZT_QFE0X2h4Gm49aE2b58Je4rA5X52XtwPfsc-1lTxRpAxtDcd4FTpQdj-CnpFZcTczOIpLDXUyDwF9Bfq06QxBHAMWoODUZxWVanPqHfDJoc2LSZvsy0SE47vxYgL4oNMjbUulaMBzQ14_QV3DDCoeXlvIVTbODMkkQU10JGm5EzUfFn-Gf1m4x97_AqPIMI3KLEhSTlE2SFBl3oS_VtxyuxFNNxryy_pfwyU_7N_zQyK6CYrOUqUzobOhrC0cf8msR9PmQWmf5-SAeMDVcHjiA6UvNScp3FJDFSeu7NDtk1Xt9xVl0hobjA1sTKhb158CQpWgM0Rh79-eOdwF58HLjvmGKTAl8d9rK34V8Mfa3T5OgA2NQkviAZI8_OiuaOQ91cgXvEfZ41qRgsbY0phOgeIZbVu2kHEmRKALd9z7oOnDSkqCZxNn_zWHAuSMWZ-zHEpVFfEMC9d2mH9Po32KRNW4_pI775fHoyGFLPOPi65is6UmJBYI4p8pTiU4rOpJYS4YoylNhkiybghxKbcvFR06yRKutwNeZ7JIZceNTeJJGQJWQLuuVlBYvFa_LNQ8Cm9E2VTO2gf-UO81C64GdY5KS5u8SNAR2a1h-oorKpIV0fjCBLf8GHfc3zTc32r2iSKAizlKOccPTj8RQwnO7FfClKSv8WWcOETmXh4Q4gNJJNgmbcjgcPA8XY-KLVWUbO2HrKnE72qCAaV4TegppoWFPKIK0qIgcLE49zurC5Jt9ipRmm1lNqN0czyVlVtbsyKHK4nm5DpCKwPQYqAWN2GCnf4h3HKptlGbFTqWE02BdL8fWQ-gNDecosgW3UKcDRl_hq-RvkXsNEGKn8-TylR9dnTmohOFtyWhY8HIquTKfhokSKSU4r4bmtMsNEakqETAM0lL3eCsfJUOt8LqWnbxxRGc3S8cGvcfwrxvP0gfdtXOEwzF41FG3hsP7s15Xp9b2HZ49ROKvpWotTgDEtkWDKk1J0iTDy4OvvT--IJ2J9OGN7d9OPS-MW-t66yD1pMwOHD9-Hf45R0zC75882cra1LuXa2bhNUc6cXAH10KParHWLQs6MYbhlgKPNP7ZNnFtZTJfUt7dn0tcLILsbF246IAMSV56UyfdWiO9PH3nUyfWOM3N2OPLzG9yX3CJzFqfapT3NG-AWrcAX569-yl4pJxO9cC3osuIMfG4p7o28F55ax0rcjjIJ27Kop9ERLmozUa7IR0M2fc1TWgI_aAmrjSwcvBCF1SBHTEhnThGnrdFajzCycI4UCW4-YFbN9AIcB275zddEXmgE0Na--UlNhnEEIxuvVV7t_vry8taaB7862JIZg9t-DYJ6Sp5c9nwMfpFV3xV1_wHWq6f2qQ0k-pkoPckQ-PyofpeUd_IE8pJemQ-TS-TuVKlizy17m6ZL5RMYLWJB-5d-v1sW_gRS249eU45uy0ZP9ebpdYN7gbAtS5yTsYiMArX7DNCoBS7wurp0ew8s9dceknf8o71QICPvg0TGzSaaj6JscmlclXZWzGqxdmeoX0HVr0f5eR9_2iFJCkph8TBLFW29FMlIC6JuePS8yJ8BXCjmF3vek0JCubxMWNIBvADQY1lQ8UD7Tn9TRRf7XWOrWbMB9IvfD0Bkuef8jHJhq4FigPNGmPckrSIYRhTjGsTwMKcHcdXqJKvulhu7sbTsjB2DgyMg8SO8YI5WwSXSBWp6Pa2wHFHDZhg3TmuFDAm87MBQ6xcUgIbwaBYmi7x1ONEeHmfOhtEKbvhSKG-K4X1tyFQDsNZAjXWzyVwyBy-xu8Z8eNbpLgQIZh_XASBMOPL_3mUfk66Tx05o9RgBXbJRzBPiJb3ND1Bfm6-ZXBIDWFetOTpyc7dLjS-M6s3JGznAwTVqV5OT96nr7PqZWNzRVgLzpaFhFJTMKmqY_Fesj39Iv-iQ1vHnOZS1Ijo_HEDh8Qg0WaL0SDtGFV1YljQ-my7oJXkKosg9sIZ76OkK7y4X-NhSvUwjNVZyFxVPYU_FJ5oHzFJxvvvBxvH6rMxwoAFXXSRjdTDnmTRSO5k5067MZ_EQXTGczntpeOFBmUVzBlpLV8jOU30McozvfVSI-7tLkGpGK7vP15IBmtZbodt_K8YX18uC9otsAzivbYnQyvGzHQ4pmXbmFn_0e4YtLnrJTmmL5lb7h3kcQOX6Rei9sVDoAUhiyAtIV2V7LVjKTuxmR8D1VyFm00)


* 由于新方案中采用了双MeidaCodec编码器，而编码的操作实际上大同小异，因此可以通过一个接口方法进行规范，定义了一个编码器通用接口：`IEncoder`
```java
interface IEncoder {
    void prepare() throws IOException;

    void stop();

    void release();

    void setCallback(Callback callback);

    interface Callback {
        void onError(IEncoder encoder, Exception exception);
    }
}
```

* 编码器基类`BaseMediaEncoder`用于实现编码器操作的`IEncoder`基本功能，而具体的数据提供，数据处理等操作则通过定义回掉方法让音视频编码器独立选择处理：

```java
abstract class BaseMediaEncoder implements IEncoder {
    ......
    private final EncoderCallback mEncoderCallback = new EncoderCallback();
    private String mCodecName;
    private MediaCodec mEncoder;
    private Callback mCallback;

    BaseMediaEncoder() {
    }

    public BaseMediaEncoder(String codecName) {
        this.mCodecName = codecName;
    }

    @Override
    public void setCallback(IEncoder.Callback callback) {
        if (!(callback instanceof BaseMediaEncoder.Callback)) {
            throw new IllegalArgumentException();
        }
        this.setCallback((BaseMediaEncoder.Callback) callback);
    }

    void setCallback(BaseMediaEncoder.Callback callback) {
        if (Objects.nonNull(this.mEncoder))
            throw new IllegalStateException("Callback must set before prepare()!");
        this.mCallback = callback;
    }

    @Override
    public void prepare() throws IOException {
        if (Looper.myLooper() == null
                || Looper.myLooper() == Looper.getMainLooper())
            throw new IllegalStateException("Must run on worker thread!");

        if (Objects.nonNull(mEncoder))
            throw new IllegalStateException("Can't repeat the preparation!");

        MediaFormat mediaFormat = createMediaFormat();
        Log.d(TAG, "Create media format: " + mediaFormat);
        final String mimeType = mediaFormat.getString(MediaFormat.KEY_MIME);
        final MediaCodec encoder = createEncoder(mimeType);
        try {
            // Call setCallback() let MediaCodec run as async mode
            if (Objects.nonNull(this.mCallback))
                encoder.setCallback(mEncoderCallback);
            encoder.configure(mediaFormat, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);
            onEncoderConfigured(encoder);
            encoder.start(); // start MediaCodec
        } catch (MediaCodec.CodecException e) {
            Log.e(TAG, "Configure codec failure!\n  with format" + mediaFormat, e);
            throw e;
        }
        mEncoder = encoder;
    }

    @Override
    public void stop() {
        if (Objects.nonNull(mEncoder)) {
            mEncoder.stop();
            Log.i(TAG, "<stop> Encoder has been stopped.");
        }
    }

    @Override
    public void release() {
        if (Objects.nonNull(mEncoder)) {
            mEncoder.release();
            mEncoder = null;
            Log.i(TAG, "<release> Encoder has been released.");
        }
    }

    protected void onEncoderConfigured(MediaCodec encoder) { }

    private MediaCodec createEncoder(String type) throws IOException {
        try {
            if (Objects.nonNull(this.mCodecName)) {
                return MediaCodec.createByCodecName(mCodecName);
            }
        } catch (IOException e) {
            Log.e(TAG, "Create MediaCodec by name '" + mCodecName + "' failed!", e);
        }
        return MediaCodec.createEncoderByType(type);
    }

    protected abstract MediaFormat createMediaFormat();

    protected final MediaCodec getEncoder() {
        return Objects.requireNonNull(mEncoder,
                "Encoder prepare() hasn't been called yet!");
    }

    public final ByteBuffer getOutputBuffer(int index) {
        return getEncoder().getOutputBuffer(index);
    }

    public final ByteBuffer getInputBuffer(int index) {
        return getEncoder().getInputBuffer(index);
    }

    public final void queueInputBuffer(int index, int offset, int size, long pstTs, int flags) {
        getEncoder().queueInputBuffer(index, offset, size, pstTs, flags);
    }

    public final void releaseOutputBuffer(int index) {
        getEncoder().releaseOutputBuffer(index, false);
    }

    static abstract class Callback implements IEncoder.Callback {
        void onInputBufferAvailable(BaseMediaEncoder encoder, int index) {
        }

        void onOutputFormatChanged(BaseMediaEncoder encoder, MediaFormat format) {
        }

        void onOutputBufferAvailable(BaseMediaEncoder encoder, int index, MediaCodec.BufferInfo info) {
        }
    }

    private class EncoderCallback extends MediaCodec.Callback {
        @Override
        public void onInputBufferAvailable(MediaCodec codec, int index) {
            mCallback.onInputBufferAvailable(BaseMediaEncoder.this, index);
        }

        @Override
        public void onOutputBufferAvailable(MediaCodec codec, int index, MediaCodec.BufferInfo info) {
            mCallback.onOutputBufferAvailable(BaseMediaEncoder.this, index, info);
        }

        @Override
        public void onError(MediaCodec codec, MediaCodec.CodecException e) {
            mCallback.onError(BaseMediaEncoder.this, e);
        }

        @Override
        public void onOutputFormatChanged(MediaCodec codec, MediaFormat format) {
            mCallback.onOutputFormatChanged(BaseMediaEncoder.this, format);
        }
    }
}
```

* 音频编码器及配置类`AudioEncoder`中定义了音频编码所需要的配置，如采样率、比特率、通道数等等，可通过修改对应变量实现配置不同的音频录制参数：

```java
class AudioEncoder extends BaseMediaEncoder {
    private final EncodeConfig mConfig;

    AudioEncoder(EncodeConfig config) {
        super(config.codecName);
        this.mConfig = config;
    }

    @Override
    protected MediaFormat createMediaFormat() {
        return mConfig.toFormat();
    }

    public static class EncodeConfig {
        ......
        
        public EncodeConfig() {
            this.codecName = null;
            this.bitRate = AUDIO_BIT_RATE_196K;
            this.sampleRate = AUDIO_SAMPLE_RATE_44_1KHz;
            this.audioEncodeFormat = AUDIO_ENCODE_16BIT;
            this.channelCount = SINGLE_CHANNEL;
            this.audioChannelInMask = AUDIO_CHANNEL_IN_MONO;
        }

        ......
        
        MediaFormat toFormat() {
            MediaFormat format = MediaFormat.createAudioFormat(MIMETYPE_AUDIO_AAC, sampleRate, channelCount);
            format.setInteger(MediaFormat.KEY_AAC_PROFILE, MediaCodecInfo.CodecProfileLevel.AACObjectLC);
            format.setInteger(MediaFormat.KEY_BIT_RATE, bitRate);
            return format;
        }
        ......
    }
}
```

* 视频编码器及配置类`VideoEncoder`与音频配置类似：

```java
class VideoEncoder extends BaseMediaEncoder {

    private static final String TAG = VideoEncoder.class.getSimpleName();
    public static final String TEMP_VIDEO_FILE_PREFIX = "temp_video";
    public static final String TEMP_VIDEO_FILE_SUFFIX = ".mp4";
    public static final String RECORD_VIDEO_FILE_NAME_PATTERN = "'screen-'yyyyMMdd-HHmmss'.mp4'";
    public static final String MEDIA_STORE_RELATIVE_PATH = "Movies/屏幕录制";

    private final EncodeConfig mConfig;
    private Surface mSurface;

    VideoEncoder(EncodeConfig config) {
        super(config.codecName);
        this.mConfig = config;
    }

    @Override
    protected void onEncoderConfigured(MediaCodec codec) {
        mSurface = codec.createInputSurface();
        Log.i(TAG, "VideoEncoder create input surface: " + mSurface);
    }

    @Override
    protected MediaFormat createMediaFormat() {
        return mConfig.toFormat();
    }

    Surface getInputSurface() {
        return Objects.requireNonNull(mSurface, "doesn't prepare()");
    }

    @Override
    public void release() {
        if (mSurface != null) {
            mSurface.release();
            mSurface = null;
        }
        super.release();
    }

    public static class EncodeConfig {
        public int width;
        public int height;
        public int densityDpi;
        ......
        public EncodeConfig(int width, int height, int densityDpi, String codecName) {
            this.codecName = codecName;
            this.width = width;
            this.height = height;
            this.densityDpi = densityDpi;
            this.encodeBitRate = VIDEO_ENCODE_BIT_RATE_BPS_8M;
            this.frameRate = VIDEO_FRAME_RATE_FPS_60;
            this.iframeInterval = VIDEO_I_FRAME_INTERVAL_1_S;
            this.mimeType = MIMETYPE_VIDEO_AVC;
            this.recordMaxDuration = MAX_DURATION_1_HOUR;
        }

        public MediaFormat toFormat() {
            MediaFormat format = MediaFormat.createVideoFormat(mimeType, width, height);
            format.setInteger(MediaFormat.KEY_COLOR_FORMAT,
                    MediaCodecInfo.CodecCapabilities.COLOR_FormatSurface);
            format.setInteger(MediaFormat.KEY_BIT_RATE, encodeBitRate);
            format.setInteger(MediaFormat.KEY_FRAME_RATE, frameRate);
            format.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, iframeInterval);
            if (Objects.nonNull(codecProfileLevel)
                    && codecProfileLevel.profile != 0
                    && codecProfileLevel.level != 0) {
                format.setInteger(MediaFormat.KEY_PROFILE, codecProfileLevel.profile);
                format.setInteger(MediaFormat.KEY_LEVEL, codecProfileLevel.level);
            }
            // maybe useful
            // format.setInteger(MediaFormat.KEY_REPEAT_PREVIOUS_FRAME_AFTER, 10_000_000);
            return format;
        }
        ......
    }
}
```

* 而实际的音视频录制逻辑放在`AudioRecorder`中，整个录制的重要逻辑使用了`HandlerThread`进行处理：

```java
private class RecordHandler extends Handler {

    private final LinkedList<MediaCodec.BufferInfo> mCachedInfoList = new LinkedList<>();
    private final LinkedList<Integer> mMuxOutputBufferIndices = new LinkedList<>();
    private final int mPollRate = 2048_000 / mSampleRate; // poll per 2048 samples

    RecordHandler(Looper l) {
        super(l);
    }

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_AUDIO_PREPARE:
                mMic = createAudioRecord(MediaRecorder.AudioSource.MIC,
                        mSampleRate, mChannelConfig, mEncodingFormat);
                mInternal = createAudioRecord(MediaRecorder.AudioSource.REMOTE_SUBMIX,
                        mSampleRate, mChannelConfig, mEncodingFormat);
                if (Objects.isNull(mMic) || Objects.isNull(mInternal)) {
                    Log.e(TAG, "create audio record failure");
                    mCallbackHandler.onError(
                            AudioRecorder.this, new IllegalArgumentException());
                    break;
                } else {
                    mMic.startRecording();
                    mInternal.startRecording();
                }

                try {
                    mAudioEncoder.prepare();
                } catch (Exception e) {
                    mCallbackHandler.onError(AudioRecorder.this, e);
                    break;
                }
            case MSG_AUDIO_DEQUEUE_INPUT:
                if (!mForceStop.get()) {
                    int index = getInputBufferIndex(); // get input buffer index from codec
                    if (DEBUG) Log.d(TAG, "audio encoder returned input buffer index=" + index);
                    if (index >= 0) { // Encoder has available input buffer
                        queueAudioEncoderInputBuffer(index); // provide the non-encode data for encoder
                        if (!mForceStop.get()) sendEmptyMessage(MSG_AUDIO_DRAIN_OUTPUT);
                    } else { // Encoder has not available input buffer, try again later...
                        Log.i(TAG, "try later to queue input buffer");
                        sendEmptyMessageDelayed(MSG_AUDIO_DEQUEUE_INPUT, mPollRate);
                    }
                }
                break;
            case MSG_AUDIO_DRAIN_OUTPUT:
                offerOutput();
                signalDequeueInputBuffer();
                break;
            // After processing the data, release the output buffer of the encoder for next use.
            case MSG_AUDIO_RELEASE_OUTPUT:
                mAudioEncoder.releaseOutputBuffer(msg.arg1);
                mMuxOutputBufferIndices.poll(); // Discard processed index records
                if (DEBUG) Log.d(TAG, "audio encoder released output buffer index - "
                        + msg.arg1 + ", remaining=" + mMuxOutputBufferIndices.size());
                signalDequeueInputBuffer();
                break;
            case MSG_AUDIO_STOP:
                if (mMic != null) {
                    mMic.stop();
                }
                if (mInternal != null) {
                    mInternal.stop();
                }
                mAudioEncoder.stop();
                break;
            case MSG_AUDIO_RELEASE:
                if (mMic != null) {
                    mMic.release();
                    mMic = null;
                }
                if (mInternal != null) {
                    mInternal.release();
                    mInternal = null;
                }
                mAudioEncoder.release();
                break;
        }
    }

    private void offerOutput() {
        while (!mForceStop.get()) {
            MediaCodec.BufferInfo cacheBufferInfo = mCachedInfoList.poll();
            if (Objects.isNull(cacheBufferInfo)) {
                cacheBufferInfo = new MediaCodec.BufferInfo();
            }
            // Get output buffer index from encoder
            int index = mAudioEncoder.getEncoder().dequeueOutputBuffer(cacheBufferInfo, 1);
            if (DEBUG) Log.d(TAG, "audio encoder output buffer index - " + index);
            if (index == INFO_OUTPUT_FORMAT_CHANGED) {
                mCallbackHandler.onOutputFormatChanged(
                        mAudioEncoder, mAudioEncoder.getEncoder().getOutputFormat());
            }

            if (index < 0) {
                cacheBufferInfo.set(0, 0, 0, 0);
                mCachedInfoList.offer(cacheBufferInfo);
                break;
            }

            mMuxOutputBufferIndices.offer(index); // Add unprocessed(un-mux) buffer index to queue
            mCallbackHandler.onOutputBufferAvailable(mAudioEncoder, index, cacheBufferInfo);
        }
    }

    private int getInputBufferIndex() {
        return mAudioEncoder.getEncoder().dequeueInputBuffer(0); // Not wait
    }

    private void signalDequeueInputBuffer() {
        if (mMuxOutputBufferIndices.size() <= 1 && !mForceStop.get()) {
            removeMessages(MSG_AUDIO_DEQUEUE_INPUT);
            sendEmptyMessageDelayed(MSG_AUDIO_DEQUEUE_INPUT, 0);
        }
    }
}
```

* 由于产品定义，在录制结束后通知栏需要进行视频缩略图的显示，并且能够点击跳转至视频播放器进行录制文件的播放，因此通过构造如下结构用于保存录制视频的缩略图Bitmap以及存储在MeidaStore中的Uri：

```java
public static class RecordLegacy {
    /**
     * mUri for saving the access Media file Uri.
     */
    private final Uri mUri;
    /**
     * mThumbnailBitmap for saving parsed screen recording video thumbnails.
     */
    private Bitmap mThumbnailBitmap;

    protected RecordLegacy(Uri uri, File file, Size thumbnailSize) {
        mUri = uri;
        try {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                mThumbnailBitmap = ThumbnailUtils.createVideoThumbnail(
                        file, thumbnailSize, null);
            } else {
                mThumbnailBitmap = ThumbnailUtils.createVideoThumbnail(
                        file.getPath(), MediaStore.Images.Thumbnails.FULL_SCREEN_KIND);
            }
        } catch (IOException e) {
            e.printStackTrace();
            Log.e(TAG, "Error creating thumbnail", e);
        }
    }

    public Uri getUri() {
        Log.d(TAG, "getUri: " + mUri);
        return mUri;
    }

    public Bitmap getThumbnailBitmap() {
        return mThumbnailBitmap;
    }

    @Override
    public String toString() {
        return "RecordLegacy{" +
                "mUri=" + mUri +
                ", mThumbnailBitmap=" + mThumbnailBitmap +
                '}';
    }
}
```
