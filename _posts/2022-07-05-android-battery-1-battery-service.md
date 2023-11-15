---
date: 2022-07-05 08:00:00
layout: post
title: AndroidQ Battery（一） - 深入分析BatteryService
subtitle: AndroidQ Battery（一） - 深入分析BatteryService
description: AndroidQ Battery（一） - 深入分析BatteryService
image: /assets/img/markdown_img/image-title-android-battery.jpg
optimized_image: /assets/img/markdown_img/image-title-android-battery.jpg
category: Android System
tags:
    - Android System
    - Android Battery
author: etoilexx
paginate: false
---

## BatteryService Overview（概述）

根据Google官方的描述，`BatteryService`用于监控设备电池的充电状态和充电水平。 当这些状态发生改变时，`BatteryService`会将新的状态值通过广播` android.content.Intent#ACTION_BATTERY_CHANGED BATTERY_CHANGED`发送到所有监听了` android.content.BroadcastReceiver`的`IntentReceivers`中。

新的状态值会存放在广播Intent中的Extra Data中，可以通过`Intent.getExtra()`进行获取，其中包含如下的信息：

|     Key     |   Type  |                Des               |
| :---------: | :-----: | :------------------------------: |
|    scale    |   int   |     用于标识充电等级的最大值（如100：表示100%）    |
|    level    |   int   |         当前电量等级（如90表示90%）         |
|    status   |  String |              当前充电状态              |
|    health   |  String |             当前电池健康状态             |
|   present   | boolean |             标识电池是否存在             |
|   plugged   |   int   | 0 - 没有电源插入 1 - AC电源适配器 2 - USB连接 |
|   vlotage   |   int   |          当前电池电压（以毫伏为单位）          |
| temperature |   int   |    当前电池温度（摄氏度\*10：300 - 30'C）    |
|  technology |  String |     安装的电池类型，例如 "Li-ion" - 锂电池    |

由于当`BatteryService`持有`PowerManagerService`的锁时会被其调用，因此所有的外部调用全部放在`ActivityManagerService`中处理。

## BatteryService Start（启动）

*   `frameworks/base/services/java/com/android/server/SystemServer.java`

```java
public final class SystemServer {
    ......
    private void startCoreServices() {
        traceBeginAndSlog("StartBatteryService");
        // Tracks the battery level.  Requires LightService.
        mSystemServiceManager.startService(BatteryService.class);
        traceEnd();
        ......
    }
    ......
}
```

BatteryService作为系统核心服务，由SystemService启动。

## BatteryService Constructor（构造）

*   `frameworks/base/services/core/java/com/android/server/BatteryService.java`

```java
public final class BatteryService extends SystemService {
    ......
    public BatteryService(Context context) {
        super(context);
    
        mContext = context;
        mHandler = new Handler(true /*async*/);
        // 充电指示灯，需硬件支持
        mLed = new Led(context, getLocalService(LightsManager.class));
        mBatteryStats = BatteryStatsService.getService(); // 获取BatteryStatsService代理
        // 获取ActivityManagerInternal 内部接口服务
        mActivityManagerInternal = LocalServices.getService(ActivityManagerInternal.class);
        ......
        // 当电池电量降至此值时显示低电量警告 默认15%
        mLowBatteryWarningLevel = mContext.getResources().getInteger(
                com.android.internal.R.integer.config_lowBatteryWarningLevel);
        // 当电池电量达到 lowBatteryWarningLevel 时关闭低电量警告，默认20%
        mLowBatteryCloseWarningLevel = mLowBatteryWarningLevel + mContext.getResources().getInteger(
                com.android.internal.R.integer.config_lowBatteryCloseWarningBump);
        // 电池到达该温度时触发自动关机， 默认65’C
        mShutdownBatteryTemperature = mContext.getResources().getInteger(
                com.android.internal.R.integer.config_shutdownBatteryTemperature);
    
        mBatteryLevelsEventQueue = new ArrayDeque<>();
        mMetricsLogger = new MetricsLogger();// 用于打印sysui为TAG的logcat event
    
        // 注册一个UEvent监听器，监听从Kernel发送的无效充电器插拔事件
        if (new File("/sys/devices/virtual/switch/invalid_charger/state").exists()) {
            UEventObserver invalidChargerObserver = new UEventObserver() {
                @Override
                public void onUEvent(UEvent event) {
                    ......
                }
            };
            invalidChargerObserver.startObserving(
                    "DEVPATH=/devices/virtual/switch/invalid_charger");
        }
    }
    ......
}
```

