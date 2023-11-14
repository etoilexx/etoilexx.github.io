---
date: 2022-11-23 16:30:00
layout: post
title: Android Screen Recording Practice(Part.1) - MediaProjection
subtitle: Andorid屏幕录制实践
description: Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
image: https://res.cloudinary.com/dm7h7e8xj/image/upload/v1559825145/theme16_o0seet.jpg
optimized_image: https://res.cloudinary.com/dm7h7e8xj/image/upload/c_scale,w_380/v1559825145/theme16_o0seet.jpg
category: Andorid System
tags:
  - Android System
author: etoile_xx
---

## What is MediaProjection

这个问题很好，根据字面意思Media（媒体）Projection（投影）说明这个东西是用来处理媒体投影的，但实际上它只是一个令牌（许可证），Google官方文档中的定义如下：

> The MediaProjection API allows apps to acquire a MediaProjection token that gives them one time access to capture screen contents or audio. The Android OS asks the user for their permission before granting the token to your app.
> MediaProjection API 允许应用程序获取一个 MediaProjection 令牌，该令牌使他们可以一次性访问捕获屏幕内容或音频。在将令牌授予您的应用之前，Android 操作系统会询问用户的许可。

## How to Acquire MediaProjection

根据Android Developer官方以及各种资料中提到的方式，通常获取MediaProjection的常见步骤如下：

```java
@Override
void onCreate() {
......
    // 获取MediaProjectionManager
    MediaProjectionManager mediaProjectionManager =
                    (MediaProjectionManager) getSystemService(Context.MEDIA_PROJECTION_SERVICE);
    // 获取屏幕捕获权限弹窗的Intent
    Intent mediaProjPermissionIntent = mediaProjectionManager.createScreenCaptureIntent();
    startActivityForResult(mediaProjPermissionIntent, REQUEST_MEDIA_PROJECTION); // 跳转至屏幕捕获权限申请弹窗
......
}

// 在onActivityResult中获取MediaProjection的实例
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    ......
    mediaProjection = mediaProjectionManager.getMediaProjection(resultCode, resultData);
    ......
}
```

通过`MediaProjectionManager`创建一个专门用于获取`MediaProjection的Intent`，之后通过`startActivityForResult()`方法尝试开启这个`Intent`，之后在`onActivityResult()`回掉方法中通过`MediaProjectionManager`根据传回来的`resultCode`以及`resultData`获取到`MediaProjection`的实例，如果没有错误的情况下就得到了MediaProjection这个令牌了，那到底原理是怎么样的呢？我们来分析一下。

### The Process of Acquiring MediaProjection

