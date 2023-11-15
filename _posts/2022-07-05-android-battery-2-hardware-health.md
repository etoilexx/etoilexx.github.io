---
date: 2022-07-05 08:00:00
layout: post
title: AndroidQ Battery（二） - 深入分析android.hardware.health@2.0
subtitle: AndroidQ Battery（二） - 深入分析android.hardware.health@2.0
description: AndroidQ Battery（二） - 深入分析android.hardware.health@2.0
image: /assets/img/markdown_img/image-title-android-battery.jpg
optimized_image: /assets/img/markdown_img/image-title-android-battery.jpg
category: Android System
tags:
    - Android System
    - Android Battery
author: etoilexx
paginate: false
---

## android.hardware.health@2.0 Overview（架构概览）

在Android 8.0之前Google采用了一个守护进程`Healthd`来进行进行底层电池数据的获取与上层数据的提供，而在Android 9.0中引入了`android.hardware.health HAL 2.0`，为了Framework与HAL层的升级解耦，能够让Android Framework的升级不再依赖OEM厂商的设备通用接口，采用了HIDL的方式。

Google官方对于AndroidQ中Battery框架的定义如下图所示：

![](https://source.android.com/static/devices/tech/health/images/health-component-2.png?hl=zh-cn)

应用层通过调用`BatteryManager`中的API接口获取对应信息，或者通过getSystemService获取`BatteryService`实现调用，`BatteryManager`与`BatteryService`通过Binder接口`IBatteryPropertiesRegistrar`进行通信，往下层，BatteryService中具有一个用于兼容新旧版本的包装器`HealthServiceWrapper`，该框架尝试从 hwservicemanager 中检索`health@2.0-service`，如果检索失败，将尝试调用 health@1.0-service（AndroidO），检索成功后会通过hwbinder与health@x.0-service进行通信与电池数据获取。

可以看到在`health@2.0-service`中主要由`health@2.0-impl`、`libhealthservice`、`libbatterymonitor`这些依赖，在AndroidP中新增了存储的健康状态，我们这里先不做分析，先逐一分析电池健康相关的对应的实现。

### health@2.0-HAL Interface（HAL层接口定义）

上层HealthServiceWrapper与HAL层的通信桥梁就是通过HAL提供的接口，所以我们先从接口定义进行分析：

* `hardware/interfaces/health/2.0/IHealth.hal`

```cpp
package android.hardware.health@2.0;

import @1.0::BatteryStatus;

import IHealthInfoCallback;

/**
 * IHealth manages health info and posts events on registered callbacks.
 */
interface IHealth {

    /**
     * 用于注册健康状态发生变化的事件回调，用于替换 IBatteryPropertiesRegistrar.registerListener
     */
    registerCallback(IHealthInfoCallback callback) generates (Result result);

    /**
     * 注销回调，用于替换 IBatteryPropertiesRegistrar.unregisterListener
     */
    unregisterCallback(IHealthInfoCallback callback) generates (Result result);

    /**
     * Schedule update，用于替换 IBatteryPropertiesRegistrar.scheduleUpdate
     */
    update() generates (Result result);

    /**
    * 以下信息获取接口用于替换原本在Healthd中的IBatteryPropertiesRegistrar.getProperties接口
    */
    
    /**
     * 以µAh为单位获取电池容量
     */
    getChargeCounter() generates (Result result, int32_t value);

    /**
     * 以微安 (µA) 为单位获取瞬时电池电流
     *
     * 正值表示从充电源进入电池的净电流，负值表示从电池放电的净电流。
     */
    getCurrentNow() generates (Result result, int32_t value);

    /**
     * 以微安 (µA) 为单位获取平均电池电流
     *
     * 正值表示从充电源进入电池的净电流，负值表示从电池放电的净电流。
     * 计算平均值的时间段可能取决于电量计硬件及其配置。
     */
    getCurrentAverage() generates (Result result, int32_t value);

    /**
     * 获取剩余电池容量占总容量的百分比（没有小数部分）
     */
    getCapacity() generates (Result result, int32_t value);

    /**
     * 以毫瓦时计算电池剩余容量
     */
    getEnergyCounter() generates (Result result, int64_t value);

    /**
     * 获取电池充电状态
     */
    getChargeStatus() generates (Result result, BatteryStatus value);

    /**
     * 获取存储信息
     */
    getStorageInfo() generates (Result result, vec<StorageInfo> value);

    /**
     * 获取磁盘统计信息（已处理的读/写次数、正在处理的I/O操作次数等）
     */
    getDiskStats() generates (Result result, vec<DiskStats> value);

    /**
     * 获取健康状态信息。
     */
    getHealthInfo() generates (Result result, @2.0::HealthInfo value);
};
```

其中使用到的数据结构在如下文件中定义：

* `hardware/interfaces/health/2.0/types.hal`

```cpp
package android.hardware.health@2.0;

import @1.0::HealthInfo;
import @1.0::Result;

/**
 * Status values for HAL methods.
 */
enum Result : @1.0::Result {
    NOT_FOUND,
    CALLBACK_DIED,
};

......

struct HealthInfo { // 2.0版本的HealthInfo数据结构
    /**
     * 保留了V1.0 HealthInfo的数据内容，但增加了存储相关的信息
     */
    @1.0::HealthInfo legacy;
    /**
     * Average battery current in uA. Will be 0 if unsupported.
     */
    int32_t batteryCurrentAverage;
    /**
     * Disk Statistics. Will be an empty vector if unsupported.
     */
    vec<DiskStats> diskStats;
    /**
     * Information on storage devices. Will be an empty vector if
     * unsupported.
     */
    vec<StorageInfo> storageInfos;
};
```

其中V1.0用于传输的`HealthInfo`数据结构定义如下：

* `hardware/interfaces/health/1.0/types.hal`

```java
/**
 * The parameter to healthd mainloop update calls
 */
struct HealthInfo {
    /** AC charger state - 'true' if online */
    bool chargerAcOnline;

    /** USB charger state - 'true' if online */
    bool chargerUsbOnline;

    /** Wireless charger state - 'true' if online */
    bool chargerWirelessOnline;

    /** Maximum charging current supported by charger in uA */
    int32_t maxChargingCurrent;

    /** Maximum charging voltage supported by charger in uV */
    int32_t maxChargingVoltage;

    BatteryStatus batteryStatus;

    BatteryHealth batteryHealth;

    /** 'true' if battery is present */
    bool batteryPresent;

    /** Remaining battery capacity in percent */
    int32_t batteryLevel;

    /**
     * Instantaneous battery voltage in millivolts (mV).
     *
     * Historically, the unit of this field is microvolts (uV), but all
     * clients and implementations uses millivolts in practice, making it
     * the de-facto standard.
     */
    int32_t batteryVoltage;

    /** Instantaneous battery temperature in tenths of degree celcius */
    int32_t batteryTemperature;

    /** Instantaneous battery current in uA */
    int32_t batteryCurrent;

    /** Battery charge cycle count */
    int32_t batteryCycleCount;

    /** Battery charge value when it is considered to be "full" in uA-h */
    int32_t batteryFullCharge;

    /** Instantaneous battery capacity in uA-h */
    int32_t batteryChargeCounter;

    /** Battery technology, e.g. "Li-ion, Li-Poly" etc. */
    string batteryTechnology;
};
```

Google提供了IHealth服务的对应默认实现，如果OEM厂商没有专门的板卡实现的话会使用该默认的实现：

* `hardware/interfaces/health/2.0/default/Android.bp`

```
cc_defaults { // 系统使用的默认实现
    name: "android.hardware.health@2.0-impl_defaults",

    recovery_available: true,
    cflags: [ ...... ],

    shared_libs: [
        "libbase",
        "libhidlbase",
        "libhidltransport",
        "libhwbinder",
        ......
        "android.hardware.health@2.0",
    ],

    static_libs: [
        "libbatterymonitor", // 依赖于libbatterymonitor的静态链接库
        "android.hardware.health@1.0-convert",
    ],
}

cc_library_static {
    name: "android.hardware.health@2.0-impl",
    defaults: ["android.hardware.health@2.0-impl_defaults"],

    vendor_available: true,
    srcs: [
        "Health.cpp",
        "healthd_common.cpp",
    ],

    export_include_dirs: ["include"],
}

cc_library_shared {
    name: "android.hardware.health@2.0-impl-default",
    defaults: ["android.hardware.health@2.0-impl_defaults"],

    recovery_available: true,
    relative_install_path: "hw",

    static_libs: [
        "android.hardware.health@2.0-impl",
        "libhealthstoragedefault",
    ],

    srcs: [
        "HealthImplDefault.cpp",
    ],
}
```

### android.hardware.health@2.0-impl

根据`Android.bp`中的定义，如果OEM厂商没有IHealth对应的实现的话则会使用Google提供的默认实现`android.hardware.health@2.0-impl_defaults`，该实现依赖于`Health.cpp` 和 `healthd_common.cpp`这个两个源文件：

* `hardware/interfaces/health/2.0/default/Health.cpp`

```cpp
#define LOG_TAG "android.hardware.health@2.0-impl"
......

extern void healthd_battery_update_internal(bool);

namespace android {
namespace hardware {
namespace health {
namespace V2_0 {
namespace implementation {

sp<Health> Health::instance_; // 全局单例

Health::Health(struct healthd_config* c) {
    healthd_board_init(c);
    // 采用 unique_ptr的智能指针进行BatteryMonitor构造，并进行初始化
    battery_monitor_ = std::make_unique<BatteryMonitor>();
    battery_monitor_->init(c);
}

// 实现HIDL接口的回调注册方法
Return<Result> Health::registerCallback(const sp<IHealthInfoCallback>& callback) {
    if (callback == nullptr) {
        return Result::SUCCESS;
    }

    {
        std::lock_guard<decltype(callbacks_lock_)> lock(callbacks_lock_);
        // Health.h std::vector<sp<IHealthInfoCallback>> callbacks_;
        callbacks_.push_back(callback); // 将上层回调放在一个全局vector中进行管理
        // unlock
    }

    auto linkRet = callback->linkToDeath(this, 0u /* cookie */);
    if (!linkRet.withDefault(false)) {
        LOG(WARNING) << __func__ << "Cannot link to death: "
                     << (linkRet.isOk() ? "linkToDeath returns false" : linkRet.description());
        // ignore the error
    }

    return updateAndNotify(callback);
}

// 实现HIDL接口的回调注销方法
bool Health::unregisterCallbackInternal(const sp<IBase>& callback) {
    if (callback == nullptr) return false;

    bool removed = false;
    std::lock_guard<decltype(callbacks_lock_)> lock(callbacks_lock_);
    for (auto it = callbacks_.begin(); it != callbacks_.end();) {
        if (interfacesEqual(*it, callback)) {
            it = callbacks_.erase(it); // 从vector中移除对应的上层回调
            removed = true;
        } else {
            ++it;
        }
    }
    (void)callback->unlinkToDeath(this).isOk();  // ignore errors
    return removed;
}

Return<Result> Health::unregisterCallback(const sp<IHealthInfoCallback>& callback) {
    return unregisterCallbackInternal(callback) ? Result::SUCCESS : Result::NOT_FOUND;
}

// 样本方法，用于统一处理HIDL中定义的接口回调
template <typename T>
void getProperty(const std::unique_ptr<BatteryMonitor>& monitor, int id, T defaultValue,
                 const std::function<void(Result, T)>& callback) {
    struct BatteryProperty prop; // 准备用于获取电池属性的存储结构
    T ret = defaultValue;
    Result result = Result::SUCCESS;
    // 调用BatteryMonitor的getProperty方法获取属性数据
    status_t err = monitor->getProperty(static_cast<int>(id), &prop);
    if (err != OK) {
        LOG(DEBUG) << "getProperty(" << id << ")"
                   << " fails: (" << err << ") " << strerror(-err);
    } else {
        ret = static_cast<T>(prop.valueInt64);
    }
    switch (err) {
        case OK:
            result = Result::SUCCESS;
            break;
        case NAME_NOT_FOUND:
            result = Result::NOT_SUPPORTED;
            break;
        default:
            result = Result::UNKNOWN;
            break;
    }
    callback(result, static_cast<T>(ret)); // 调用HIDL接口中定义的回调方法如： getChargeCounter_cb
}

/* 通过BatteryMonitor获取电池健康的状态方法 begin*/
Return<void> Health::getChargeCounter(getChargeCounter_cb _hidl_cb) {
    getProperty<int32_t>(battery_monitor_, BATTERY_PROP_CHARGE_COUNTER, 0, _hidl_cb);
    return Void();
}

Return<void> Health::getCurrentNow(getCurrentNow_cb _hidl_cb) {......}

Return<void> Health::getCurrentAverage(getCurrentAverage_cb _hidl_cb) {......}

Return<void> Health::getCapacity(getCapacity_cb _hidl_cb) {......}

Return<void> Health::getEnergyCounter(getEnergyCounter_cb _hidl_cb) {......}

Return<void> Health::getChargeStatus(getChargeStatus_cb _hidl_cb) {......}
/* 获取电池健康的状态方法 end*/

Return<Result> Health::update() {
    if (!healthd_mode_ops || !healthd_mode_ops->battery_update) {
        LOG(WARNING) << "health@2.0: update: not initialized. "
                     << "update() should not be called in charger";
        return Result::UNKNOWN;
    }

    // 调用BatteryMonitor中的update方法进行数据更新
    bool chargerOnline = battery_monitor_->update();

    // 调整 uevent / wakealarm的数据采样间隔
    healthd_battery_update_internal(chargerOnline);

    return Result::SUCCESS;
}

Return<Result> Health::updateAndNotify(const sp<IHealthInfoCallback>& callback) {
    std::lock_guard<decltype(callbacks_lock_)> lock(callbacks_lock_);
    std::vector<sp<IHealthInfoCallback>> storedCallbacks{std::move(callbacks_)};
    callbacks_.clear();
    if (callback != nullptr) {
        callbacks_.push_back(callback);
    }
    Return<Result> result = update();
    callbacks_ = std::move(storedCallbacks);
    return result;
}

void Health::notifyListeners(HealthInfo* healthInfo) {
    std::vector<StorageInfo> info;
    get_storage_info(info);

    std::vector<DiskStats> stats;
    get_disk_stats(stats);

    int32_t currentAvg = 0;

    struct BatteryProperty prop;
    status_t ret = battery_monitor_->getProperty(BATTERY_PROP_CURRENT_AVG, &prop);
    if (ret == OK) {
        currentAvg = static_cast<int32_t>(prop.valueInt64);
    }

    healthInfo->batteryCurrentAverage = currentAvg;
    healthInfo->diskStats = stats;
    healthInfo->storageInfos = info;

    std::lock_guard<decltype(callbacks_lock_)> lock(callbacks_lock_);
    for (auto it = callbacks_.begin(); it != callbacks_.end();) {
        // 调用上层回调接口中的healthInfoChanged()方法，将数据传递给上层，通知更新
        auto ret = (*it)->healthInfoChanged(*healthInfo);
        if (!ret.isOk() && ret.isDeadObject()) {
            it = callbacks_.erase(it);
        } else {
            ++it;
        }
    }
}

......

// HIDL接口中的getHealthInfo方法实现
Return<void> Health::getHealthInfo(getHealthInfo_cb _hidl_cb) {
    using android::hardware::health::V1_0::hal_conversion::convertToHealthInfo;

    updateAndNotify(nullptr);
    struct android::BatteryProperties p = getBatteryProperties(battery_monitor_.get());

    // 1.0 2.0兼容处理，将V1.0的数据转换为2.0数据
    V1_0::HealthInfo batteryInfo;
    convertToHealthInfo(&p, batteryInfo);
    ......
    int32_t currentAvg = 0;

    struct BatteryProperty prop;
    // 调用BatteryMonitor的getProperty方法获取电池健康信息
    status_t ret = battery_monitor_->getProperty(BATTERY_PROP_CURRENT_AVG, &prop);
    if (ret == OK) {
        currentAvg = static_cast<int32_t>(prop.valueInt64);
    }

    V2_0::HealthInfo healthInfo = {};
    healthInfo.legacy = std::move(batteryInfo);
    healthInfo.batteryCurrentAverage = currentAvg;
    healthInfo.diskStats = stats;
    healthInfo.storageInfos = info;

    _hidl_cb(Result::SUCCESS, healthInfo); // 通知上层回调处理healthInfo数据
    return Void();
}

void Health::serviceDied(uint64_t /* cookie */, const wp<IBase>& who) {
    (void)unregisterCallbackInternal(who.promote());
}

// 初始化全局单例
sp<IHealth> Health::initInstance(struct healthd_config* c) {
    if (instance_ == nullptr) {
        instance_ = new Health(c);
    }
    return instance_;
}

sp<Health> Health::getImplementation() {
    CHECK(instance_ != nullptr);
    return instance_;
}

}  // namespace implementation
}  // namespace V2_0
}  // namespace health
}  // namespace hardware
}  // namespace android
```

`Health.cpp`是`IHealth`的实现，将实际实现都委托给了`BatteryMonitor.cpp`主要为了兼容V1.0的旧版本。

* `hardware/interfaces/health/2.0/default/healthd_common.cpp`

```cpp
#define LOG_TAG "android.hardware.health@2.0-impl"
#define KLOG_LEVEL 6

#include <healthd/BatteryMonitor.h>
#include <healthd/healthd.h>

#include <batteryservice/BatteryService.h>
......
#include <health2/Health.h>

using namespace android;

// 当运行在快速模式（充电模式）下时1分钟更新一次数据
#define DEFAULT_PERIODIC_CHORES_INTERVAL_FAST (60 * 1)
// 慢速模式（电池供电）下10分钟更新一次数据
#define DEFAULT_PERIODIC_CHORES_INTERVAL_SLOW (60 * 10)

static struct healthd_config healthd_config = {
    .periodic_chores_interval_fast = DEFAULT_PERIODIC_CHORES_INTERVAL_FAST,
    .periodic_chores_interval_slow = DEFAULT_PERIODIC_CHORES_INTERVAL_SLOW,
    .batteryStatusPath = String8(String8::kEmptyString),
    .batteryHealthPath = String8(String8::kEmptyString),
    .batteryPresentPath = String8(String8::kEmptyString),
    .batteryCapacityPath = String8(String8::kEmptyString),
    .batteryVoltagePath = String8(String8::kEmptyString),
    .batteryTemperaturePath = String8(String8::kEmptyString),
    .batteryTechnologyPath = String8(String8::kEmptyString),
    .batteryCurrentNowPath = String8(String8::kEmptyString),
    .batteryCurrentAvgPath = String8(String8::kEmptyString),
    .batteryChargeCounterPath = String8(String8::kEmptyString),
    .batteryFullChargePath = String8(String8::kEmptyString),
    .batteryCycleCountPath = String8(String8::kEmptyString),
    .energyCounter = NULL,
    .boot_min_cap = 0,
    .screen_on = NULL,
};

static int eventct; // 全局事件计数器
static int epollfd; // 全局的epollfd，在init时由epoll_create创建

#define POWER_SUPPLY_SUBSYSTEM "power_supply"

static int uevent_fd;
static int wakealarm_fd;

// -1 for no epoll timeout
static int awake_poll_interval = -1;

static int wakealarm_wake_interval = DEFAULT_PERIODIC_CHORES_INTERVAL_FAST;

using ::android::hardware::health::V2_0::implementation::Health;

struct healthd_mode_ops* healthd_mode_ops = nullptr;

int healthd_register_event(int fd, void (*handler)(uint32_t), EventWakeup wakeup) {
    struct epoll_event ev;

    ev.events = EPOLLIN;

    if (wakeup == EVENT_WAKEUP_FD) ev.events |= EPOLLWAKEUP;

    ev.data.ptr = (void*)handler;
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &ev) == -1) { // 注册epollfd到epoll中
        KLOG_ERROR(LOG_TAG, "epoll_ctl failed; errno=%d\n", errno);
        return -1;
    }

    eventct++; // 记录注册成功 fd 的总数
    return 0;
}

// 配置唤醒更新数据的时间间隔
static void wakealarm_set_interval(int interval) {
    struct itimerspec itval;

    if (wakealarm_fd == -1) return;

    wakealarm_wake_interval = interval;

    if (interval == -1) interval = 0; // 关闭定时唤醒

    itval.it_interval.tv_sec = interval;
    itval.it_interval.tv_nsec = 0;
    itval.it_value.tv_sec = interval;
    itval.it_value.tv_nsec = 0;

    if (timerfd_settime(wakealarm_fd, 0, &itval, NULL) == -1)// 开启或关闭定时器
        KLOG_ERROR(LOG_TAG, "wakealarm_set_interval: timerfd_settime failed\n");
}

// 更新数据的采样间隔
void healthd_battery_update_internal(bool charger_online) {
    // 判断是否充电状态，充电状态下更新间隔默认1分钟，电池状态下更新间隔默认10分钟
    int new_wake_interval = charger_online ? healthd_config.periodic_chores_interval_fast
                                           : healthd_config.periodic_chores_interval_slow;

    if (new_wake_interval != wakealarm_wake_interval) wakealarm_set_interval(new_wake_interval);

    if (healthd_config.periodic_chores_interval_fast == -1) // 关闭定时唤醒
        awake_poll_interval = -1;
    else
        awake_poll_interval = new_wake_interval == healthd_config.periodic_chores_interval_fast
                                  ? -1
                                  : healthd_config.periodic_chores_interval_fast * 1000;
}

static void healthd_battery_update(void) {
    Health::getImplementation()->update();
}

static void periodic_chores() {
    healthd_battery_update();
}

#define UEVENT_MSG_LEN 2048
static void uevent_event(uint32_t /*epevents*/) {
    char msg[UEVENT_MSG_LEN + 2];
    char* cp;
    int n;

    // 从kernel接收 uevent 消息
    n = uevent_kernel_multicast_recv(uevent_fd, msg, UEVENT_MSG_LEN);
    if (n <= 0) return;
    if (n >= UEVENT_MSG_LEN) /* overflow -- discard */
        return;

    msg[n] = '\0';
    msg[n + 1] = '\0';
    cp = msg;

    // 查找消息中是否存在 SUBSYSTEM=power_supply 字符串
    while (*cp) {
        if (!strcmp(cp, "SUBSYSTEM=" POWER_SUPPLY_SUBSYSTEM)) {
            healthd_battery_update();// 触发更新电池信息
            break;
        }

        /* advance to after the next \0 */
        while (*cp++)
            ;
    }
}

// uevent初始化
static void uevent_init(void) {
    // 创建 netlink socket，用于监听 uevent 事件，返回 socket 文件描述符
    uevent_fd = uevent_open_socket(64 * 1024, true);

    if (uevent_fd < 0) {
        KLOG_ERROR(LOG_TAG, "uevent_init: uevent_open_socket failed\n");
        return;
    }

    fcntl(uevent_fd, F_SETFL, O_NONBLOCK);// 调用 fcntl 设置文件描述符为非阻塞模式。
    // 注册 uevent 事件处理相关
    if (healthd_register_event(uevent_fd, uevent_event, EVENT_WAKEUP_FD))
        KLOG_ERROR(LOG_TAG, "register for uevent events failed\n");
}

static void wakealarm_event(uint32_t /*epevents*/) {
    unsigned long long wakeups;

    if (read(wakealarm_fd, &wakeups, sizeof(wakeups)) == -1) {
        KLOG_ERROR(LOG_TAG, "wakealarm_event: read wakealarm fd failed\n");
        return;
    }

    periodic_chores();
}

static void wakealarm_init(void) {
    wakealarm_fd = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK);
    if (wakealarm_fd == -1) {
        KLOG_ERROR(LOG_TAG, "wakealarm_init: timerfd_create failed\n");
        return;
    }

    if (healthd_register_event(wakealarm_fd, wakealarm_event, EVENT_WAKEUP_FD))
        KLOG_ERROR(LOG_TAG, "Registration of wakealarm event failed\n");

    wakealarm_set_interval(healthd_config.periodic_chores_interval_fast);
}

static void healthd_mainloop(void) {
    int nevents = 0;
    while (1) {
        struct epoll_event events[eventct];
        int timeout = awake_poll_interval;
        int mode_timeout;

        /* Don't wait for first timer timeout to run periodic chores */
        if (!nevents) periodic_chores();

        healthd_mode_ops->heartbeat();

        mode_timeout = healthd_mode_ops->preparetowait();
        if (timeout < 0 || (mode_timeout > 0 && mode_timeout < timeout)) timeout = mode_timeout;
        nevents = epoll_wait(epollfd, events, eventct, timeout); // 等待epoll调度，并将事件进行记录
        if (nevents == -1) {
            if (errno == EINTR) continue;
            KLOG_ERROR(LOG_TAG, "healthd_mainloop: epoll_wait failed\n");
            break;
        }

        for (int n = 0; n < nevents; ++n) {
            // 调用对应的事件处理方法进行事件处理
            if (events[n].data.ptr) (*(void (*)(int))events[n].data.ptr)(events[n].events);
        }
    }

    return;
}

static int healthd_init() {
    epollfd = epoll_create1(EPOLL_CLOEXEC); // 创建epoll句柄，获取epoll的fd用于事件调度
    if (epollfd == -1) {
        KLOG_ERROR(LOG_TAG, "epoll_create1 failed; errno=%d\n", errno);
        return -1;
    }

    healthd_mode_ops->init(&healthd_config); // 初始化相关回调
    wakealarm_init(); // 初始化wakealarm
    uevent_init(); // 初始化uevent

    return 0;
}

int healthd_main() {
    int ret;

    klog_set_level(KLOG_LEVEL);// 配置日志等级

    if (!healthd_mode_ops) {
        KLOG_ERROR("healthd ops not set, exiting\n");
        exit(1);
    }

    ret = healthd_init(); // 初始化healthd
    if (ret) {
        KLOG_ERROR("Initialization failed, exiting\n");
        exit(2);
    }

    healthd_mainloop(); // 进入主循环
    KLOG_ERROR("Main loop terminated, exiting\n");
    return 3;
}
```

### libbatterymonitor

在上面的分析中我们看到`android.hardware.health@2.0-impl_defaults`依赖于libbatterymonitor的静态链接库，而`libbatterymonitor`对应的代码实现在`system/core/healthd/BatteryMonitor.cpp`中：

* `system/core/healthd/Android.bp`

```
......
cc_library_static {
    name: "libbatterymonitor",
    srcs: ["BatteryMonitor.cpp"],
    cflags: ["-Wall", "-Werror"],
    vendor_available: true,
    recovery_available: true,
    export_include_dirs: ["include"],
    shared_libs: [
        "libutils",
        "libbase",
    ],
    header_libs: ["libhealthd_headers"],
    export_header_lib_headers: ["libhealthd_headers"],
}
......
```

其实，`BatteryMonitor`提供了监控`/sys/class/power_supply`下的文件节点的方法，这些class节点由内核提供数据，以全志A133板卡搭载AXP707 PMIC为例，kernel生成的节点有如下三个，分别对应`AC电源供电`、`电池供电`、`USB供电`：

```
ssp:/sys/class/power_supply # ls -la
total 0
drwxr-xr-x  2 root root 0 1970-01-01 11:46 .
drwxr-xr-x 54 root root 0 1970-01-01 11:46 ..
lrwxrwxrwx  1 root root 0 1970-01-01 11:48 axp803-ac -> ../../devices/platform/soc/7081400.s_twi/i2c-6/6-0034/axp803-ac-power-supply.0/power_supply/axp803-ac
lrwxrwxrwx  1 root root 0 1970-01-01 11:48 axp803-battery -> ../../devices/platform/soc/7081400.s_twi/i2c-6/6-0034/axp803-battery-power-supply.0/power_supply/axp803-battery
lrwxrwxrwx  1 root root 0 1970-01-01 11:48 axp803-usb -> ../../devices/platform/soc/7081400.s_twi/i2c-6/6-0034/axp803-usb-power-supply.0/power_supply/axp803-usb

ssp:/sys/class/power_supply # ls -la axp803-ac/
total 0
drwxr-xr-x 3 root root    0 1970-01-01 11:46 .
drwxr-xr-x 3 root root    0 1970-01-01 11:46 ..
lrwxrwxrwx 1 root root    0 2022-07-04 16:49 device -> ../../../axp803-ac-power-supply.0
-r--r--r-- 1 root root 4096 2022-07-04 16:49 model_name
-r--r--r-- 1 root root 4096 1970-01-01 11:48 online
drwxr-xr-x 2 root root    0 1970-01-01 11:46 power
-r--r--r-- 1 root root 4096 2022-07-04 16:49 present
lrwxrwxrwx 1 root root    0 2022-07-04 16:49 subsystem -> ../../../../../../../../../class/power_supply
-r--r--r-- 1 root root 4096 1970-01-01 11:48 type
-rw-r--r-- 1 root root 4096 1970-01-01 11:46 uevent

ssp:/sys/class/power_supply # ls -la axp803-battery/
total 0
drwxr-xr-x 3 root root    0 1970-01-01 11:46 .
drwxr-xr-x 3 root root    0 1970-01-01 11:46 ..
-r--r--r-- 1 root root 4096 1970-01-01 11:48 capacity
-r--r--r-- 1 root root 4096 1970-01-01 11:48 charge_counter
-r--r--r-- 1 root root 4096 1970-01-01 11:48 charge_full
-r--r--r-- 1 root root 4096 1970-01-01 11:48 current_now
lrwxrwxrwx 1 root root    0 2022-07-04 15:45 device -> ../../../axp803-battery-power-supply.0
-r--r--r-- 1 root root 4096 1970-01-01 11:48 health
-r--r--r-- 1 root root 4096 2022-07-04 15:45 model_name
-r--r--r-- 1 root root 4096 2022-07-04 15:45 online
drwxr-xr-x 2 root root    0 1970-01-01 11:46 power
-r--r--r-- 1 root root 4096 1970-01-01 11:48 present
-r--r--r-- 1 root root 4096 1970-01-01 11:48 status
lrwxrwxrwx 1 root root    0 2022-07-04 15:45 subsystem -> ../../../../../../../../../class/power_supply
-r--r--r-- 1 root root 4096 1970-01-01 11:48 technology
-r--r--r-- 1 root root 4096 1970-01-01 11:48 temp
-r--r--r-- 1 root root 4096 1970-01-01 11:48 type
-rw-r--r-- 1 root root 4096 1970-01-01 11:46 uevent
-r--r--r-- 1 root root 4096 2022-07-04 15:45 voltage_max_design
-r--r--r-- 1 root root 4096 2022-07-04 15:45 voltage_min_design
-r--r--r-- 1 root root 4096 1970-01-01 11:48 voltage_now

ssp:/sys/class/power_supply # ls -la axp803-usb/
total 0
drwxr-xr-x 3 root root    0 1970-01-01 11:46 .
drwxr-xr-x 3 root root    0 1970-01-01 11:46 ..
lrwxrwxrwx 1 root root    0 2022-07-04 17:11 device -> ../../../axp803-usb-power-supply.0
-rw-r--r-- 1 root root 4096 2022-07-04 17:11 input_current_limit
-r--r--r-- 1 root root 4096 2022-07-04 17:11 model_name
-r--r--r-- 1 root root 4096 1970-01-01 11:48 online
drwxr-xr-x 2 root root    0 1970-01-01 11:46 power
-r--r--r-- 1 root root 4096 2022-07-04 17:11 present
lrwxrwxrwx 1 root root    0 2022-07-04 17:11 subsystem -> ../../../../../../../../../class/power_supply
-r--r--r-- 1 root root 4096 1970-01-01 11:48 type
-rw-r--r-- 1 root root 4096 1970-01-01 11:46 uevent
```

* `system/core/healthd/BatteryMonitor.cpp`

```cpp
#include <healthd/healthd.h>
#include <healthd/BatteryMonitor.h>
......
#define POWER_SUPPLY_SUBSYSTEM "power_supply"
#define POWER_SUPPLY_SYSFS_PATH "/sys/class/" POWER_SUPPLY_SUBSYSTEM
#define FAKE_BATTERY_CAPACITY 42
#define FAKE_BATTERY_TEMPERATURE 424
#define MILLION 1.0e6
#define DEFAULT_VBUS_VOLTAGE 5000000

namespace android {
......
// 电池健康信息的默认配置
static void initBatteryProperties(BatteryProperties* props) {
    props->chargerAcOnline = false;
    props->chargerUsbOnline = false;
    props->chargerWirelessOnline = false;
    props->maxChargingCurrent = 0;
    props->maxChargingVoltage = 0;
    props->batteryStatus = BATTERY_STATUS_UNKNOWN;
    props->batteryHealth = BATTERY_HEALTH_UNKNOWN;
    props->batteryPresent = false;
    props->batteryLevel = 0;
    props->batteryVoltage = 0;
    props->batteryTemperature = 0;
    props->batteryCurrent = 0;
    props->batteryCycleCount = 0;
    props->batteryFullCharge = 0;
    props->batteryChargeCounter = 0;
    props->batteryTechnology.clear();
}

// 构造函数
BatteryMonitor::BatteryMonitor()
    : mHealthdConfig(nullptr),
      mBatteryDevicePresent(false),
      mBatteryFixedCapacity(0),
      mBatteryFixedTemperature(0) {
    initBatteryProperties(&props);
}

struct BatteryProperties getBatteryProperties(BatteryMonitor* batteryMonitor) {
    return batteryMonitor->props;
}

// 根据sys节点下的数据解析充电状态
int BatteryMonitor::getBatteryStatus(const char* status) {
    int ret;
    struct sysfsStringEnumMap batteryStatusMap[] = {
        { "Unknown", BATTERY_STATUS_UNKNOWN },
        { "Charging", BATTERY_STATUS_CHARGING },
        { "Discharging", BATTERY_STATUS_DISCHARGING },
        { "Not charging", BATTERY_STATUS_NOT_CHARGING },
        { "Full", BATTERY_STATUS_FULL },
        { NULL, 0 },
    };

    ret = mapSysfsString(status, batteryStatusMap);
    if (ret < 0) {
        KLOG_WARNING(LOG_TAG, "Unknown battery status '%s'\n", status);
        ret = BATTERY_STATUS_UNKNOWN;
    }

    return ret;
}

// 根据sys节点下的数据解析电池健康信息
int BatteryMonitor::getBatteryHealth(const char* status) {
    int ret;
    struct sysfsStringEnumMap batteryHealthMap[] = {
        { "Unknown", BATTERY_HEALTH_UNKNOWN },
        { "Good", BATTERY_HEALTH_GOOD },
        { "Overheat", BATTERY_HEALTH_OVERHEAT },
        { "Dead", BATTERY_HEALTH_DEAD },
        { "Over voltage", BATTERY_HEALTH_OVER_VOLTAGE },
        { "Unspecified failure", BATTERY_HEALTH_UNSPECIFIED_FAILURE },
        { "Cold", BATTERY_HEALTH_COLD },
        // battery health values from JEITA spec
        { "Warm", BATTERY_HEALTH_GOOD },
        { "Cool", BATTERY_HEALTH_GOOD },
        { "Hot", BATTERY_HEALTH_OVERHEAT },
        { NULL, 0 },
    };

    ret = mapSysfsString(status, batteryHealthMap);
    if (ret < 0) {
        KLOG_WARNING(LOG_TAG, "Unknown battery health '%s'\n", status);
        ret = BATTERY_HEALTH_UNKNOWN;
    }

    return ret;
}
......

// 从sysfs节点获取支持的充电类型
BatteryMonitor::PowerSupplyType BatteryMonitor::readPowerSupplyType(const String8& path) {
    std::string buf;
    int ret;
    struct sysfsStringEnumMap supplyTypeMap[] = {
            { "Unknown", ANDROID_POWER_SUPPLY_TYPE_UNKNOWN },
            { "Battery", ANDROID_POWER_SUPPLY_TYPE_BATTERY },
            { "UPS", ANDROID_POWER_SUPPLY_TYPE_AC },
            { "Mains", ANDROID_POWER_SUPPLY_TYPE_AC },
            { "USB", ANDROID_POWER_SUPPLY_TYPE_USB },
            { "USB_DCP", ANDROID_POWER_SUPPLY_TYPE_AC },
            { "USB_HVDCP", ANDROID_POWER_SUPPLY_TYPE_AC },
            { "USB_CDP", ANDROID_POWER_SUPPLY_TYPE_AC },
            { "USB_ACA", ANDROID_POWER_SUPPLY_TYPE_AC },
            { "USB_C", ANDROID_POWER_SUPPLY_TYPE_AC },
            { "USB_PD", ANDROID_POWER_SUPPLY_TYPE_AC },
            { "USB_PD_DRP", ANDROID_POWER_SUPPLY_TYPE_USB },
            { "Wireless", ANDROID_POWER_SUPPLY_TYPE_WIRELESS },
            { NULL, 0 },
    };

    if (readFromFile(path, &buf) <= 0)
        return ANDROID_POWER_SUPPLY_TYPE_UNKNOWN;

    ret = mapSysfsString(buf.c_str(), supplyTypeMap);
    if (ret < 0) {
        KLOG_WARNING(LOG_TAG, "Unknown power supply type '%s'\n", buf.c_str());
        ret = ANDROID_POWER_SUPPLY_TYPE_UNKNOWN;
    }

    return static_cast<BatteryMonitor::PowerSupplyType>(ret);
}

......

// 从sys/class/prower_supply文件节点获取电池状态信息更新
bool BatteryMonitor::update(void) {
    bool logthis;

    initBatteryProperties(&props); // 先清除历史旧数据

    if (!mHealthdConfig->batteryPresentPath.isEmpty())
        props.batteryPresent = getBooleanField(mHealthdConfig->batteryPresentPath);
    else
        props.batteryPresent = mBatteryDevicePresent;

    props.batteryLevel = mBatteryFixedCapacity ?
        mBatteryFixedCapacity :
        getIntField(mHealthdConfig->batteryCapacityPath);
    props.batteryVoltage = getIntField(mHealthdConfig->batteryVoltagePath) / 1000;

    if (!mHealthdConfig->batteryCurrentNowPath.isEmpty())
        props.batteryCurrent = getIntField(mHealthdConfig->batteryCurrentNowPath) / 1000;

    if (!mHealthdConfig->batteryFullChargePath.isEmpty())
        props.batteryFullCharge = getIntField(mHealthdConfig->batteryFullChargePath);

    if (!mHealthdConfig->batteryCycleCountPath.isEmpty())
        props.batteryCycleCount = getIntField(mHealthdConfig->batteryCycleCountPath);

    if (!mHealthdConfig->batteryChargeCounterPath.isEmpty())
        props.batteryChargeCounter = getIntField(mHealthdConfig->batteryChargeCounterPath);

    props.batteryTemperature = mBatteryFixedTemperature ?
        mBatteryFixedTemperature :
        getIntField(mHealthdConfig->batteryTemperaturePath);

    std::string buf;

    if (readFromFile(mHealthdConfig->batteryStatusPath, &buf) > 0)
        props.batteryStatus = getBatteryStatus(buf.c_str());

    if (readFromFile(mHealthdConfig->batteryHealthPath, &buf) > 0)
        props.batteryHealth = getBatteryHealth(buf.c_str());

    if (readFromFile(mHealthdConfig->batteryTechnologyPath, &buf) > 0)
        props.batteryTechnology = String8(buf.c_str());

    double MaxPower = 0;

    for (size_t i = 0; i < mChargerNames.size(); i++) {
        String8 path;
        path.appendFormat("%s/%s/online", POWER_SUPPLY_SYSFS_PATH,
                          mChargerNames[i].string());
        if (getIntField(path)) {
            path.clear();
            path.appendFormat("%s/%s/type", POWER_SUPPLY_SYSFS_PATH,
                              mChargerNames[i].string());
            switch(readPowerSupplyType(path)) {
            case ANDROID_POWER_SUPPLY_TYPE_AC:
                props.chargerAcOnline = true;
                break;
            case ANDROID_POWER_SUPPLY_TYPE_USB:
                props.chargerUsbOnline = true;
                break;
            case ANDROID_POWER_SUPPLY_TYPE_WIRELESS:
                props.chargerWirelessOnline = true;
                break;
            default:
                KLOG_WARNING(LOG_TAG, "%s: Unknown power supply type\n",
                             mChargerNames[i].string());
            }
            path.clear();
            path.appendFormat("%s/%s/current_max", POWER_SUPPLY_SYSFS_PATH,
                              mChargerNames[i].string());
            int ChargingCurrent =
                    (access(path.string(), R_OK) == 0) ? getIntField(path) : 0;

            path.clear();
            path.appendFormat("%s/%s/voltage_max", POWER_SUPPLY_SYSFS_PATH,
                              mChargerNames[i].string());

            int ChargingVoltage =
                (access(path.string(), R_OK) == 0) ? getIntField(path) :
                DEFAULT_VBUS_VOLTAGE;

            double power = ((double)ChargingCurrent / MILLION) *
                           ((double)ChargingVoltage / MILLION);
            if (MaxPower < power) {
                props.maxChargingCurrent = ChargingCurrent;
                props.maxChargingVoltage = ChargingVoltage;
                MaxPower = power;
            }
        }
    }

    logthis = !healthd_board_battery_update(&props);

    if (logthis) {
        char dmesgline[256];
        size_t len;
        if (prop.batteryPresent) {
            snprint(dmesgline, sizeof(dmesgline),
                "battery l=%d v=%d t=%s%d.%d h=%d st=%d",
                props.batteryLevel, props.batteryVoltage,
                props.batteryTemperature < 0 ? "-" : "",
                abs(props.batteryTemperature / 10),
                abs(props.batteryTemperature % 10), props.batteryHealth,
                props.batteryStatus);
            ......
        }
        ......
        // 将更新的电池数据信息打印到内核日志中，tag为healthd
        KLOG_WARNING(LOG_TAG, "%s\n", dmesgline);
    }
    ......

    healthd_mode_ops->battery_update(&props); // 调用ops回调更新电池健康参数prop
    return props.chargerAcOnline | props.chargerUsbOnline |
            props.chargerWirelessOnline;
}

......
// 从sysfs获取BatteryProperty
status_t BatteryMonitor::getProperty(int id, struct BatteryProperty *val) {
    status_t ret = BAD_VALUE;
    std::string buf;

    val->valueInt64 = LONG_MIN;

    switch(id) {
    case BATTERY_PROP_CHARGE_COUNTER:
        if (!mHealthdConfig->batteryChargeCounterPath.isEmpty()) {
            val->valueInt64 =
                getIntField(mHealthdConfig->batteryChargeCounterPath);
            ret = OK;
        } else {
            ret = NAME_NOT_FOUND;
        }
        break;

    case BATTERY_PROP_CURRENT_NOW:
        if (!mHealthdConfig->batteryCurrentNowPath.isEmpty()) {
            val->valueInt64 =
                getIntField(mHealthdConfig->batteryCurrentNowPath);
            ret = OK;
        } else {
            ret = NAME_NOT_FOUND;
        }
        break;

    case BATTERY_PROP_CURRENT_AVG:
        if (!mHealthdConfig->batteryCurrentAvgPath.isEmpty()) {
            val->valueInt64 =
                getIntField(mHealthdConfig->batteryCurrentAvgPath);
            ret = OK;
        } else {
            ret = NAME_NOT_FOUND;
        }
        break;

    case BATTERY_PROP_CAPACITY:
        if (!mHealthdConfig->batteryCapacityPath.isEmpty()) {
            val->valueInt64 =
                getIntField(mHealthdConfig->batteryCapacityPath);
            ret = OK;
        } else {
            ret = NAME_NOT_FOUND;
        }
        break;

    case BATTERY_PROP_ENERGY_COUNTER:
        if (mHealthdConfig->energyCounter) {
            ret = mHealthdConfig->energyCounter(&val->valueInt64);
        } else {
            ret = NAME_NOT_FOUND;
        }
        break;

    case BATTERY_PROP_BATTERY_STATUS:
        val->valueInt64 = getChargeStatus();
        ret = OK;
        break;

    default:
        break;
    }

    return ret;
}

......

// 监控/sys/class/power_supply下的文件节点来获取电池数据
void BatteryMonitor::init(struct healthd_config *hc) {
    String8 path;
    char pval[PROPERTY_VALUE_MAX];

    mHealthdConfig = hc;
    std::unique_ptr<DIR, decltype(&closedir)> dir(opendir(POWER_SUPPLY_SYSFS_PATH), closedir);
    if (dir == NULL) {
        KLOG_ERROR(LOG_TAG, "Could not open %s\n", POWER_SUPPLY_SYSFS_PATH);
    } else {
        struct dirent* entry;

        while ((entry = readdir(dir.get()))) {
            const char* name = entry->d_name;
            std::vector<String8>::iterator itIgnoreName;

            if (!strcmp(name, ".") || !strcmp(name, ".."))
                continue;

            itIgnoreName = find(hc->ignorePowerSupplyNames.begin(),
                                hc->ignorePowerSupplyNames.end(), String8(name));
            if (itIgnoreName != hc->ignorePowerSupplyNames.end())
                continue;

            // 从sysfs子目录查找设备支持的充电类型
            path.clear();
            path.appendFormat("%s/%s/type", POWER_SUPPLY_SYSFS_PATH, name);
            switch(readPowerSupplyType(path)) {
            case ANDROID_POWER_SUPPLY_TYPE_AC:
            case ANDROID_POWER_SUPPLY_TYPE_USB:
            case ANDROID_POWER_SUPPLY_TYPE_WIRELESS:
                path.clear();
                path.appendFormat("%s/%s/online", POWER_SUPPLY_SYSFS_PATH, name);
                if (access(path.string(), R_OK) == 0)
                    mChargerNames.add(String8(name));
                break;
            // 使用电池供电的情况下的子系统文件节点解析，获取Healthd的配置文件
            case ANDROID_POWER_SUPPLY_TYPE_BATTERY:
                mBatteryDevicePresent = true;

                if (mHealthdConfig->batteryStatusPath.isEmpty()) {
                    ......
                    if (access(path, R_OK) == 0)
                        mHealthdConfig->batteryStatusPath = path;
                }

                if (mHealthdConfig->batteryHealthPath.isEmpty()) {
                    ......
                }
                ......
                if (mHealthdConfig->batteryTechnologyPath.isEmpty()) {
                    ......
                }
                break;

            case ANDROID_POWER_SUPPLY_TYPE_UNKNOWN:
                break;
            }
        }
    }

    // 如果设备本身没有搭载电池，只插AC电源供电，则关闭周期性数据更新
    if (!mBatteryDevicePresent) {
        KLOG_WARNING(LOG_TAG, "No battery devices found\n");
        hc->periodic_chores_interval_fast = -1;
        hc->periodic_chores_interval_slow = -1;
    } else { // 错误判断
        if (mHealthdConfig->batteryStatusPath.isEmpty())
            KLOG_WARNING(LOG_TAG, "BatteryStatusPath not found\n");
        if (mHealthdConfig->batteryHealthPath.isEmpty())
            KLOG_WARNING(LOG_TAG, "BatteryHealthPath not found\n");
        ......
    }

    if (property_get("ro.boot.fake_battery", pval, NULL) > 0
                                               && strtol(pval, NULL, 10) != 0) {
        mBatteryFixedCapacity = FAKE_BATTERY_CAPACITY;
        mBatteryFixedTemperature = FAKE_BATTERY_TEMPERATURE;
    }
}
}; // namespace android
```

### libhealthservice

分析完上面的两大依赖后，我们来看下`libhealthservice`这个静态库的依赖，这部分的默认实现在`hardware/interfaces/health/2.0/utils/libhealthservice`中：

* `hardware/interfaces/health/2.0/utils/libhealthservice/Android.bp`

```
cc_library_static {
    name: "libhealthservice",
    vendor_available: true,
    srcs: ["HealthServiceCommon.cpp"],

    export_include_dirs: ["include"],

    cflags: [ ...... ],
    shared_libs: [
        "android.hardware.health@2.0",
    ],
    static_libs: [
        "android.hardware.health@2.0-impl",
        "android.hardware.health@1.0-convert",
    ],
    export_static_lib_headers: [
        "android.hardware.health@1.0-convert",
    ],
    header_libs: ["libhealthd_headers"],
    export_header_lib_headers: ["libhealthd_headers"],
}
```

可以看到该依赖的源文件是`HealthServiceCommon.cpp`:

* `hardware/interfaces/health/2.0/utils/libhealthservice/HealthServiceCommon.cpp`

```cpp
#define LOG_TAG "health@2.0/"
......
#include <android/hardware/health/1.0/types.h>
#include <hal_conversion.h>
#include <health2/Health.h>
#include <health2/service.h>
#include <healthd/healthd.h>
#include <hidl/HidlTransportSupport.h>
#include <hwbinder/IPCThreadState.h>

using android::hardware::IPCThreadState;
using android::hardware::configureRpcThreadpool;
using android::hardware::handleTransportPoll;
using android::hardware::setupTransportPolling;
using android::hardware::health::V2_0::HealthInfo;
using android::hardware::health::V1_0::hal_conversion::convertToHealthInfo;
using android::hardware::health::V2_0::IHealth;
using android::hardware::health::V2_0::implementation::Health;

extern int healthd_main(void);

static int gBinderFd = -1;
static std::string gInstanceName;

static void binder_event(uint32_t /*epevents*/) {
    if (gBinderFd >= 0) handleTransportPoll(gBinderFd);
}

void healthd_mode_service_2_0_init(struct healthd_config* config) {
    LOG(INFO) << LOG_TAG << gInstanceName << " Hal is starting up...";

    gBinderFd = setupTransportPolling();

    if (gBinderFd >= 0) {
        if (healthd_register_event(gBinderFd, binder_event))
            LOG(ERROR) << LOG_TAG << gInstanceName << ": Register for binder events failed";
    }

    android::sp<IHealth> service = Health::initInstance(config);
    CHECK_EQ(service->registerAsService(gInstanceName), android::OK)
        << LOG_TAG << gInstanceName << ": Failed to register HAL";

    LOG(INFO) << LOG_TAG << gInstanceName << ": Hal init done";
}

int healthd_mode_service_2_0_preparetowait(void) {
    IPCThreadState::self()->flushCommands();
    return -1;
}

void healthd_mode_service_2_0_heartbeat(void) {
    // noop
}

void healthd_mode_service_2_0_battery_update(struct android::BatteryProperties* prop) {
    HealthInfo info;
    convertToHealthInfo(prop, info.legacy);
    Health::getImplementation()->notifyListeners(&info);
}

// 配置service_2_0的操作回调函数
static struct healthd_mode_ops healthd_mode_service_2_0_ops = {
    .init = healthd_mode_service_2_0_init,
    .preparetowait = healthd_mode_service_2_0_preparetowait,
    .heartbeat = healthd_mode_service_2_0_heartbeat,
    .battery_update = healthd_mode_service_2_0_battery_update,
};

// 将Health-service2.0的实现标记为default
int health_service_main(const char* instance) {
    gInstanceName = instance;
    if (gInstanceName.empty()) {
        gInstanceName = "default";
    }
    healthd_mode_ops = &healthd_mode_service_2_0_ops;
    LOG(INFO) << LOG_TAG << gInstanceName << ": Hal starting main loop...";
    return healthd_main();
}
```

可以看到，实际在`health_service_main()`方法中调用了`healthd_main()`进行处理，但是将`healthd_mode_ops`的操作方法进行了更新，并标记服务的实现名称为`default`，也就是在`HealthServiceWrapper`中看到的`INSTANCE_HEALTHD`，而`healthd_main()`的默认实现在上面分析过的`hardware/interfaces/health/2.0/default/healthd_common.cpp`中，也就是调用healthd的主方法开始整个`health_2.0`服务。

### android.hardware.health@2.0-service

分析完整个service主要的依赖后我们来看下`android.hardware.health@2.0-service`的具体实现：

* `system/core/healthd/android.hardware.health@2.0-service.rc`

```
service health-hal-2-0 /vendor/bin/hw/android.hardware.health@2.0-service
    class hal
    user system
    group system
    capabilities WAKE_ALARM
    file /dev/kmsg w
```

可以看到，`android.hardware.health@2.0-service`是在init.rc中跟着`class hal`一起启动的，也就是在设备`boot`，由init进程调用启动，其`Android.bp`的定义如下：

* `system/core/healthd/Android.bp`

```
......
cc_defaults {
    name: "android.hardware.health@2.0-service_defaults",

    cflags: [ ...... ],

    static_libs: [
        "android.hardware.health@2.0-impl",
        "android.hardware.health@1.0-convert",
        "libhealthservice",
        "libhealthstoragedefault",
        "libbatterymonitor",
    ],

    shared_libs: [
        ......
        "android.hardware.health@2.0",
    ],
}

cc_binary {
    name: "android.hardware.health@2.0-service",
    defaults: ["android.hardware.health@2.0-service_defaults"],

    vendor: true,
    relative_install_path: "hw",
    init_rc: ["android.hardware.health@2.0-service.rc"],
    srcs: [
        "HealthServiceDefault.cpp",
    ],

    overrides: [ "healthd", ]
}
......
```

看到`android.hardware.health@2.0-service`默认实现的源文件为`HealthServiceDefault.cpp`:

* `system/core/healthd/HealthServiceDefault.cpp`

```cpp
#include <health2/service.h>
#include <healthd/healthd.h>

void healthd_board_init(struct healthd_config*) {
    // Implementation-defined init logic goes here.
    // 1. config->periodic_chores_interval_* variables
    // 2. config->battery*Path variables
    // 3. config->energyCounter. In this implementation, energyCounter is not defined.

    // use defaults
}

int healthd_board_battery_update(struct android::BatteryProperties*) {
    // 返回0，会将电池的更新数据打印到kernel message中
    return 0;
}

int main() {
    return health_service_main(); // 调用libhealthservice中的main方法
}
```

可以看到，整个服务的默认实现中基本为空方法，实际上调用的就是`libhealthservice::HealthServiceCommon.cpp`中的`health_service_main()`方法，最终其实也就是调用`default/healthd_common.cpp`中的`healthd_main()`方法（如果对应OEM厂商没有自己的实现的话），所以整个服务的架构就真相大白了。

## 总结

如果OEM厂商没有实现自己的`android.hardware.health@2.0`服务的话则会采用Google默认的`android.hardware.health@2.0-service`，整个`android.hardware.health@2.0-service`主要依赖于`health@2.0-impl`、`libhealthservice`、`libbatterymonitor`，若采用Google提供的默认实现的话最终就是通过`hardware/interfaces/health/2.0/default/Health.cpp`和`hardware/interfaces/health/2.0/default/healthd_common.cpp`实现对`BatteryMonitor`方法的调用，而`BatteryMonitor`则会定时从`sys/class/power_supply`的节点下进行电量数据的读取，如果在接入AC/USB电源的情况下每`60s`读取一次，未接入电源的情况下`600s`获取一次，在更新数据时会通过hwbinder调用Framework层的回调通知到Framework层进行数据更新，这样就完成了整个电量健康状态的数据更新。