## BatteryService Regeist（注册）

*   `frameworks/base/services/core/java/com/android/server/BatteryService.java`

```java
@Override
public void onStart() {
    registerHealthCallback(); // 注册电池健康状态回调

    mBinderService = new BinderService();
    // 注册App层可调用服务battery
    publishBinderService("battery", mBinderService);
    mBatteryPropertiesRegistrar = new BatteryPropertiesRegistrar();
    // 注册App层可调用服务batteryproperties
    publishBinderService("batteryproperties", mBatteryPropertiesRegistrar);
    // 注册system_server内部调用的服务
    publishLocalService(BatteryManagerInternal.class, new LocalService());
}

@Override
public void onBootPhase(int phase) {
    // ActivityManager准备就绪，为了保证关机弹窗能够正常弹出
    if (phase == PHASE_ACTIVITY_MANAGER_READY) {
        synchronized (mLock) {
            ContentObserver obs = new ContentObserver(mHandler) {
                @Override
                public void onChange(boolean selfChange) {
                    synchronized (mLock) {
                        updateBatteryWarningLevelLocked();
                    }
                }
            };
            final ContentResolver resolver = mContext.getContentResolver();
            resolver.registerContentObserver(Settings.Global.getUriFor(
                    Settings.Global.LOW_POWER_MODE_TRIGGER_LEVEL), // 监听低电量模式settings key
                    false, obs, UserHandle.USER_ALL);
            updateBatteryWarningLevelLocked();
        }
    }
}

private final class HealthHalCallback extends IHealthInfoCallback.Stub
        implements HealthServiceWrapper.Callback {
    // 当HAL层数据发生更新时会主动调用该接口进行更新数据
    @Override public void healthInfoChanged(android.hardware.health.V2_0.HealthInfo props) {
        BatteryService.this.update(props);
    }
    // on new service registered
    @Override public void onRegistration(IHealth oldService, IHealth newService,
            String instance) {
        if (newService == null) return;

        traceBegin("HealthUnregisterCallback");
        try {
            if (oldService != null) { // 先注销旧服务回调
                int r = oldService.unregisterCallback(this);
                if (r != Result.SUCCESS) {
                    Slog.w(TAG, "health: cannot unregister previous callback: " +
                            Result.toString(r));
                }
            }
        } catch (RemoteException ex) {
            Slog.w(TAG, "health: cannot unregister previous callback (transaction error): "
                        + ex.getMessage());
        } finally {
            traceEnd();
        }

        traceBegin("HealthRegisterCallback");
        try {
            int r = newService.registerCallback(this); // 向HAL层注册新的上层回调，用于更新数据
            if (r != Result.SUCCESS) {
                Slog.w(TAG, "health: cannot register callback: " + Result.toString(r));
                return;
            }

            newService.update(); // 向HAL注册完成后主动调用一次更新
        } catch (RemoteException ex) {
            Slog.e(TAG, "health: cannot register callback (transaction error): "
                    + ex.getMessage());
        } finally {
            traceEnd();
        }
    }
}

// 注册电池健康状态更新回调方法，当底层电池状态发生更新时会触发该回调进行数据更新
private void registerHealthCallback() {
    traceBegin("HealthInitWrapper");
    mHealthServiceWrapper = new HealthServiceWrapper();// 监听HAL层服务的包装器
    mHealthHalCallback = new HealthHalCallback(); // HAL层的callback，由HAL层服务调用
    try {
        // 通过HealthServiceWrapper建立与HAL层的连接
        mHealthServiceWrapper.init(mHealthHalCallback,
                new HealthServiceWrapper.IServiceManagerSupplier() {},
                new HealthServiceWrapper.IHealthSupplier() {});
    } catch (RemoteException ex) {
        Slog.e(TAG, "health: cannot register callback. (RemoteException)");
        throw ex.rethrowFromSystemServer();
    } catch (NoSuchElementException ex) {
        Slog.e(TAG, "health: cannot register callback. (no supported health HAL service)");
        throw ex;
    } finally {
        traceEnd();
    }

    traceBegin("HealthInitWaitUpdate");
    long beforeWait = SystemClock.uptimeMillis();
    synchronized (mLock) {
        while (mHealthInfo == null) {
            Slog.i(TAG, "health: Waited " + (SystemClock.uptimeMillis() - beforeWait) +
                    "ms for callbacks. Waiting another " + HEALTH_HAL_WAIT_MS + " ms...");
            try {
                mLock.wait(HEALTH_HAL_WAIT_MS);
            } catch (InterruptedException ex) {
                Slog.i(TAG, "health: InterruptedException when waiting for update. "
                    + " Continuing...");
            }
        }
    }

// 触发低电量告警时获取电池状态更新(开机及监听到低电量告警Settings时)
private void updateBatteryWarningLevelLocked() {
    ......
    processValuesLocked(true);
}    
```

