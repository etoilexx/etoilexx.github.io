---
date: 2022-11-24 08:00:00
layout: post
title: Android Screen Recording Practice(Part.2) - Record Audio
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

# Android Screen Recording Practice(Part.2) - Record Audio

[TOC]

## Sec.1 How to capture audio on Android device

根据Google官方描述，Android 系统的音频架构如如所示：

![](https://source.android.com/static/docs/core/audio/images/ape_fwk_audio.png?hl=zh-cn)

应用层通过调用framework层提供的`android.media.*`API方法，能够与音频硬件进行交互，通过JNI后调用到`frameworks/av/libmeida`中，`libmedia`所在的Native层中的`AudioRecord`即负责音频流的捕获、`AudioTrack`负责音频流播放，通过Binder调用与`MediaServer`的服务进行交互，其中`AudioPolicy Service`负责系统的音频策略，`AudioFlinger Service`负责音频合成，通过HAL层接口调用至Linux kernel操作具体的ALSA设备驱动。

## Sec.2 What is AudioRecord

[AudioRecord](https://developer.android.com/reference/android/media/AudioRecord.html)是frwmework层提供的用于管理音频输入硬件资源的音频录制类。AudioRecord捕获的音频数据为未经过编码的PCM裸流。

其中重要的接口方法与定义如下：

* `framework/base/media/java/android/media/AudioRecord.java`

```java
public class AudioRecord implements AudioRouting, MicrophoneDirection,
        AudioRecordingMonitor, AudioRecordingMonitorClient
{

    /**
    * @param audioSource ：音频源，具体定义在MediaRecorder.AudioSource中
    * @param sampleRateInHz：音频采样率
    * @param channelConfig：音频声道数配置
    * @param audioFormat：录音采样位数，在AudioFormat中有定义
    * @param bufferSizeInBytes：音频捕获的的缓冲区大小
    */
    public AudioRecord(int audioSource, int sampleRateInHz, int channelConfig, int audioFormat,
            int bufferSizeInBytes) throws IllegalArgumentException {
        this((new AudioAttributes.Builder())
            .setInternalCapturePreset(audioSource)
            .build(),
        (new AudioFormat.Builder())
            .setChannelMask(getChannelMaskFromLegacyConfig(channelConfig,
                                true/*allow legacy configurations*/))
            .setEncoding(audioFormat)
            .setSampleRate(sampleRateInHz)
            .build(),
        bufferSizeInBytes,
        AudioManager.AUDIO_SESSION_ID_GENERATE);
    }
    
    ......
    
    /*
    * 获取AudioRecord录制Buffer的最小长度
    **/
    static public int getMinBufferSize(int sampleRateInHz, int channelConfig, int audioFormat) {
        int channelCount = 0;
        switch (channelConfig) {
        case AudioFormat.CHANNEL_IN_DEFAULT: // AudioFormat.CHANNEL_CONFIGURATION_DEFAULT
        case AudioFormat.CHANNEL_IN_MONO:
        case AudioFormat.CHANNEL_CONFIGURATION_MONO:
            channelCount = 1;
            break;
        case AudioFormat.CHANNEL_IN_STEREO:
        case AudioFormat.CHANNEL_CONFIGURATION_STEREO:
        case (AudioFormat.CHANNEL_IN_FRONT | AudioFormat.CHANNEL_IN_BACK):
            channelCount = 2;
            break;
        case AudioFormat.CHANNEL_INVALID:
        default:
            loge("getMinBufferSize(): Invalid channel configuration.");
            return ERROR_BAD_VALUE;
        }

        // 根据采样率、通道数以及相关配置通过Native层接口获取大小
        int size = native_get_min_buff_size(sampleRateInHz, channelCount, audioFormat);
        if (size == 0) {
            return ERROR_BAD_VALUE;
        }
        else if (size == -1) {
            return ERROR;
        }
        else {
            return size;
        }
    }    
    
    ......
    
    public void startRecording()
    throws IllegalStateException {
        if (mState != STATE_INITIALIZED) {
            throw new IllegalStateException("startRecording() called on an "
                    + "uninitialized AudioRecord.");
        }

        // start recording
        synchronized(mRecordingStateLock) {
            if (native_start(MediaSyncEvent.SYNC_EVENT_NONE, 0) == SUCCESS) {
                handleFullVolumeRec(true);
                mRecordingState = RECORDSTATE_RECORDING;
            }
        }
    }
    
    // 以byte数组buffer进行PCM数据的读取
    public int read(@NonNull byte[] audioData, int offsetInBytes, int sizeInBytes) {
        return read(audioData, offsetInBytes, sizeInBytes, READ_BLOCKING);
    }
    
    // 以short数组buffer进行PCM数据读取
    public int read(@NonNull short[] audioData, int offsetInShorts, int sizeInShorts) {
        return read(audioData, offsetInShorts, sizeInShorts, READ_BLOCKING);
    }
    
    /**
     * Stops recording.
     * @throws IllegalStateException
     */
    public void stop()
    throws IllegalStateException {
        if (mState != STATE_INITIALIZED) {
            throw new IllegalStateException("stop() called on an uninitialized AudioRecord.");
        }

        // stop recording
        synchronized(mRecordingStateLock) {
            handleFullVolumeRec(false);
            native_stop();
            mRecordingState = RECORDSTATE_STOPPED;
        }
    }
}
```

## Sec 3. Audio source of Record

上面提到，在应用层创建AudioRecord时需要制定audioSource，具体在`MediaRecorder.AudioSource`中有定义：

```java
public final class AudioSource {

        private AudioSource() {}

        /** @hide */
        public final static int AUDIO_SOURCE_INVALID = -1;

        /** Default audio source **/
        public static final int DEFAULT = 0;

        /** Microphone audio source */
        public static final int MIC = 1;

        /** 语音通话上行通道
         */
        public static final int VOICE_UPLINK = 2;

        /** 语音通话下行通道
         */
        public static final int VOICE_DOWNLINK = 3;

        /** 语音通话通道（包括上行+下行）
         */
        public static final int VOICE_CALL = 4;

        /** 用于视频录制的麦克风音频源，如果可用，其方向与相机相同 */
        public static final int CAMCORDER = 5;

        /** 用于语音识别的麦克风音频源 */
        public static final int VOICE_RECOGNITION = 6;

        /** 用于语音通信（如 VoIP）的麦克风音频源
         */
        public static final int VOICE_COMMUNICATION = 7;

        /**
         * 远端混音通道，该通道用于向远程接收器（例如 Wifi 显示器）输出混合音频流。
         */
        @RequiresPermission(android.Manifest.permission.CAPTURE_AUDIO_OUTPUT)
        public static final int REMOTE_SUBMIX = 8;

        /** 未处理（原始）声音调整的麦克风音频源 */
        public static final int UNPROCESSED = 9;
        ......
    }
}
```

因此，若要进行麦克风+系统声音的录制，可以选用`MediaRecorder.AudioSource.MIC`和`MediaRecorder.AudioSource.REMOTE_SUBMIX`通道进行录制。需要注意的是`REMOTE_SUBMIX`通道需要具备`android.Manifest.permission.CAPTURE_AUDIO_OUTPUT`权限，而该权限在`AndroidManifest.xml`中的定义如下：

```xml
<permission android:name="android.Manifest.permission.CAPTURE_AUDIO_OUTPUT"
    android:protectionLevel="signature|privileged" />
```

这表明该权限不支持三方应用获取，仅允许被系统签名的应用（不一定非得UID）才能够录制`REMOTE_SUBMIX`通道数据。

## Sec 4. How to record system internal Audio And MicroPhone

由于涉及到双路音频（MIC和REMOTE_SUBMIX）的录制，因此需要使用两路AudioRecord进行，整体的录制方案架构如下图所示：

![media_projection_dialog](https://cdn.jsdelivr.net/gh/jonewan/MD_PNG@master/ScreenRecord/audio_record_process_v1.png)

由于AudioRecord录制出来的音频流为未经编码的PMC数据，因此需要配置MediaCodec编码器（后续对MeidaCodec进行详细解释）为AAC编码，申请两路AudioRecord后，将数据源源不断的送给MediaCodec编码器的输入数据缓冲区，之后从编码器中读出对应的数据流，通过MediaMuxer写入音频录制的结果文件中。具体代码逻辑如下：

* 音频录制配置：

```java
public ScreenRecordParams() {
    this.audioSource = AUDIO_SOURCE_MIC_INTERNAL; // 同时录制MIC与系统声音
    this.audioSampleRate = AUDIO_SAMPLE_RATE_48KHz;// 采样率使用48KHz
    this.audioEncodeFormat = AUDIO_ENCODE_16BIT; //采用16bit PCM编码
    this.audioChannelInMask = AUDIO_CHANNEL_IN_MONO; // 单声道录制
    this.micVolumeScale = VOLUME_SCALE_NO_SCALE;
    this.systemVolumeScale = VOLUME_SCALE_80_TIMES;
    ......
}
```

* 音频录制环境资源申请与配置：

```java
private void setupRecordConfig() throws IOException {
    if (!mInternal && !mMic) {
        Log.w(TAG, "Not recording audio, do not setup!");
        return;
    }

    final int recordBufferSize = AudioRecord.getMinBufferSize(
            mRecordParams.audioSampleRate, mRecordParams.audioChannelInMask,
            mRecordParams.audioEncodeFormat);
    Log.d(TAG, "audio buffer recordBufferSize: " + recordBufferSize);

    mAudioRecordInternal = new AudioRecord(MediaRecorder.AudioSource.REMOTE_SUBMIX,
            mRecordParams.audioSampleRate, mRecordParams.audioChannelInMask,
            mRecordParams.audioEncodeFormat, recordBufferSize);
    if (mAudioRecordInternal.getRecordingState() != AudioRecord.RECORDSTATE_STOPPED) {
        String expMsg = "Failed to setup record: Audio source REMOTE_SUBMIX was occupied!";
        Log.e(TAG, expMsg);
        throw new RuntimeException(expMsg);
    }
    
    mAudioRecordMic = new AudioRecord(MediaRecorder.AudioSource.MIC,
            mRecordParams.audioSampleRate, mRecordParams.audioChannelInMask,
            mRecordParams.audioEncodeFormat, recordBufferSize);
    if (mAudioRecordMic.getRecordingState() != AudioRecord.RECORDSTATE_STOPPED) {
        String expMsg = "Setup record failed: Audio source MIC was occupied!";
        Log.e(TAG, expMsg);
        throw new RuntimeException(expMsg);
    }

    mMediaCodec = MediaCodec.createEncoderByType(MediaFormat.MIMETYPE_AUDIO_AAC);
    MediaFormat mediaFormat = MediaFormat.createAudioFormat(
            MediaFormat.MIMETYPE_AUDIO_AAC, mRecordParams.audioSampleRate, 1);
    mediaFormat.setInteger(MediaFormat.KEY_AAC_PROFILE,
            MediaCodecInfo.CodecProfileLevel.AACObjectLC);
    mediaFormat.setInteger(MediaFormat.KEY_BIT_RATE, 196000);
    mediaFormat.setInteger(MediaFormat.KEY_PCM_ENCODING, mRecordParams.audioEncodeFormat);
    mMediaCodec.configure(mediaFormat, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);
    mMediaMuxer = new MediaMuxer(outFile, MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4);

    mRecordingThread = createRecordThreadWork(recordBufferSize, 0, 0);
}
```

* 开启线程从AudioRecord读取MIC与系统的音频数据进行处理：

由于在定义AudioRecord PCM编码参数时选用了16bit编码，因此一个PCM数据为2Byte，故，在从AudioRecord读取数据时选用short数组进行回读，而在进行编码是使用byte数组进行编码，因此byte数组缓冲区的长度需要为short数组的2倍：

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

由于采用了两路AudioRecord读取，因此需要保证两路数据同步，通过AudioRecord的read方法能够获取到此次实际读取的buffer长度，需要保证每次读取和合成的buffer长度是一致的，因此通过定义两个偏移量`audioMicOffset`和`audioInternalOffset`来记录两路数据的偏移，每次写入两路数据读取的最小长度，在下次将剩余的数据再行写入：

```java
/**
 * moves all bits from start to end to the beginning of the array
 */
private void shiftToStart(short[] target, int start, int end) {
    if (end - start >= 0) System.arraycopy(target, start, target, 0, end - start);
}
```

PCM数据实际标识了音频数据的振幅大小，因此能够通过对PCM编码数据进行倍数达到增加/减小音量的目的：

```java
private void bufferVolumeScale(short[] buff, int len, float scale) {
    for (int i = 0; i < len; i++) {
        int newValue = (int) (buff[i] * scale);
        buff[i] = (short) MathUtils.constrain(newValue, Short.MIN_VALUE, Short.MAX_VALUE);
    }
}
```

其中，由于使用了两路AudioRecord进行音频录制，因此存在音频合成的过程，AudioRecord获取到的数据是PCM数据，因此能够通过叠加的方式进行音频数据合成，将MIC通道数据与REMOTE_SUBMIX通道数据叠加后按字节写会byte[]的buffer中：

```java
private void addAndConvertBuffers(short[] src1, short[] src2, byte[] dst, int sizeShorts) {
    int sum = 0, byteIndex = 0;
    for (int i = 0; i < sizeShorts; i++) {
        sum = (short) MathUtils.constrain(
                (int) src1[i] + (int) src2[i], Short.MIN_VALUE, Short.MAX_VALUE);
        byteIndex = i * 2;
        // Use Little-endian to storage the audio buffer data
        dst[byteIndex] = (byte) (sum & 0xff);
        dst[byteIndex + 1] = (byte) ((sum >> 8) & 0xff);
    }
}
```

* 将数据送入MediaCodec编码器编码：

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

* 获取编码后的AAC音频数据，通过MediaMuxer写入结果文件：

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
        ByteBuffer buff = mMediaCodec.getOutputBuffer(bufferIndex);

        if (!((bufferInfo.flags & MediaCodec.BUFFER_FLAG_CODEC_CONFIG) != 0
                && bufferInfo.size != 0)) {
            mMediaMuxer.writeSampleData(mTrackId, buff, bufferInfo);
        }
        mMediaCodec.releaseOutputBuffer(bufferIndex, false);
    }
}
```

## Sec 5. Issues need resolving

### REMOTE_SUBMIX channel blocked

由于AudioPolicy的限制，当对`REMOTE_SUBMIX`通道进行录制时，会截流原本传输至扬声器与耳机的音频流，导致较差的用户体验。

解决该问题的方式就是，当进行REMOTE_SUBMIX通道的录制时不进行截流限制，将数据同时传输给远端与本地播放设备（扬声器、耳机），具体修改如下：

```diff
diff --git a/services/audiopolicy/enginedefault/src/Engine.cpp b/services/audiopolicy/enginedefault/src/Engine.cpp
index b8a089913..5c5f8e2db 100755
--- a/services/audiopolicy/enginedefault/src/Engine.cpp
+++ b/services/audiopolicy/enginedefault/src/Engine.cpp
@@ -419,6 +419,13 @@ audio_devices_t Engine::getDeviceForStrategyInt(legacy_strategy strategy,
             if (availableOutputDevices.getDevice(AUDIO_DEVICE_OUT_REMOTE_SUBMIX,
                                                  String8("0"), AUDIO_FORMAT_DEFAULT) != 0) {
                 device2 = availableOutputDevices.types() & AUDIO_DEVICE_OUT_REMOTE_SUBMIX;
+                device2 |= (availableOutputDevices.types() & AUDIO_DEVICE_OUT_WIRED_HEADSET);
+                device2 &= ~(availableOutputDevices.types() & AUDIO_DEVICE_OUT_SPEAKER);
+                if((availableOutputDevices.types() & AUDIO_DEVICE_OUT_WIRED_HEADSET) == 0){
+                    device2 |= (availableOutputDevices.types() & AUDIO_DEVICE_OUT_SPEAKER);
+                }
             }
         }
         if (isInCall() && (strategy == STRATEGY_MEDIA)) {
```

### How to resolve microphone recording conflicts

在低于AndroidQ的版本中能够通过判断AudioRecord的通道有效性判断当前通道是否可以被录制，不允许多个进程对MIC通道的同时录制，故而方便应用端进行判断；而在AndroidQ中引入了共享音频策略，在不再强制禁止多个进程同时使用同一通道，经过对AndroidQ中AudioPolicy深入梳理分析后发现，应用层受限的音频策略如下：
- 当两个普通应用进程同时进行通道捕获时，只有一个应用接收音频，另一个应用会受到静默处理。
- 如果两个应用都不具备隐私敏感性，则由界面位于顶部的应用接收音频。如果两个应用都没有界面，则较晚开始者接收音频。
- 如果其中一个应用具备隐私敏感性，则由其接收音频，另一个应用则会受到静默处理，即使后者由界面位于顶部或较晚开始捕获也是如此。如果两个应用都具备隐私敏感性，则由最晚开始捕获的应用接收音频，另一个应用则会受到静默处理。

具体的逻辑代码存在于`frameworks/av/services/audiopolicy/service/AudioPolicyService.cpp`：

```cpp
void AudioPolicyService::updateUidStates_l()
{
    ......
        // By default allow capture if:
        //     The assistant is not on TOP
        //     AND is on TOP or latest started
        //     AND there is no active privacy sensitive capture or call
        //             OR client has CAPTURE_AUDIO_OUTPUT privileged permission
        bool allowCapture = !isAssistantOnTop
                && (isTopOrLatestActive || isTopOrLatestSensitive)
                && !(isSensitiveActive
                    && !(isTopOrLatestSensitive || current->canCaptureOutput))
                && canCaptureIfInCallOrCommunication(current);

        if (isVirtualSource(source)) {
            // Allow capture for virtual (remote submix, call audio TX or RX...) sources
            allowCapture = true;
        } else if (mUidPolicy->isAssistantUid(current->uid)) {
            // For assistant allow capture if:
            //     An accessibility service is on TOP or a RTT call is active
            //            AND the source is VOICE_RECOGNITION or HOTWORD
            //     OR is on TOP AND uses VOICE_RECOGNITION
            //            OR uses HOTWORD
            //         AND there is no active privacy sensitive capture or call
            //             OR client has CAPTURE_AUDIO_OUTPUT privileged permission
            if (isA11yOnTop || rttCallActive) {
                if (source == AUDIO_SOURCE_HOTWORD || source == AUDIO_SOURCE_VOICE_RECOGNITION) {
                    allowCapture = true;
                }
            } else {
                if (((isAssistantOnTop && source == AUDIO_SOURCE_VOICE_RECOGNITION) ||
                        source == AUDIO_SOURCE_HOTWORD)
                        && !(isSensitiveActive && !current->canCaptureOutput)
                        && canCaptureIfInCallOrCommunication(current)) {
                    allowCapture = true;
                }
            }
            ......
        }
        setAppState_l(current->portId,
                      allowCapture ? apmStatFromAmState(mUidPolicy->getUidState(current->uid)) :
                                APP_STATE_IDLE);
    }
}

void AudioPolicyService::setAppState_l(audio_port_handle_t portId, app_state_t state)
{
    AutoCallerClear acc;

    if (mAudioPolicyManager) {
        mAudioPolicyManager->setAppState(portId, state);
    }
    sp<IAudioFlinger> af = AudioSystem::get_audio_flinger();
    if (af) {
        bool silenced = state == APP_STATE_IDLE;
        af->setRecordSilenced(portId, silenced);
    }
}
```

可以看到，当不符合对应的策略时会调用AudioFlinger将对应的进程静默掉，如果不想被静默则增加如下修改解决：

```diff
diff --git a/services/audiopolicy/service/AudioPolicyService.cpp b/services/audiopolicy/service/AudioPolicyService.cpp
index eb1f85df4..31aa9a805 100644
--- a/services/audiopolicy/service/AudioPolicyService.cpp
+++ b/services/audiopolicy/service/AudioPolicyService.cpp
@@ -646,6 +646,9 @@ bool AudioPolicyService::isVirtualSource(audio_source_t source)
         case AUDIO_SOURCE_VOICE_CALL:
         case AUDIO_SOURCE_REMOTE_SUBMIX:
         case AUDIO_SOURCE_FM_TUNER:
+        case AUDIO_SOURCE_MIC:
             return true;
         default:
             break;
```
