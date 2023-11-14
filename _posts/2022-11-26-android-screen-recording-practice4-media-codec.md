---
date: 2022-11-26 08:00:00
layout: post
title: Android Screen Recording Practice(Part.4) - MediaCodec
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

# Android Screen Recording Practice(Part.4) - MediaCodec

## Sec 1. What is MediaCodec

在前面的文章中，我们了解了音视频的录制方法，这篇文章我们重点说一下编解码。`编码`就是将原始音视频数据流通过处理后变成按一定格式存储的编码音视频数据流；而`解码`则与之相反，由于原始数据的数据量太大，重复数据过多，不利于存储与传输，因此，通常对音视频的首先斗艳经过编码的处理，其实在前几篇文章中已经或多或少提到过编码器了，那就是 - `MediaCodec`。

根据Google官方的定义，MediaCodec是Android平台提供的一种底层音视频编解码框架，是底层多媒体基础结构的一部分，通常与`MediaExtractor`、`MediaSync`、`MediaMuxer`、`AudioTrack`等功能一起使用。

## Sec 2. How MediaCodec works

`MediaCodec`非常经典的诗句处理架构如下图所示，从定义的角度来说，编解码器的作用就是接受输入数据，进行编解码处理后输出数据：

![](https://cdn.jsdelivr.net/gh/jonewan/MD_PNG@master/ScreenRecord/mediacodec_buffers.png)

`MediaCodec`采用了循环buffer的处理方式，在输入和输出端就像对应两个传送带，MediaCodec对buffer的数据通过下标进行管理：
* 由输入方（Client）将需要进行编码的数据填充给从`MediaCodec`获取到的buffer下标中，然后将数据交给`MediaCodec`进行处理，并清空对应下标的输入缓冲区内容，标记为Empty后再次提供给Client用作数据填充；
* 当MediaCodec处理完成数据后会循环将输出数据填充到对应下标的输出缓冲区中提供给输出端进行消费；


压缩数据可以作为解码器的输入数据或者编码器的输出数据，需要指定数据格式，这样编码/解码器才能知道如何处理这些压缩数据。当使用视频时，一般是包含完整的一帧数据，也就是我们要输入给解码器一帧完整的数据或者从编码器得到一帧完整的数据。

`MeidaCodec`能够接受三种数据的输入：`原始音频数据`、`原始视频数据`和`压缩数据`，原始音频数据和原始视频数据可用于音视频编码，压缩数据则用于音视频解码。

### Lifecycle of MediaCodec

MediaCodec同样也是具备自己的状态机的，其通过如下的状态流程进行运作：

![](https://cdn.jsdelivr.net/gh/jonewan/MD_PNG@master/ScreenRecord/mediacodec_states.png)

从总体来看主要存在三种状态：`Stopped`、`Executing`、`Released`，而细分的话，在Executing中又存在`Flushed`、`Running`和`End of Stream`状态，Stopped同样存在`Configured`、`Uninitialized`和`Error`。

* 在MeidaCodec申请创建对象实例时，处于`Uninitialized`状态，此时通过调用`configure()`方法进入`Configured`状态，此时代表编解码器已经准备就绪，此时调用`start()`方法就进入`Excuting:Flushed`状态了，也代表着MeidaCodec已经开始了工作；

* 在`Executing:Flushed`状态下还没有数据，当调用`dequeueInputBuffer()`方法，向MediaCodec申请一个空的缓冲区后，此时MediaCodec进入`Executing:Running`状态，将需要编解码的数据放入对应Buffer中，调用`queueInputBuffer()`将填满待编解码数据的Buffer归还给MediaCodec进行处理；

* 当数据全部处理完时，input端在Buffer中携带一个`End-of-Stream`标志，由output端处理完最后的数据后就进入到`Executing:End of Stream`状态了，此时代表数据全部处理结束，可调用`release()`方法释放MedieCodec的资源后进入`Released`状态；

* 如果在Ecuting的过程中出现了任何错误，则MeidaCodec会停止工作，进入到`Stopped:Error`状态；

* 在Executing过程中，调用stop()/reset()能够重新让MediaCodec重新回到未初始化状态；

## Sec 3. How to use MediaCodec

上面介绍了MeidaCodec的状态流程，那如何进行使用呢？MediaCodec具备两种运行模式，`同步模式`和`异步模式`，MediaCodec在数据处理过程中主要调用的API接口有：

* dequeueInputBuffer：从MediaCodec中获取一个空的输入缓冲区
* queueInputBuffer: 将填满待编解码数据的输入缓冲区归还给MeidaCodec 
* dequeueOutputBuffer：从MediaCodec中获取一个编解码后的输出缓冲区
* releaseOutputBuffer：归还给MeidaCodec输出缓冲区

### Synchronize Mode

以同步模式为例，按上面提到的状态机模型对MIC与REMOTE_SUBMIX通道同时录制并编码：

* 创建两路音频的录制线程：

```java
private Thread createRecordThreadWork(int shortBufferSize, int offsetShortsInternal,
                                      int offsetShortsMic) {
    return new Thread(() -> {
        short[] bufferInternal = new short[shortBufferSize];
        short[] bufferMic = new short[shortBufferSize];
        byte[] bufferByteOut = new byte[shortBufferSize * 2]; // one byte array equals double short array length

        int readBytes = 0;
        int readShortsInternal = 0;
        int audioInternalOffset = offsetShortsInternal;
        int readShortsMic = 0;
        int audioMicOffset = offsetShortsMic;
        while (true) {
            readShortsInternal = mAudioRecordInternal.read(bufferInternal, audioInternalOffset,
                    bufferInternal.length - audioInternalOffset);
            readShortsMic = mAudioRecordMic.read(
                    bufferMic, audioMicOffset, bufferMic.length - audioMicOffset);

            // if both error, end the recording
            if (readShortsInternal < 0 && readShortsMic < 0) {
                break;
            }

            // if internal audio record channel has an errors, fill its bufferByteOut with zeros
            // and assume it is mute with the same recordBufferSize as the other bufferByteOut.
            if (readShortsInternal < 0) {
                readShortsInternal = readShortsMic;
                audioInternalOffset = audioMicOffset;
                java.util.Arrays.fill(bufferInternal, (short) 0);
            }

            // if mic audio record channel has an errors or silence by user, fill its
            // bufferByteOut with zeros and assume it is mute with the same recordBufferSize
            // as the other bufferByteOut.
            if (readShortsMic < 0 || mMicSilence) {
                readShortsMic = readShortsInternal;
                audioMicOffset = audioInternalOffset;
                java.util.Arrays.fill(bufferMic, (short) 0);
            }

            // Add offset (previous unmixed values) to the bufferByteOut
            readShortsInternal += audioInternalOffset;
            readShortsMic += audioMicOffset;

            int minShorts = Math.min(readShortsInternal, readShortsMic);
            readBytes = minShorts * 2;

            bufferVolumeScale(bufferMic, minShorts, mRecordParams.micVolumeScale);
            bufferVolumeScale(bufferInternal, minShorts, mRecordParams.systemVolumeScale);

            // Mix system internal bufferByteOut and Mic bufferByteOut to outBuffer
            addAndConvertBuffers(bufferInternal, bufferMic, bufferByteOut, minShorts);

            // shift unmixed shorts to the beginning of the bufferByteOut
            shiftToStart(bufferInternal, minShorts, audioInternalOffset);
            shiftToStart(bufferMic, minShorts, audioMicOffset);

            // reset the offset for the next loop
            audioInternalOffset = readShortsInternal - minShorts;
            audioMicOffset = readShortsMic - minShorts;

            if (readBytes < 0) {
                Log.e(TAG, "read error " + readBytes + ", shorts internal: " + readShortsInternal +
                        ", shorts mic: " + readShortsMic);
                break;
            }
            encode(bufferByteOut, readBytes);
        }
        endStream();
    });
}
```

* 对音频数据进行编码

```java
private void encode(byte[] buffer, int readBytes) {
    int offset = 0;
    while (readBytes > 0) {
        int totalBytesRead = 0;
        int bufferIndex = mMediaCodec.dequeueInputBuffer(TIMEOUT_US);
        if (bufferIndex < 0) {
            writeOutput();
            return;
        }
        ByteBuffer buff = mMediaCodec.getInputBuffer(bufferIndex);
        buff.clear();
        int bufferSize = buff.capacity();
        int bytesToRead = Math.min(readBytes, bufferSize);
        totalBytesRead += bytesToRead;
        readBytes -= bytesToRead;
        buff.put(buffer, offset, bytesToRead);
        offset += bytesToRead;
        mMediaCodec.queueInputBuffer(bufferIndex, 0, bytesToRead, mPresentationTime, 0);
        mTotalBytes += totalBytesRead;
        mPresentationTime = 1000000L * (mTotalBytes / 2) / mRecordParams.audioSampleRate;

        writeOutput();
    }
}
```

* End-of-Stream状态：

```java
private void endStream() {
    int bufferIndex = mMediaCodec.dequeueInputBuffer(TIMEOUT_US);
    mMediaCodec.queueInputBuffer(bufferIndex, 0, 0, mPresentationTime,
            MediaCodec.BUFFER_FLAG_END_OF_STREAM);
    writeOutput();
}
```

* 处理编码完成的数据，将编码后的数据通过MediaMuxer写入文件中：

```java
private void writeOutput() {
    while (true) {
        MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();
        int bufferIndex = mMediaCodec.dequeueOutputBuffer(bufferInfo, TIMEOUT_US);
        if (bufferIndex == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
            mTrackId = mMediaMuxer.addTrack(mMediaCodec.getOutputFormat());
            mMediaMuxer.start();
            continue;
        }
        if (bufferIndex == MediaCodec.INFO_TRY_AGAIN_LATER) {
            break;
        }
        if (mTrackId < 0) return;
        // 获取编码完成后的输出缓冲区
        ByteBuffer buff = mMediaCodec.getOutputBuffer(bufferIndex);

        if (!((bufferInfo.flags & MediaCodec.BUFFER_FLAG_CODEC_CONFIG) != 0
                && bufferInfo.size != 0)) {
            // 写入文件
            mMediaMuxer.writeSampleData(mTrackId, buff, bufferInfo);
        }
        // 释放输出数据缓冲区
        mMediaCodec.releaseOutputBuffer(bufferIndex, false);
    }
}
```

可以看到，在同步模式下，主要通过while的循环嵌套进行数据处理，需要等上个操作结束后才能执行下一个操作。

### Asynchronous Mode

MediaCodec的异步模式主要通过配置回掉方法进行实现，同样以两路音频录制为例：

```java
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
```

* 配置音频编码器的Callback：

```java
private void prepareAudioEncoder() throws IOException {
    final AudioRecorder audioRecorder = mAudioEncoder;
    if (Objects.isNull(audioRecorder)) {
        Log.w(TAG, "<prepareAudioEncoder> Audio encoder object not applied.");
        return;
    }

    AudioEncoder.Callback callback = new AudioEncoder.Callback() {
        @Override
        public void onOutputBufferAvailable(BaseMediaEncoder codec, int index,
                                            MediaCodec.BufferInfo info) {
            if (DEBUG) Log.i(TAG, "[" + Thread.currentThread().getId()
                    + "] AudioEncoder output buffer available: index=" + index);
            try {
                muxAudio(index, info);
            } catch (Exception e) {
                Log.e(TAG, "Muxer encountered an error! ", e);
                Message.obtain(mHandler, MSG_MEDIA_ERROR, e).sendToTarget();
            }
        }

        @Override
        public void onOutputFormatChanged(BaseMediaEncoder codec, MediaFormat format) {
            if (DEBUG) Log.d(TAG, "[" + Thread.currentThread().getId()
                    + "] AudioEncoder returned new format " + format);
            resetAudioOutputFormat(format);
            startMuxerIfReady();
        }

        @Override
        public void onError(IEncoder encoder, Exception e) {
            Log.e(TAG, "Audio encoder occurred an error!", e);
            Message.obtain(mHandler, MSG_MEDIA_ERROR, e).sendToTarget();
        }
    };
    audioRecorder.setCallback(callback);
    audioRecorder.prepare();
}
```

* 利用Handler机制处理音频录制的逻辑状态跳转：

```java
private class RecordHandler extends Handler {
    ......
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_AUDIO_PREPARE:
                mMic = createAudioRecord(MediaRecorder.AudioSource.MIC,
                        mSampleRate, mChannelConfig, mEncodingFormat);
                mInternal = createAudioRecord(MediaRecorder.AudioSource.REMOTE_SUBMIX,
                        mSampleRate, mChannelConfig, mEncodingFormat);
                mMic.startRecording();
                mInternal.startRecording();

                try {
                    mAudioEncoder.prepare();
                } catch (Exception e) {
                    mCallbackHandler.onError(AudioRecorder.this, e);
                    break;
                }
            case MSG_AUDIO_DEQUEUE_INPUT:
                if (!mForceStop.get()) {
                    int index = getInputBufferIndex(); // get input buffer index from codec
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

* 提供给MediaCodec未编码的音频数据：

```
private void queueAudioEncoderInputBuffer(int index) {
    if (index < 0 || mForceStop.get()) return;
    final AudioRecord micRecord = Objects.requireNonNull(mMic,
            "Mic record might be released.");
    final AudioRecord internalRecord = Objects.requireNonNull(mInternal,
            "Internal record might be released.");
    final boolean endOfStream = micRecord.getRecordingState() == AudioRecord.RECORDSTATE_STOPPED
            || internalRecord.getRecordingState() == AudioRecord.RECORDSTATE_STOPPED;
    final ByteBuffer frameByteBuffer = mAudioEncoder.getInputBuffer(index);
    int offset = frameByteBuffer.position();
    int limit = frameByteBuffer.limit();
    final int shortBufferSize = limit <= 0 ? 0 : limit / 2;
    short[] bufferInternal = new short[shortBufferSize];
    short[] bufferMic = new short[shortBufferSize];
    byte[] bufferByteOut = new byte[limit]; // one byte array equals double short array length
    int readShortsInternal = 0;
    int readShortsMic = 0;
    int readBytes = 0;
    if (!endOfStream) {
        readShortsInternal = internalRecord.read(bufferInternal, 0, bufferInternal.length);
        readShortsMic = micRecord.read(bufferMic, 0, bufferMic.length);

        if (readShortsInternal < 0) {
            readShortsInternal = readShortsMic;
            java.util.Arrays.fill(bufferInternal, (short) 0);
        }

        if (readShortsMic < 0 || mMicSilence.get()) {
            readShortsMic = readShortsInternal;
            java.util.Arrays.fill(bufferMic, (short) 0);
        }

        int minShorts = Math.min(readShortsInternal, readShortsMic);
        readBytes = minShorts * 2;

        // Adjust volume scaling for respective sound channels
        bufferVolumeScale(bufferMic, minShorts, VOLUME_SCALE_NO_SCALE);
        bufferVolumeScale(bufferInternal, minShorts, VOLUME_SCALE_80_TIMES);
        // Mix system internal bufferByteOut and Mic bufferByteOut to outBuffer
        addAndConvertBuffers(bufferInternal, bufferMic, bufferByteOut, minShorts);

        // shift unmixed shorts to the beginning of the bufferByteOut
        shiftToStart(bufferInternal, minShorts, 0);
        shiftToStart(bufferMic, minShorts, 0);
        frameByteBuffer.clear(); // Clear dirty buffer cache
        frameByteBuffer.put(bufferByteOut);

        readBytes = Math.max(readBytes, 0);
    }

    int totalBits = readBytes << 3; // calculate bit count for read bytes (readBytes * 8bits)
    long presentationTimeUs = calculateFrameTimestamp(totalBits);
    int flags = endOfStream ? BUFFER_FLAG_END_OF_STREAM : BUFFER_FLAG_KEY_FRAME;

    if (DEBUG) {
        Log.d(TAG, "Feed codec index=" + index + ", presentationTimeUs="
                + presentationTimeUs + ", flags=" + flags);
    }
    // provide frame buffer to encoder
    mAudioEncoder.queueInputBuffer(index, offset, readBytes, presentationTimeUs, flags);
}
```

* 音频输出到文件：

```java
private void muxAudio(int index, MediaCodec.BufferInfo bufferInfo) {
    if (!mIsRunning.get()) {
        Log.w(TAG, "<muxAudio> Already stopped!");
        return;
    }

    // If MediaMuxer not ready or audio track doesn't add into muxer,
    // put unprocessed output buffer index to pending index list, wait
    // for operate next time.
    if (!mMuxerStarted.get() || mAudioTrackIndex == INVALID_INDEX) {
        mPendingAudioEncoderBufferIndexList.add(index);
        mPendingAudioEncoderBufferInfoList.add(bufferInfo);
        return;
    }

    // Get output buffer encoded data and write to media output file
    ByteBuffer encodedData = mAudioEncoder.getOutputBuffer(index);
    writeSampleData(mAudioTrackIndex, bufferInfo, encodedData);
    mAudioEncoder.releaseOutputBuffer(index);

    // current encoded buffer with end-of-stream flag, signal to stop record
    if ((bufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
        if (DEBUG)
            Log.d(TAG, "Stop encoder and muxer, since the buffer has been marked with EOS");
        mAudioTrackIndex = INVALID_INDEX;
        signalStop(true);
    }
}
```