在`registerHealthCallback()`中，我们看到源码中定义了`HealthServiceWrapper`与`HealthHalCallback`对象用于与HAL层进行通信，获取HAL层传递上来的电池健康数据，那到底整个流程是怎么样注册的呢？

*   `frameworks/base/services/core/java/com/android/server/BatteryService.java`

```java
/**
* HealthServiceWrapper用于包装内部接口IHealth并且在必要时更新对应的服务
* 当注册了一个新的IHealth服务时，onRegistration将会被调用，同时更新内部服务。
*/
static final class HealthServiceWrapper {
    private static final String TAG = "HealthServiceWrapper";
    // 实现Health HAL层接口的实例名称
    public static final String INSTANCE_HEALTHD = "backup";
    public static final String INSTANCE_VENDOR = "default";
    // 优先级从高到低，INSTANCE_VENDOR > INSTANCE_HEALTHD
    // 也就是说会先从HWServiceManager中查找default的实现，如果没找到才会使用backup的实现
    private static final List<String> sAllInstances =
            Arrays.asList(INSTANCE_VENDOR, INSTANCE_HEALTHD);

    private final IServiceNotification mNotification = new Notification();
    private final HandlerThread mHandlerThread = new HandlerThread("HealthServiceRefresh");
    // These variables are fixed after init.
    private Callback mCallback;
    private IHealthSupplier mHealthSupplier;
    private String mInstanceName;

    // Last IHealth service received.
    private final AtomicReference<IHealth> mLastService = new AtomicReference<>();

    HealthServiceWrapper() { }

    IHealth getLastService() {
        return mLastService.get();
    }

    void init(Callback callback,
              IServiceManagerSupplier managerSupplier,
              IHealthSupplier healthSupplier)
            throws RemoteException, NoSuchElementException, NullPointerException {
        if (callback == null || managerSupplier == null || healthSupplier == null)
            throw new NullPointerException();

        IServiceManager manager;

        mCallback = callback;
        mHealthSupplier = healthSupplier;

        // Initialize mLastService and call callback for the first time (in init thread)
        IHealth newService = null;
        for (String name : sAllInstances) {
            traceBegin("HealthInitGetService_" + name);
            try {
                // 从HAL层ServiceManager中查找IHealth的实现
                newService = healthSupplier.get(name);
            } catch (NoSuchElementException ex) {
                /* ignored, handled below */
            } finally {
                traceEnd();
            }
            if (newService != null) {
                mInstanceName = name;
                mLastService.set(newService);
                break;
            }
        }

        // 如果没找到IHealth的对应实现则抛出异常
        if (mInstanceName == null || newService == null) {
            throw new NoSuchElementException(String.format(
                    "No IHealth service instance among %s is available. Perhaps no permission?",
                    sAllInstances.toString()));
        }
        // 通过HIDL接口进行回调注册
        mCallback.onRegistration(null, newService, mInstanceName);

        // Register for future service registrations
        traceBegin("HealthInitRegisterNotification");
        mHandlerThread.start();
        try {
            managerSupplier.get().registerForNotifications(
                    IHealth.kInterfaceName, mInstanceName, mNotification);
        } finally {
            traceEnd();
        }
        Slog.i(TAG, "health: HealthServiceWrapper listening to instance " + mInstanceName);
    }

    @VisibleForTesting
    HandlerThread getHandlerThread() {
        return mHandlerThread;
    }

    interface Callback {
        void onRegistration(IHealth oldService, IHealth newService, String instance);
    }

    interface IServiceManagerSupplier {
        default IServiceManager get() throws NoSuchElementException, RemoteException {
            return IServiceManager.getService();// 获取HAL层的ServiceManagerService
        }
    }
    
    interface IHealthSupplier {
        default IHealth get(String name) throws NoSuchElementException, RemoteException {
            return IHealth.getService(name, true /* retry */); // 获取Health service HAL层的服务
        }
    }

    private class Notification extends IServiceNotification.Stub {
        @Override
        public final void onRegistration(String interfaceName, String instanceName,
                boolean preexisting) {
            if (!IHealth.kInterfaceName.equals(interfaceName)) return;
            if (!mInstanceName.equals(instanceName)) return;

            mHandlerThread.getThreadHandler().post(new Runnable() {
                @Override
                public void run() {
                    try {
                        IHealth newService = mHealthSupplier.get(mInstanceName);
                        IHealth oldService = mLastService.getAndSet(newService);

                        // preexisting may be inaccurate (race). Check for equality here.
                        if (Objects.equals(newService, oldService)) return;

                        Slog.i(TAG, "health: new instance registered " + mInstanceName);
                        mCallback.onRegistration(oldService, newService, mInstanceName);
                    } catch (NoSuchElementException | RemoteException ex) {
                        Slog.e(TAG, "health: Cannot get instance '" + mInstanceName
                                + "': " + ex.getMessage() + ". Perhaps no permission?");
                    }
                }
            });
        }
    }
}
```

