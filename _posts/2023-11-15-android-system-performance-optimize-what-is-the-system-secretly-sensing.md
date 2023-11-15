---
date: 2023-11-14 08:00:00
layout: post
title: Android 性能优化实践 - 系统在偷偷感知什么
subtitle: Android 性能优化实践 - 系统在偷偷感知什么
description: Android 性能优化实践 - 系统在偷偷感知什么
image: /assets/img/markdown_img/image-title-android-system-optimize-sensor.jpg
optimized_image: /assets/img/markdown_img/image-title-android-system-optimize-sensor.jpg
category: Android system performance optimization
tags:
    - Android System
    - Android system performance optimization
author: etoilexx
paginate: false
---

## 一、知己知彼，百战不殆 - Android 系统的传感器架构

Android操作系统提供了广泛的传感器支持，包括加速度计、陀螺仪、磁力计、光线传感器、温度传感器等。这些传感器不仅可以提供设备当前的状态信息，还可以用于应用程序的开发。

通俗来说，Android的传感器架构主要由三个组件组成：`底层传感器硬件`、`SensorManager`和 `Application`。传感器硬件负责将物理量转换为电信号，并向操作系统发送数据；SensorManager作为传感器的管理者，负责访问和管理传感器；上层Application则负责接收和处理传感器数据，以经典Android Sensor架构为例，如下图所示：

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-performance-optimization/what-is-the-system-secretly-sensing/1.png)

Framework层主要提供了如下几种类用于应用层的传感器数据获取：

* SensorManager - 可以使用这个类来创建传感器服务的实例。该类提供了各种方法来访问和列出传感器，注册和取消注册传感器事件监听器，以及获取屏幕方向信息。它还提供了几个传感器常量，用于报告传感器精确度，设置数据采集频率和校准传感器。
* Sensor - 使用这个类来创建特定传感器的实例。该类提供了各种方法来确定传感器的特性。
* SensorEvent - 系统使用这个类来创建传感器事件对象，该对象提供有关传感器事件的信息。传感器事件对象中包含以下信息：原始传感器数据、生成事件的传感器类型、数据的准确度和事件的时间戳。
* SensorEventListener - 使用此接口创建两种回调方法，以在传感器值或传感器精确度发生变化时接收通知。
 

通常，上层应用/服务通过如下方法获取传感器事件（以环境光线传感器为例）：

```java
public class SensorActivity extends Activity implements SensorEventListener {
    private SensorManager sensorManager;
    private Sensor mLight;

    @Override
    public final void onCreate(Bundle savedInstanceState) {
        ......
        sensorManager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);
        mLight = sensorManager.getDefaultSensor(Sensor.TYPE_LIGHT);
    }

    @Override
    public final void onAccuracyChanged(Sensor sensor, int accuracy) {
        // Do something here if sensor accuracy changes.
    }

    @Override
    public final void onSensorChanged(SensorEvent event) {
        // The light sensor returns a single value.
        // Many sensors return 3 values, one for each axis.
        float lux = event.values[0];
        // Do something with this sensor value.
    }

    @Override
    protected void onResume() {
        super.onResume();
        sensorManager.registerListener(this, mLight, SensorManager.SENSOR_DELAY_NORMAL);
    }

    @Override
    protected void onPause() {
        super.onPause();
        sensorManager.unregisterListener(this);
    }
}
```

其中在通过registerListener() 方法注册传感器回掉方法是可以指定采样频率（数值越小采样率越高）：
* SENSOR_DELAY_GAME - 20000 us
* SENSOR_DELAY_UI - 60000 us
* SENSOR_DELAY_FASTEST - 0 us


## 二、若知其根，逐个攻破 - 排查有损性能的传感器使用

### 2.1 排查无需注册的传感器，减少资源浪费

虽然在SensorManager中罗列了这么多的传感器类型，但并不代表你所开发的设备就一定存在这些物理传感器，因此应用端在使用传感器时需要先进行检查，以Light Sensor为例，可以通过如下方式检查当前设备是否存在光感：

```java
private SensorManager sensorManager;
...
sensorManager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);
if (sensorManager.getDefaultSensor(Sensor.TYPE_LIGHT) != null){
    // Success! There's a light sensor.
} else {
    // Failure! No light sensor.
}
```

在很多定制场景下，我们并不需要完全使用系统的完整功能，因此，需要排查哪些无用功能注册了传感器，将这些功能砍掉或者不要注册，减少资源浪费。

### 2.2 及时注销传感器事件监听器