![](//www.plantuml.com/plantuml/png/TP1DQiGm38NtFeMN6QRf0R8eOrmAMGH2JGyW4Z6un3-rvSBSldPSqsPeLnRM9xttT8a4WPBPcC-lNhG7IrnuBpUDkOB8XJpq9bmrpC6zuqBQsGaiN34KS9gd0wbkaT2yZQMd61EIz_rJeSxdWIy1WL3b1wq4JodWQ0ajTIUMaHqLikyBQvhaZC7e6BDiPTjELRazYyYsFZNTNbnunPWswluTV-01Fn96acx54rDrry37lpC-Liy__rsRQKQTRUhVqgTVgIjoHRhRteMTBHwAeL8_0y7n3CFztTa5xPhCzWK0)

在SystemServer中由MediaProjectionManager暴漏给应用层接口进行管理MediaProjection的获取操作：

* `frameworks/base/media/java/android/media/projection/MediaProjectionManager.java`
```java
public final class MediaProjectionManager {

    /** @hide */
    public static final int TYPE_SCREEN_CAPTURE = 0;
    /** @hide */
    public static final int TYPE_MIRRORING = 1;
    /** @hide */
    public static final int TYPE_PRESENTATION = 2;

    private Context mContext;
    private Map<Callback, CallbackDelegate> mCallbacks;
    private IMediaProjectionManager mService;

    /** @hide */
    public MediaProjectionManager(Context context) {
        mContext = context;
        IBinder b = ServiceManager.getService(Context.MEDIA_PROJECTION_SERVICE);
        mService = IMediaProjectionManager.Stub.asInterface(b);
        mCallbacks = new ArrayMap<>();
    }

    public Intent createScreenCaptureIntent() {
        Intent i = new Intent();
        final ComponentName mediaProjectionPermissionDialogComponent =
                ComponentName.unflattenFromString(mContext.getResources().getString(
                        com.android.internal.R.string
                        .config_mediaProjectionPermissionDialogComponent));
        i.setComponent(mediaProjectionPermissionDialogComponent);
        return i;
    }

    public MediaProjection getMediaProjection(int resultCode, @NonNull Intent resultData) {
        if (resultCode != Activity.RESULT_OK || resultData == null) {
            return null;
        }
        IBinder projection = resultData.getIBinderExtra(EXTRA_MEDIA_PROJECTION);
        if (projection == null) {
            return null;
        }
        return new MediaProjection(mContext, IMediaProjection.Stub.asInterface(projection));
    }
    ......
}
```

在MediaProjectionManager中维护了一个`IMediaProjectionManager`接口的mService代理，在通过`createScreenCaptureIntent()`方法创建Intent时实际上是根据Framework Resource中定义的`config_mediaProjectionPermissionDialogComponent`数据来配置Intent的Component，经过在`frameworks/res/res/values/config.xml`中查找，默认配置的值为：

```xml
<string name="config_mediaProjectionPermissionDialogComponent" translatable="false">
    com.android.systemui/com.android.systemui.media.MediaProjectionPermissionActivity
</string>
```

之后在`getMediaProjection()`中实际通过`resultData`的Extra中获取到了MediaProjection的Binder，再通过Binder创建了一个新的`MediaProjection`实例返回。

那也就是说，在systemUI中就已经创建了MediaProjection的Binder实例，那我们看下SystemUI中具体是怎么做的。

`frameworks/base/packages/SystemUI/src/com/android/systemui/media/MediaProjectionPermissionActivity.java`
```java
public class MediaProjectionPermissionActivity extends Activity
        implements DialogInterface.OnClickListener, DialogInterface.OnCancelListener {
    private static final String TAG = "MediaProjectionPermissionActivity";
    private static final float MAX_APP_NAME_SIZE_PX = 500f;
    private static final String ELLIPSIS = "\u2026";

    private String mPackageName;
    private int mUid;
    private IMediaProjectionManager mService;

    private AlertDialog mDialog;

    @Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);

        mPackageName = getCallingPackage();
        IBinder b = ServiceManager.getService(MEDIA_PROJECTION_SERVICE);
        mService = IMediaProjectionManager.Stub.asInterface(b);

        ......
            aInfo = packageManager.getApplicationInfo(mPackageName, 0);
            mUid = aInfo.uid;
        ......

        try {
            if (mService.hasProjectionPermission(mUid, mPackageName)) {
                setResult(RESULT_OK, getMediaProjectionIntent(mUid, mPackageName));
                finish();
                return;
            }
        } catch (RemoteException e) {
            Log.e(TAG, "Error checking projection permissions", e);
            finish();
            return;
        }

        ......
        String dialogTitle = getString(R.string.media_projection_dialog_title);

        View dialogTitleView = View.inflate(this, R.layout.media_projection_dialog_title, null);
        TextView titleText = (TextView) dialogTitleView.findViewById(R.id.dialog_title);
        titleText.setText(dialogTitle);

        mDialog = new AlertDialog.Builder(this)
                .setCustomTitle(dialogTitleView)
                .setMessage(dialogText)
                .setPositiveButton(R.string.media_projection_action_text, this)
                .setNegativeButton(android.R.string.cancel, this)
                .setOnCancelListener(this)
                .create();

        mDialog.create();
        mDialog.getButton(DialogInterface.BUTTON_POSITIVE).setFilterTouchesWhenObscured(true);

        final Window w = mDialog.getWindow();
        w.setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
        w.addSystemFlags(SYSTEM_FLAG_HIDE_NON_SYSTEM_OVERLAY_WINDOWS);

        mDialog.show();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (mDialog != null) {
            mDialog.dismiss();
        }
    }

    @Override
    public void onClick(DialogInterface dialog, int which) {
        try {
            if (which == AlertDialog.BUTTON_POSITIVE) {
                setResult(RESULT_OK, getMediaProjectionIntent(mUid, mPackageName));
            }
        } catch (RemoteException e) {
            Log.e(TAG, "Error granting projection permission", e);
            setResult(RESULT_CANCELED);
        } finally {
            if (mDialog != null) {
                mDialog.dismiss();
            }
            finish();
        }
    }

    private Intent getMediaProjectionIntent(int uid, String packageName)
            throws RemoteException {
        IMediaProjection projection = mService.createProjection(uid, packageName,
                 MediaProjectionManager.TYPE_SCREEN_CAPTURE, false /* permanentGrant */);
        Intent intent = new Intent();
        intent.putExtra(MediaProjectionManager.EXTRA_MEDIA_PROJECTION, projection.asBinder());
        return intent;
    }

    @Override
    public void onCancel(DialogInterface dialog) {
        finish();
    }
}
```

可以看到，当调用了StartActivityForResult后，会跳转到SystemUI中的`MediaProjectionPermissionActivity`中，在该Activity中会通过应用包名以及UID进行判断，如果当前应用已经获取到了MediaProjection权限则直接返回，否则会弹出权限弹窗让用户进行选择：

![media_projection_dialog](https://cdn.jsdelivr.net/gh/jonewan/MD_PNG@master/ScreenRecord/media_projection_dialog.jpg)

只有当用户`确认`权限后才回调用`getMediaProjectionIntent()`方法，通过`IMediaProjection`的Binder接口代理根据应用uid以及应用包名创建一个`TYPE_SCREEN_CAPTURE`的实例并通过`Extra`回传给调用的Activity。如此一来调用方就获取了MediaProjection的有效实例。

## Use Acquired MediaProjection

那获取到了MediaProjection之后能干什么呢？实际上，获取MediaProjection的最终目的就是为了能够拿到屏幕的镜像，Android中通过`虚拟屏幕 - VirtualDisplay`来进行屏幕内容的获取，通常我们可以通过`MediaRecorder`来进行屏幕内容的录制操作：

```java
private void startRecording() {
        try {
            mTempFile = File.createTempFile("temp", ".mp4");
            Log.d(TAG, "Writing video output to: " + mTempFile.getAbsolutePath());

            mMediaRecorder = new MediaRecorder();
            if (mUseAudio) {
                mMediaRecorder.setAudioSource(MediaRecorder.AudioSource.MIC);
            }
            mMediaRecorder.setVideoSource(MediaRecorder.VideoSource.SURFACE);
            mMediaRecorder.setOutputFormat(MediaRecorder.OutputFormat.MPEG_4);

            DisplayMetrics metrics = getResources().getDisplayMetrics();
            int screenWidth = metrics.widthPixels;
            int screenHeight = metrics.heightPixels;
            mMediaRecorder.setVideoEncoder(MediaRecorder.VideoEncoder.H264);
            mMediaRecorder.setVideoSize(screenWidth, screenHeight);
            mMediaRecorder.setVideoFrameRate(VIDEO_FRAME_RATE);
            mMediaRecorder.setVideoEncodingBitRate(VIDEO_BIT_RATE);

            if (mUseAudio) {
                mMediaRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.AMR_NB);
                mMediaRecorder.setAudioChannels(TOTAL_NUM_TRACKS);
                mMediaRecorder.setAudioEncodingBitRate(AUDIO_BIT_RATE);
                mMediaRecorder.setAudioSamplingRate(AUDIO_SAMPLE_RATE);
            }

            mMediaRecorder.setOutputFile(mTempFile);
            mMediaRecorder.prepare();

            mInputSurface = mMediaRecorder.getSurface();
            mVirtualDisplay = mMediaProjection.createVirtualDisplay(
                    "Recording Display",
                    screenWidth,
                    screenHeight,
                    metrics.densityDpi,
                    DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR,
                    mInputSurface,
                    null,
                    null
            );

            mMediaRecorder.start();
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }
    }
```

通过`MediaRecorder.getSurface()`获取到MediaRecorder的输入数据源 - `Surface`，通过获取到的`MediaProjection`调用`createVirtualDisplay()`方法创建一个虚拟屏幕，之后调用`start()`，MediaRecorder就可以源源不断将屏幕内容录制到预先配置的文件路径中了。

所以，根据上面的分析，`MediaProjection`最重要的使用场景就是用来创建虚拟屏幕`VirtualDisplay`，那一定要通过这种方式才能获取到屏幕镜像吗？难道创建虚拟屏幕除了拿到令牌（MediaProjection）就别无他法了吗？

## Process for Creating VirtualDisplay

根据目前的分析，如果应用层想要获取屏幕内容就必须首先获取令牌（MediaProjection），而令牌是独占的，设想一种场景，如果我想在上网课是通过网课应用共享屏幕，并且还要持续录制屏幕内容，那么以当前的分析结果来看这种场景就无法实现，果然，通过对现有Android平板及手机方案的测试：

* 在开启腾讯会议的共享屏幕时同时开启屏幕录制，则腾讯会议的共享屏幕会被中断
* 在进行屏幕直播推流（家长伴学）时开启屏幕录制，则直播推流中断
* 两个录屏软件同时进行屏幕录制，后开启的录制会打断先开始的录制功能
* 等等......

为了破解上面的问题，我们来深入分析一下`CreateVirtualDisplay()`的逻辑：

![](//www.plantuml.com/plantuml/png/VT5HQuCm40VmTp_5FMxKz0C86zsGoQ01cR0zNvEhovec8SQDVVjPDAxQrYS5V__p-D-c2UX3UzI9wvvAA8Sc02UfiFJsYqHGrW0smCk9o5NZDFvoD5YJFu6SBu12sntgdXKBhYB_hGJri4eINW9ZZztxQfM1y8I1tbMtB-eXTtUVv7mm1MEBh1XSrRjdPUJun-ncgegfMxDVJI1lK4zr1bFr9D5rRdrjmjQA0ybvVj_jjxMr4gDxiJwdqxRN0enb87kthYd4vUIbZKN9ICwwAR9uaVWP7g2cy4R_hOovyrBxaOP-ODPX-5HD6eUb853h7Oh3XqVdoAVGj7wFWiOxyf8ncBUNRCZ8zgZ_0000)

* `media/java/android/media/projection/MediaProjection.java`

```java
public final class MediaProjection {
    private static final String TAG = "MediaProjection";

    private final IMediaProjection mImpl;
    private final Context mContext;
    private final Map<Callback, CallbackRecord> mCallbacks;

    /** @hide */
    public MediaProjection(Context context, IMediaProjection impl) {
        mCallbacks = new ArrayMap<Callback, CallbackRecord>();
        mContext = context;
        mImpl = impl;
        try {
            mImpl.start(new MediaProjectionCallback());
        } catch (RemoteException e) {
            throw new RuntimeException("Failed to start media projection", e);
        }
    }

    // 能够通过注册回掉方法的方式，处理令牌丢失时候的状态
    public void registerCallback(Callback callback, Handler handler) {
        if (callback == null) {
            throw new IllegalArgumentException("callback should not be null");
        }
        if (handler == null) {
            handler = new Handler();
        }
        mCallbacks.put(callback, new CallbackRecord(callback, handler));
    }

    ......

    public VirtualDisplay createVirtualDisplay(@NonNull String name,
            int width, int height, int dpi, int flags, @Nullable Surface surface,
            @Nullable VirtualDisplay.Callback callback, @Nullable Handler handler) {
        DisplayManager dm = (DisplayManager) mContext.getSystemService(Context.DISPLAY_SERVICE);
        return dm.createVirtualDisplay(this, name, width, height, dpi, surface, flags, callback,
                handler, null /* uniqueId */);
    }

    // 提供方法主动放弃令牌
    public void stop() {
        try {
            mImpl.stop();
        } catch (RemoteException e) {
            Log.e(TAG, "Unable to stop projection", e);
        }
    }
    
    ......
    
    public static abstract class Callback {
        public void onStop() { }
    }

    private final class MediaProjectionCallback extends IMediaProjectionCallback.Stub {
        @Override
        public void onStop() {
            for (CallbackRecord cbr : mCallbacks.values()) {
                cbr.onStop();
            }
        }
    }

    private final static class CallbackRecord {
        private final Callback mCallback;
        private final Handler mHandler;

        public CallbackRecord(Callback callback, Handler handler) {
            mCallback = callback;
            mHandler = handler;
        }

        public void onStop() {
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    mCallback.onStop();
                }
            });
        }
    }
}
```

可以发现，通过令牌MediaProjection来创建虚拟屏幕最终是调用了DisplayManager的接口：

* `DisplayManager.java`

```java
public final class DisplayManager {

    private final DisplayManagerGlobal mGlobal;

    ......

    /** @hide */
    public DisplayManager(Context context) {
        mContext = context;
        mGlobal = DisplayManagerGlobal.getInstance();
    }
    ......

    /** @hide */
    public VirtualDisplay createVirtualDisplay(@Nullable MediaProjection projection,
            @NonNull String name, int width, int height, int densityDpi, @Nullable Surface surface,
            int flags, @Nullable VirtualDisplay.Callback callback, @Nullable Handler handler,
            @Nullable String uniqueId) {
        return mGlobal.createVirtualDisplay(mContext, projection,
                name, width, height, densityDpi, surface, flags, callback, handler, uniqueId);
    }
    ......
}
```

而`DisplayManager`又调用了单例`DisplayManagerGlobal`来创建：

* `DisplayManagerGlobal.java`

```java
public final class DisplayManagerGlobal {
    
    ......
    @UnsupportedAppUsage
    private final IDisplayManager mDm;
    ......
    
    private DisplayManagerGlobal(IDisplayManager dm) {
        mDm = dm;
        try {
            mWideColorSpace =
                    ColorSpace.get(
                            ColorSpace.Named.values()[mDm.getPreferredWideGamutColorSpaceId()]);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }

    /**
     * 通过Binder获取DisplayManagerService的实例，并返回单例
     */
    @UnsupportedAppUsage
    public static DisplayManagerGlobal getInstance() {
        synchronized (DisplayManagerGlobal.class) {
            if (sInstance == null) {
                IBinder b = ServiceManager.getService(Context.DISPLAY_SERVICE);
                if (b != null) {
                    sInstance = new DisplayManagerGlobal(IDisplayManager.Stub.asInterface(b));
                }
            }
            return sInstance;
        }
    }
    
    public VirtualDisplay createVirtualDisplay(Context context, MediaProjection projection,
            String name, int width, int height, int densityDpi, Surface surface, int flags,
            VirtualDisplay.Callback callback, Handler handler, String uniqueId) {
        if (TextUtils.isEmpty(name)) {
            throw new IllegalArgumentException("name must be non-null and non-empty");
        }
        if (width <= 0 || height <= 0 || densityDpi <= 0) {
            throw new IllegalArgumentException("width, height, and densityDpi must be "
                    + "greater than 0");
        }

        VirtualDisplayCallback callbackWrapper = new VirtualDisplayCallback(callback, handler);
        
        // 表明projection可以为null
        IMediaProjection projectionToken = projection != null ? projection.getProjection() : null;
        int displayId;
        try {
            // 通过DisplayManagerService实际创建VirtualDisplay
            displayId = mDm.createVirtualDisplay(callbackWrapper, projectionToken,
                    context.getPackageName(), name, width, height, densityDpi, surface, flags,
                    uniqueId);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        if (displayId < 0) {
            Log.e(TAG, "Could not create virtual display: " + name);
            return null;
        }
        Display display = getRealDisplay(displayId);
        if (display == null) {
            Log.wtf(TAG, "Could not obtain display info for newly created "
                    + "virtual display: " + name);
            try {
                mDm.releaseVirtualDisplay(callbackWrapper);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            return null;
        }
        return new VirtualDisplay(this, display, callbackWrapper, surface);
    }

```

最后通过调用`DisplayManagerService:BinderService`来构建一个`LogicalDisplay`并返回创建成功的`DisplayID`:

* `DisplayManagerService.java`

```java
public final class DisplayManagerService extends SystemService {

    private int createVirtualDisplayInternal(IVirtualDisplayCallback callback,
            IMediaProjection projection, int callingUid, String packageName, String name, int width,
            int height, int densityDpi, Surface surface, int flags, String uniqueId) {
        synchronized (mSyncRoot) {
            if (mVirtualDisplayAdapter == null) {
                Slog.w(TAG, "Rejecting request to create private virtual display "
                        + "because the virtual display adapter is not available.");
                return -1;
            }

            DisplayDevice device = mVirtualDisplayAdapter.createVirtualDisplayLocked(
                    callback, projection, callingUid, packageName, name, width, height, densityDpi,
                    surface, flags, uniqueId);
            if (device == null) {
                return -1;
            }

            handleDisplayDeviceAddedLocked(device);
            LogicalDisplay display = findLogicalDisplayForDeviceLocked(device);
            if (display != null) { // 返回创建的display id
                return display.getDisplayIdLocked();
            }

            // Something weird happened and the logical display was not created.
            Slog.w(TAG, "Rejecting request to create virtual display "
                    + "because the logical display was not created.");
            mVirtualDisplayAdapter.releaseVirtualDisplayLocked(callback.asBinder());
            handleDisplayDeviceRemovedLocked(device);
        }
        return -1;
    }
    
    ......
    
    final class BinderService extends IDisplayManager.Stub {
        ......
        
        @Override // Binder call
        public int createVirtualDisplay(IVirtualDisplayCallback callback,
                IMediaProjection projection, String packageName, String name,
                int width, int height, int densityDpi, Surface surface, int flags,
                String uniqueId) {
            final int callingUid = Binder.getCallingUid();
            if (!validatePackageName(callingUid, packageName)) {
                throw new SecurityException("packageName must match the calling uid");
            }
            if (callback == null) {
                throw new IllegalArgumentException("appToken must not be null");
            }
            if (TextUtils.isEmpty(name)) {
                throw new IllegalArgumentException("name must be non-null and non-empty");
            }
            if (width <= 0 || height <= 0 || densityDpi <= 0) {
                throw new IllegalArgumentException("width, height, and densityDpi must be "
                        + "greater than 0");
            }
            if (surface != null && surface.isSingleBuffered()) {
                throw new IllegalArgumentException("Surface can't be single-buffered");
            }

            if ((flags & VIRTUAL_DISPLAY_FLAG_PUBLIC) != 0) {
                flags |= VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR;

                // Public displays can't be allowed to show content when locked.
                if ((flags & VIRTUAL_DISPLAY_FLAG_CAN_SHOW_WITH_INSECURE_KEYGUARD) != 0) {
                    throw new IllegalArgumentException(
                            "Public display must not be marked as SHOW_WHEN_LOCKED_INSECURE");
                }
            }
            if ((flags & VIRTUAL_DISPLAY_FLAG_OWN_CONTENT_ONLY) != 0) {
                flags &= ~VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR;
            }

            if (projection != null) { // 检查对应的uid是否获取到令牌
                try {
                    if (!getProjectionService().isValidMediaProjection(projection)) { // 如果不是有效令牌则抛出权限异常
                        throw new SecurityException("Invalid media projection");
                    }
                    flags = projection.applyVirtualDisplayFlags(flags);
                } catch (RemoteException e) {
                    throw new SecurityException("unable to validate media projection or flags");
                }
            }

            // 检查非SYSTEM_UID应用进程是否具有捕获视频的权限
            if (callingUid != Process.SYSTEM_UID &&
                    (flags & VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR) != 0) {
                if (!canProjectVideo(projection)) {
                    throw new SecurityException("Requires CAPTURE_VIDEO_OUTPUT or "
                            + "CAPTURE_SECURE_VIDEO_OUTPUT permission, or an appropriate "
                            + "MediaProjection token in order to create a screen sharing virtual "
                            + "display.");
                }
            }
            if (callingUid != Process.SYSTEM_UID && (flags & VIRTUAL_DISPLAY_FLAG_SECURE) != 0) {
                if (!canProjectSecureVideo(projection)) {
                    throw new SecurityException("Requires CAPTURE_SECURE_VIDEO_OUTPUT "
                            + "or an appropriate MediaProjection token to create a "
                            + "secure virtual display.");
                }
            }

            // Sometimes users can have sensitive information in system decoration windows. An app
            // could create a virtual display with system decorations support and read the user info
            // from the surface.
            // We should only allow adding flag VIRTUAL_DISPLAY_FLAG_SHOULD_SHOW_SYSTEM_DECORATIONS
            // to virtual displays that are owned by the system.
            if (callingUid != Process.SYSTEM_UID
                    && (flags & VIRTUAL_DISPLAY_FLAG_SHOULD_SHOW_SYSTEM_DECORATIONS) != 0) {
                if (!checkCallingPermission(INTERNAL_SYSTEM_WINDOW, "createVirtualDisplay()")) {
                    throw new SecurityException("Requires INTERNAL_SYSTEM_WINDOW permission");
                }
            }

            final long token = Binder.clearCallingIdentity();
            try {
                return createVirtualDisplayInternal(callback, projection, callingUid, packageName,
                        name, width, height, densityDpi, surface, flags, uniqueId);
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }
        ......

        private boolean canProjectVideo(IMediaProjection projection) {
            if (projection != null) {
                try {
                    if (projection.canProjectVideo()) {
                        return true;
                    }
                } catch (RemoteException e) {
                    Slog.e(TAG, "Unable to query projection service for permissions", e);
                }
            }
            if (checkCallingPermission(CAPTURE_VIDEO_OUTPUT, "canProjectVideo()")) {
                return true;
            }
            return canProjectSecureVideo(projection);
        }
        
        private boolean canProjectSecureVideo(IMediaProjection projection) {
            if (projection != null) {
                try {
                    if (projection.canProjectSecureVideo()){
                        return true;
                    }
                } catch (RemoteException e) {
                    Slog.e(TAG, "Unable to query projection service for permissions", e);
                }
            }
            return checkCallingPermission(CAPTURE_SECURE_VIDEO_OUTPUT, "canProjectSecureVideo()");
        }
        ......
    }
    ......
}
```

## Bypass MediaProjection To Create VirtualDisplay

根据`DisplayManagerService:createVirtualDisplay()`中的projection检测，如果projection存在，则需要通过projection的flag进行检测，否侧需要检测`CAPTURE_VIDEO_OUTPUT`和`CAPTURE_SECURE_VIDEO_OUTPUT`这两个权限。

* `AndroidManifest.xml`

```xml
    <!-- Allows an application to capture video output.
         <p>Not for use by third-party applications.</p>
          @hide
          @removed -->
    <permission android:name="android.permission.CAPTURE_VIDEO_OUTPUT"
        android:protectionLevel="signature" />

    <!-- Allows an application to capture secure video output.
         <p>Not for use by third-party applications.</p>
          @hide
          @removed -->
    <permission android:name="android.permission.CAPTURE_SECURE_VIDEO_OUTPUT"
        android:protectionLevel="signature" />
```

根据AndroidManifest.xml中的定义，`CAPTURE_VIDEO_OUTPUT`和`CAPTURE_SECURE_VIDEO_OUTPUT`均需要系统签名应用才能够获取，且这两个权限被标记了`hide`，因此三方应用无法正常声明与动态获取，因此，只有系统UID应用能够默认获取这两种权限。

因此，根据上面代码的分析，在条件允许的情况下，我们可以通过配置应用为UID_SYSTEM获取系统签名后就可以绕过`MediaProjection`直接通过`DisplayManager`来创建`VirtualDisplay`进行使用了，如此以来就避免了由于MediaProjection抢占导致的录屏冲突、直播推流冲突等功能限制：

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