## BatteryService update（更新）

`BatteryService.java`

```java
// 当HAL层数据发生更新时，通过回调方法进行电池状态信息更新
private void update(android.hardware.health.V2_0.HealthInfo info) {
    ......
    synchronized (mLock) {
        if (!mUpdatesStopped) {
            mHealthInfo = info.legacy; // 将底层传递上来的信息进行保存
            // Process the new values.
            processValuesLocked(false);
            mLock.notifyAll(); // for any waiters on new info
        } else {
            copy(mLastHealthInfo, info.legacy);
        }
    }
    ......
}

// 通过shell cmd调用dumpsys battery获取电池状态更新
private void processValuesFromShellLocked(PrintWriter pw, int opts) {
    processValuesLocked((opts & OPTION_FORCE_UPDATE) != 0);
    ......
}

// 通过解析保存的mHealthInfo的数据获取上层需要的电池信息
private void processValuesLocked(boolean force) {
    boolean logOutlier = false;
    long dischargeDuration = 0;

    mBatteryLevelCritical =
        mHealthInfo.batteryStatus != BatteryManager.BATTERY_STATUS_UNKNOWN
        && mHealthInfo.batteryLevel <= mCriticalBatteryLevel;
        
    // 获取当前的充电状态并进行记录
    if (mHealthInfo.chargerAcOnline) { // 通过电源线连接
        mPlugType = BatteryManager.BATTERY_PLUGGED_AC;
    } else if (mHealthInfo.chargerUsbOnline) { // usb充电
        mPlugType = BatteryManager.BATTERY_PLUGGED_USB;
    } else if (mHealthInfo.chargerWirelessOnline) {// 无线充电
        mPlugType = BatteryManager.BATTERY_PLUGGED_WIRELESS;
    } else { // 未充电
        mPlugType = BATTERY_PLUGGED_NONE;
    }

    try {// 更新电池状态记录
        mBatteryStats.setBatteryState(mHealthInfo.batteryStatus, mHealthInfo.batteryHealth,
                mPlugType, mHealthInfo.batteryLevel, mHealthInfo.batteryTemperature,
                mHealthInfo.batteryVoltage, mHealthInfo.batteryChargeCounter,
                mHealthInfo.batteryFullCharge);
    } catch (RemoteException e) {
        // Should never happen.
    }

    shutdownIfNoPowerLocked(); // 若电池电量耗尽时主动调用PowerManager进行关机
    shutdownIfOverTempLocked(); // 当电池过热时主动调用PowerManager进行关机

    // 当强制更新或者健康状态中任意一项发生改变时则会进行对应的电池状态更新
    if (force || (mHealthInfo.batteryStatus != mLastBatteryStatus ||
            mHealthInfo.batteryHealth != mLastBatteryHealth ||
            mHealthInfo.batteryPresent != mLastBatteryPresent ||
            mHealthInfo.batteryLevel != mLastBatteryLevel ||
            mPlugType != mLastPlugType ||
            mHealthInfo.batteryVoltage != mLastBatteryVoltage ||
            mHealthInfo.batteryTemperature != mLastBatteryTemperature ||
            mHealthInfo.maxChargingCurrent != mLastMaxChargingCurrent ||
            mHealthInfo.maxChargingVoltage != mLastMaxChargingVoltage ||
            mHealthInfo.batteryChargeCounter != mLastChargeCounter ||
            mInvalidCharger != mLastInvalidCharger)) {

        if (mPlugType != mLastPlugType) { // 充电类型发生改变
            if (mLastPlugType == BATTERY_PLUGGED_NONE) { // 未充电 -> 充电
                mChargeStartLevel = mHealthInfo.batteryLevel;
                mChargeStartTime = SystemClock.elapsedRealtime();

                final LogMaker builder = new LogMaker(MetricsEvent.ACTION_CHARGE);
                builder.setType(MetricsEvent.TYPE_ACTION);
                builder.addTaggedData(MetricsEvent.FIELD_PLUG_TYPE, mPlugType);
                builder.addTaggedData(MetricsEvent.FIELD_BATTERY_LEVEL_START,
                        mHealthInfo.batteryLevel);
                mMetricsLogger.write(builder); // 创建sysui为tag的event事件打印
                
                // 记录设备断开充电的时长
                if (mDischargeStartTime != 0 && mDischargeStartLevel != mHealthInfo.batteryLevel) {
                    dischargeDuration = SystemClock.elapsedRealtime() - mDischargeStartTime;
                    logOutlier = true;
                    EventLog.writeEvent(EventLogTags.BATTERY_DISCHARGE, dischargeDuration,
                            mDischargeStartLevel, mHealthInfo.batteryLevel);
                    mDischargeStartTime = 0;// 恢复初始记录
                }
            } else if (mPlugType == BATTERY_PLUGGED_NONE) { // 充电 -> 未充电/刚开机
                mDischargeStartTime = SystemClock.elapsedRealtime();
                mDischargeStartLevel = mHealthInfo.batteryLevel;

                long chargeDuration = SystemClock.elapsedRealtime() - mChargeStartTime;
                if (mChargeStartTime != 0 && chargeDuration != 0) {
                    final LogMaker builder = new LogMaker(MetricsEvent.ACTION_CHARGE);
                    builder.setType(MetricsEvent.TYPE_DISMISS);
                    builder.addTaggedData(MetricsEvent.FIELD_PLUG_TYPE, mLastPlugType);
                    builder.addTaggedData(MetricsEvent.FIELD_CHARGING_DURATION_MILLIS,
                            chargeDuration);
                    builder.addTaggedData(MetricsEvent.FIELD_BATTERY_LEVEL_START,
                            mChargeStartLevel);
                    builder.addTaggedData(MetricsEvent.FIELD_BATTERY_LEVEL_END,
                            mHealthInfo.batteryLevel);
                    mMetricsLogger.write(builder);
                }
                mChargeStartTime = 0;
            }
        }
        if (mHealthInfo.batteryStatus != mLastBatteryStatus || // 充电状态发生改变
                mHealthInfo.batteryHealth != mLastBatteryHealth || // 健康状态发生改变
                mHealthInfo.batteryPresent != mLastBatteryPresent || // 电池存在情况发生改变
                mPlugType != mLastPlugType) { // 充电类型发生改变
            // 记录一条event到logcat中，通过logcat -b events查看
            EventLog.writeEvent(EventLogTags.BATTERY_STATUS,
                    mHealthInfo.batteryStatus, mHealthInfo.batteryHealth, mHealthInfo.batteryPresent ? 1 : 0,
                    mPlugType, mHealthInfo.batteryTechnology);
        }
        if (mHealthInfo.batteryLevel != mLastBatteryLevel) { // 剩余电量发生改变时通过Event记录到logcat中
            EventLog.writeEvent(EventLogTags.BATTERY_LEVEL,
                                mHealthInfo.batteryLevel,
                                mHealthInfo.batteryVoltage,
                                mHealthInfo.batteryTemperature);
        }
        if (mBatteryLevelCritical && !mLastBatteryLevelCritical &&
                mPlugType == BATTERY_PLUGGED_NONE) { // 
            dischargeDuration = SystemClock.elapsedRealtime() - mDischargeStartTime;
            logOutlier = true;
        }

        if (!mBatteryLevelLow) { // 设置低电量模式
            // Should we now switch in to low battery mode?
            if (mPlugType == BATTERY_PLUGGED_NONE // 未充电状态
                    && mHealthInfo.batteryStatus !=
                       BatteryManager.BATTERY_STATUS_UNKNOWN // 触发低电量阈值
                    && mHealthInfo.batteryLevel <= mLowBatteryWarningLevel) {
                mBatteryLevelLow = true; // 配置进入低电量模式
            }
        } else { // 解除低电量模式
            if (mPlugType != BATTERY_PLUGGED_NONE) {
                mBatteryLevelLow = false; // 充电状态下解除
            } else if (mHealthInfo.batteryLevel >= mLowBatteryCloseWarningLevel)  {
                mBatteryLevelLow = false; // 高于关闭低电量警告的阈值时解除
            } else if (force && mHealthInfo.batteryLevel >= mLowBatteryWarningLevel) {
                mBatteryLevelLow = false; // 无视强制更新，只要高于低电量警告就解除
            }
        }

        mSequence++;

        // 电源连接状态的广播单独进行发送，不与BatteryChange的广播混淆
        if (mPlugType != 0 && mLastPlugType == 0) { // 电源连接
            final Intent statusIntent = new Intent(Intent.ACTION_POWER_CONNECTED);
            statusIntent.setFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
            statusIntent.putExtra(BatteryManager.EXTRA_SEQUENCE, mSequence);
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    mContext.sendBroadcastAsUser(statusIntent, UserHandle.ALL);
                }
            });
        }
        else if (mPlugType == 0 && mLastPlugType != 0) { // 断开电源连接
            final Intent statusIntent = new Intent(Intent.ACTION_POWER_DISCONNECTED);
            statusIntent.setFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
            statusIntent.putExtra(BatteryManager.EXTRA_SEQUENCE, mSequence);
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    mContext.sendBroadcastAsUser(statusIntent, UserHandle.ALL);
                }
            });
        }

        if (shouldSendBatteryLowLocked()) {
            mSentLowBatteryBroadcast = true;
            final Intent statusIntent = new Intent(Intent.ACTION_BATTERY_LOW);
            statusIntent.setFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
            statusIntent.putExtra(BatteryManager.EXTRA_SEQUENCE, mSequence);
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    mContext.sendBroadcastAsUser(statusIntent, UserHandle.ALL);
                }
            });
        } else if (mSentLowBatteryBroadcast &&
                mHealthInfo.batteryLevel >= mLowBatteryCloseWarningLevel) {
            mSentLowBatteryBroadcast = false;
            final Intent statusIntent = new Intent(Intent.ACTION_BATTERY_OKAY);
            statusIntent.setFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
            statusIntent.putExtra(BatteryManager.EXTRA_SEQUENCE, mSequence);
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    mContext.sendBroadcastAsUser(statusIntent, UserHandle.ALL);
                }
            });
        }

        // 发送BatteryChanged广播
        sendBatteryChangedIntentLocked();
        if (mLastBatteryLevel != mHealthInfo.batteryLevel || mLastPlugType != mPlugType) {
            sendBatteryLevelChangedIntentLocked(); // 发送BatteryLevelChanged广播
        }

        // 更新电源指示灯状态
        mLed.updateLightsLocked();

        // This needs to be done after sendIntent() so that we get the lastest battery stats.
        if (logOutlier && dischargeDuration != 0) {
            logOutlierLocked(dischargeDuration);
        }

        // 将所有电池状态旧数据进行更新
        mLastBatteryStatus = mHealthInfo.batteryStatus;
        mLastBatteryHealth = mHealthInfo.batteryHealth;
        mLastBatteryPresent = mHealthInfo.batteryPresent;
        mLastBatteryLevel = mHealthInfo.batteryLevel;
        mLastPlugType = mPlugType;
        mLastBatteryVoltage = mHealthInfo.batteryVoltage;
        mLastBatteryTemperature = mHealthInfo.batteryTemperature;
        mLastMaxChargingCurrent = mHealthInfo.maxChargingCurrent;
        mLastMaxChargingVoltage = mHealthInfo.maxChargingVoltage;
        mLastChargeCounter = mHealthInfo.batteryChargeCounter;
        mLastBatteryLevelCritical = mBatteryLevelCritical;
        mLastInvalidCharger = mInvalidCharger;
    }
}
```

## 总结

`BatteryService`通过实例化`HealthServiceWrapper`将回调方法`HealthHalCallback`注册到HAL层中，之后当HAL层数据发生更新时，会主动调用`healthInfoChanged()`将底层包装的电池信息数据`android.hardware.health.V2_0.HealthInfo`传递给BatteryService进行解析，之后BatteryService保存完成当此数据后则会发送`BatteryChanged`广播，通知注册了对应Intent的广播接收器，完成整个链路，那HAL层到底是如何更新数据的呢？下一节我们重点分析。