请确保在不使用传感器或传感器活动暂停时取消注册传感器的监听器。注册传感器监听器后，只要不取消注册传感器，那么即使监听器的活动已暂停，传感器仍会继续采集数据并消耗电池资源。如，通常在应用端，Activity onPause时就需要注销传感器监听：

```java
private SensorManager sensorManager;
...
@Override
protected void onPause() {
    super.onPause();
    sensorManager.unregisterListener(this);
}
```

### 2.3 谨慎选择延时，注重系统功耗

应用端注册了传感器事件监听器（SensorEventListener）后，传感器会进行采样后数据上报，传感器的数据提供频率取决于应用端在注册监听器时的延时，在使用传感器时，务必要规划好传感器的使用场景，如果应用应用端以很高频率获取数据而并不需要处理这些数据，对于系统功耗而言绝对是巨大的资源浪费。

其次，传感器以高频率采样后上报数据，也就是说系统会频繁触发`onSensorChanged(SensorEvent)`方法，这就要求应用端在此方法执行尽可能少的任务，避免阻塞。

## 三、屡试不爽的排查神奇 - dumpsys

我们可以通过执行如下命令获取当前系统的sensorservice信息，通过dump的数据，我们能进行分析当前设备存在哪些传感器，传感器的状态以及传感器的历史事件等信息：

```shell
$ adb shell dumpsys sensorservice
```

### 设备搭载的传感器信息

以我手头的这台平板设备为例，当前设备搭载了一个加速度传感器，厂商信息为`MTK`，type为`android.sensor.accelerometer（1）`，采样率为`125Hz`，其余的传感器还有光照传感器（LIGHT）、接近距离传感器（PROXIMITY）、霍尔传感器（CAMHALL）、角度传感器（G-Sensor）：

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-performance-optimization/what-is-the-system-secretly-sensing/2.png)

### 传感器历史事件

当存在应用/服务对传感器的事件进行监听注册时，`SensorHALService`就会有数据记录，dumpsys中详细记录了每个有注册回掉方法的Sensor的事件，如下是Light Sensor的最近50次的事件，每次事件的格式由时间戳、事件发生事件、sensor value组成：

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-performance-optimization/what-is-the-system-secretly-sensing/3.png)

CAMHALL Sensor与Proximity Sensor的历史事件：

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-performance-optimization/what-is-the-system-secretly-sensing/4.png)

Accelerometer Sensor（加速度传感器）的历史事件：

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-performance-optimization/what-is-the-system-secretly-sensing/5.png)

### 当前活跃的Sensor

通过dump信息可以查看到当前系统活跃的Sensor，以笔者当前手中的平板为例，可以看到当前活跃的Sensor有4个，分别是加速度传感器、光照传感器、距离传感器和霍尔传感器，一共有5个活跃的连接：

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-performance-optimization/what-is-the-system-secretly-sensing/6.png)

## 四、传感器优化实战

### 4.1 去除定制系统中的那些无效监听

在开发调试过程中发现，笔者手头的这个平板设备疯狂在打印距离感应传感器的日志信息，过多的信息影响了系统运行的流畅度：

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-performance-optimization/what-is-the-system-secretly-sensing/7.png)

起初以为是某个护眼应用的正常注册，仅仅排查了打印这行Log的地点并注释掉：

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-performance-optimization/what-is-the-system-secretly-sensing/8.png)

但该做法终究治标不治本，怀着强烈的探索欲，将sensorservice的信息dump出来：

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-performance-optimization/what-is-the-system-secretly-sensing/6.png)

没错，还是这张图，其中`com.android.server.AutomaticBrightnessController`注册监听了亮度传感器，用于更改屏幕的自动亮度；自研应用注册了加速度传感器的监听用于检测设备姿态；`WindowOrientationListener`注册监听了加速度传感器用于监听屏幕方向；这些传感器的注册监听均属于已知的正常逻辑；

而其中systemui的`BrightLineFalsingManager`监听了距离传感器，看起来有些奇怪，需要排查排查，确认到距感的注册在SystemUI的如下地方：

![](https://cdn.jsdelivr.net/gh/etoilexx/etoilexx.github.io@master/_posts/markdown_img/android-system-performance-optimization/what-is-the-system-secretly-sensing/9.png)

通过排查发现，`BrightLineFalsingManager`是Android SystemUI中用于判断锁屏（Keyguard）下是否存在用户误触的一个子管理器，而对于笔者需要定制的系统版本来说，去除了AOSP的Keyguard相关逻辑，因此无需使用此功能，因此将该功能取消后大大降低了系统静默状态下的CPU占用率。
