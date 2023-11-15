---
date: 2022-07-05 08:00:00
layout: post
title: AndroidQ Battery（三）- Linux Kernel Power Supply与PMIC分析
subtitle: AndroidQ Battery（三）- Linux Kernel Power Supply与PMIC分析
description: AndroidQ Battery（三）- Linux Kernel Power Supply与PMIC分析
image: /assets/img/markdown_img/image-title-android-battery.jpg
optimized_image: /assets/img/markdown_img/image-title-android-battery.jpg
category: Android System
tags:
    - Android System
    - Android Battery
author: etoilexx
paginate: false
---

## 一、Overview

前面两章分析了Android Framework层与HAL层Battery信息监控的链路，了解到了上层的BatteryService通过HWBinder与HAL层Health\@2.0-service进行通信，而HAL层的Health\@2.0-service实际通过BatteryMonitor去定时监控`/sys/class/power_supply`中的文件节点来获取电池的健康信息。

那么问题来了，`/sys/class/power_supply`中的数据是哪里来的？带着这个问题，我们来一探究竟。

> 注：本文基于Kernel 4.9 PMIC-全志AXP803

## 二、Linux Power Supply Class

`Linux Power Supply Class（PSY Class）`为编写电源管理设备驱动提供了统一的框架，该框架通过`SysFS`与`UEvent`将从驱动中读取到的一系列的电池、AC、DC、USB的电量属性信息数据提供给用户空间。

整个框架主要分为四个部分：

*   power supply core：用于抽象核心数据结构、实现公共逻辑
*   power supply sysfs：实现sysfs以及uevent功能
*   power supply leds：基于linux led class，提供电源指示灯的通用实现（本文暂不展开）
*   power supply drivers：psy设备的驱动实现，与设备搭载的PMIC相关

### power\_supply

Linux power supply定义了整个电源提供的配置规范，为上层提供统一接口，具体的PMIC供应商只需要实现针对特定PMIC的驱动层方法即可接入power\_supply子系统：

*   `kernel-4.9/include/linux/power_supply.h`

```h
#ifndef __LINUX_POWER_SUPPLY_H__
#define __LINUX_POWER_SUPPLY_H__
......

// 用于描述供电的属性信息枚举
enum power_supply_property {
	/* Properties of type `int' */
	POWER_SUPPLY_PROP_STATUS = 0,
	POWER_SUPPLY_PROP_CHARGE_TYPE,
	POWER_SUPPLY_PROP_HEALTH,
	......
	POWER_SUPPLY_PROP_ONLINE,
	......
	POWER_SUPPLY_PROP_TEMP,
	POWER_SUPPLY_PROP_TEMP_MAX,
	......
};
......

// 采用共用体方式存放单个属性值
union power_supply_propval {
	int intval; // int类型的属性值
	const char *strval; // 字符串类型的属性值
};

......

struct power_supply_desc { // 用于描述供电节点的数据结构
	const char *name; // 节点名称 /sys/class/power_supply/<name>
	enum power_supply_type type; // 记录供电类型，如POWER_SUPPLY_TYPE_BATTERY 表示电池
	enum power_supply_property *properties; // 记录供电信息属性枚举
	size_t num_properties; // 供电信息属性个数

	// power supply的设备驱动需要实现的方法的函数指针【重点实现】
	int (*get_property)(struct power_supply *psy, // 获取供电属性
			    enum power_supply_property psp,
			    union power_supply_propval *val);
	int (*set_property)(struct power_supply *psy, // 设置供电属性
			    enum power_supply_property psp,
			    const union power_supply_propval *val);
	int (*property_is_writeable)(struct power_supply *psy, // 判断属性可写
				     enum power_supply_property psp);
	void (*external_power_changed)(struct power_supply *psy);
	void (*set_charged)(struct power_supply *psy); // 设置充电状态

	bool no_thermal;
	int use_for_apm;
};

struct power_supply { // power supply框架
	const struct power_supply_desc *desc; // 一个节点描述指针

	char **supplied_to; // 
	size_t num_supplicants;

	char **supplied_from;
	size_t num_supplies;
	struct device_node *of_node;

	void *drv_data;// 从驱动获取的数据

	/* private */
	struct device dev;
	struct work_struct changed_work;
	struct delayed_work deferred_register_work;
	spinlock_t changed_lock;
	bool changed;
	bool initialized;
	bool removing;
	atomic_t use_cnt;
#ifdef CONFIG_THERMAL
	struct thermal_zone_device *tzd;
	struct thermal_cooling_device *tcd;
#endif

// 电源指示灯相关的一些变量信息
#ifdef CONFIG_LEDS_TRIGGERS
	struct led_trigger *charging_full_trig;
	char *charging_full_trig_name;
	struct led_trigger *charging_trig;
	char *charging_trig_name;
	struct led_trigger *full_trig;
	char *full_trig_name;
	struct led_trigger *online_trig;
	char *online_trig_name;
	struct led_trigger *charging_blink_full_solid_trig;
	char *charging_blink_full_solid_trig_name;
#endif
};
......
power_supply_register(struct device *parent,
				 const struct power_supply_desc *desc,
				 const struct power_supply_config *cfg);
power_supply_register_no_ws(struct device *parent,
				 const struct power_supply_desc *desc,
				 const struct power_supply_config *cfg);
devm_power_supply_register(struct device *parent,
				 const struct power_supply_desc *desc,
				 const struct power_supply_config *cfg);
extern struct power_supply *__must_check
devm_power_supply_register_no_ws(struct device *parent,
				 const struct power_supply_desc *desc,
				 const struct power_supply_config *cfg);
extern void power_supply_unregister(struct power_supply *psy);
extern int power_supply_powers(struct power_supply *psy, struct device *dev);

extern void *power_supply_get_drvdata(struct power_supply *psy); // 获取驱动数据
......
#endif /* __LINUX_POWER_SUPPLY_H__ */
```

### power\_supply\_sysfs

`power_supply_sysfs`提供了psy设备在`/sys/class/power_supply`下的操作方法：

*   `kernel-4.9/drivers/power/supply/power_supply_sysfs.c`

```cpp
// 通过宏定义POWER_SUPPLY_ATTR批量创建sysfs节点属性
#define POWER_SUPPLY_ATTR(_name)					\
{									\
	.attr = { .name = #_name },					\
	.show = power_supply_show_property,				\
	.store = power_supply_store_property,				\
}

static struct device_attribute power_supply_attrs[];
......

// 与power_supply.h中的power_supply_property枚举一一对应，配置对应的sysfs属性节点与存取方法
static struct device_attribute power_supply_attrs[] = {
	/* Properties of type `int' */
	POWER_SUPPLY_ATTR(status),
	POWER_SUPPLY_ATTR(charge_type),
	POWER_SUPPLY_ATTR(health),
	POWER_SUPPLY_ATTR(present),
	POWER_SUPPLY_ATTR(online),
	POWER_SUPPLY_ATTR(authentic),
	POWER_SUPPLY_ATTR(technology),
	POWER_SUPPLY_ATTR(cycle_count),
    ......
	POWER_SUPPLY_ATTR(model_name),
	POWER_SUPPLY_ATTR(manufacturer),
	POWER_SUPPLY_ATTR(serial_number),
};

// sysfs下的节点属性读取方法
static ssize_t power_supply_show_property(struct device *dev,
					  struct device_attribute *attr,
					  char *buf) {
	static char *type_text[] = { // type节点中存储的可能值
		"Unknown", "Battery", "UPS", "Mains", "USB",
		"USB_DCP", "USB_CDP", "USB_ACA", "USB_C",
		"USB_PD", "USB_PD_DRP"
	};
	static char *status_text[] = { // statu节点中存储的可能值
		"Unknown", "Charging", "Discharging", "Not charging", "Full"
	};
	static char *charge_type[] = {......};
	static char *health_text[] = { // health节点的可能值
		"Unknown", "Good", "Overheat", "Dead", "Over voltage",
		"Unspecified failure", "Cold", "Watchdog timer expire",
		"Safety timer expire"
	};
	static char *technology_text[] = { // technology节点的可能值
		"Unknown", "NiMH", "Li-ion", "Li-poly", "LiFe", "NiCd",
		"LiMn"
	};
	static char *capacity_level_text[] = {......};
	static char *scope_text[] = {......};
	ssize_t ret = 0;
	struct power_supply *psy = dev_get_drvdata(dev); // 从驱动中获取power_supply地址类型的属性数据
	const ptrdiff_t off = attr - power_supply_attrs; // 利用内存地址偏移（目标属性地址 - 属性数组基地址）计算出当前的属性节点枚举值
	union power_supply_propval value;// 采用共用体方式存储单个属性值

    /*sysfs下属性节点数据展示， cat /sys/class/power_supply/<节点>*/
	if (off == POWER_SUPPLY_PROP_TYPE) { // 电源供应类型节点（int -> enum值）
		value.intval = psy->desc->type;
	} else {
	    // 从power_supply地址中取出off偏移的属性值，存放到value中
		ret = power_supply_get_property(psy, off, &value);

		if (ret < 0) {
			if (ret == -ENODATA)
				dev_dbg(dev, "driver has no data for `%s' property\n",
					attr->attr.name);
			else if (ret != -ENODEV && ret != -EAGAIN)
				dev_err(dev, "driver failed to report `%s' property: %zd\n",
					attr->attr.name, ret);
			return ret;
		}
	}
	if (off == POWER_SUPPLY_PROP_STATUS) // 从status_text数组中找对应值进行打印
		return sprintf(buf, "%s\n", status_text[value.intval]);
	else if (off == POWER_SUPPLY_PROP_CHARGE_TYPE) // 从charge_type数组中找对应值进行打印
		return sprintf(buf, "%s\n", charge_type[value.intval]);
	else if (off == POWER_SUPPLY_PROP_HEALTH) // 从health_text数组中找对应值进行打印
		return sprintf(buf, "%s\n", health_text[value.intval]);
	else if (off == POWER_SUPPLY_PROP_TECHNOLOGY) // 从technology_text数组中找对应值进行打印
		return sprintf(buf, "%s\n", technology_text[value.intval]);
	else if (off == POWER_SUPPLY_PROP_CAPACITY_LEVEL) // 从capacity_level_text数组中找对应值进行打印
		return sprintf(buf, "%s\n", capacity_level_text[value.intval]);
	else if (off == POWER_SUPPLY_PROP_TYPE) // 从type_text数组中找对应值进行打印
		return sprintf(buf, "%s\n", type_text[value.intval]);
	else if (off == POWER_SUPPLY_PROP_SCOPE) // 从scope_text数组中找对应值进行打印
		return sprintf(buf, "%s\n", scope_text[value.intval]);
	else if (off >= POWER_SUPPLY_PROP_MODEL_NAME)
		return sprintf(buf, "%s\n", value.strval);

	if (off == POWER_SUPPLY_PROP_CHARGE_COUNTER_EXT)
		return sprintf(buf, "%lld\n", value.int64val);
	else
		return sprintf(buf, "%d\n", value.intval);
}

// sysfs下的节点属性配置方法，echo xxx > sys/class/power_supply/<节点>
static ssize_t power_supply_store_property(struct device *dev,
					   struct device_attribute *attr,
					   const char *buf, size_t count) {
	ssize_t ret;
	struct power_supply *psy = dev_get_drvdata(dev);
	const ptrdiff_t off = attr - power_supply_attrs; // 获取属性地址偏移
	union power_supply_propval value;
	long long_val;

	/* TODO: support other types than int */
	ret = kstrtol(buf, 10, &long_val); // 将str数据转成long类型存储
	if (ret < 0)
		return ret;

	value.intval = long_val;

    // 利用power_supply_set_property方法将value的值写入off偏移的属性中
	ret = power_supply_set_property(psy, off, &value);
	if (ret < 0)
		return ret;

	return count;
}

......

// power supply的uevent事件处理
int power_supply_uevent(struct device *dev, struct kobj_uevent_env *env)
{
	struct power_supply *psy = dev_get_drvdata(dev); // 从驱动获取数据
	int ret = 0, j;
	char *prop_buf;
	char *attrname;

	dev_dbg(dev, "uevent\n");

	if (!psy || !psy->desc) {
		dev_dbg(dev, "No power supply yet\n");
		return ret;
	}

	dev_dbg(dev, "POWER_SUPPLY_NAME=%s\n", psy->desc->name);
    // 通过kobject API添加一个uevent事件到用户空间（PSY的描述名称）
	ret = add_uevent_var(env, "POWER_SUPPLY_NAME=%s", psy->desc->name);
	if (ret)
		return ret;

	prop_buf = (char *)get_zeroed_page(GFP_KERNEL); //申请一页内容为0的内存空间
	if (!prop_buf)
		return -ENOMEM;
    // 根据psy的desc中的num_properties定义，将所有的property添加到uevent中
	for (j = 0; j < psy->desc->num_properties; j++) {
		struct device_attribute *attr;
		char *line;
        // 读取属性偏移
		attr = &power_supply_attrs[psy->desc->properties[j]];
        // 将属性节点值写入到prop_buf中
		ret = power_supply_show_property(dev, attr, prop_buf);
		if (ret == -ENODEV || ret == -ENODATA) {
			ret = 0;
			continue;
		}

		if (ret < 0) goto out;

		line = strchr(prop_buf, '\n'); // 增加换行符
		if (line)
			*line = 0;

		attrname = kstruprdup(attr->attr.name, GFP_KERNEL);
		if (!attrname) {
			ret = -ENOMEM;
			goto out;
		}

		dev_dbg(dev, "prop %s=%s\n", attrname, prop_buf);
        // 将prop_buf中的内容发布到uevent中
		ret = add_uevent_var(env, "POWER_SUPPLY_%s=%s", attrname, prop_buf);
		kfree(attrname); // 释放内存空间
		if (ret)
			goto out;
	}

out:
	free_page((unsigned long)prop_buf); // 释放内存页

	return ret;
}
```

### power\_supply\_core

上面两个是power\_supply对应的框架，`power_supply_core`则是真正创建设备及节点的实现，并为psy设备驱动提供统一接口：

*   `kernel-4.9/drivers/power/supply/power_supply_core.c`

```cpp
struct class *power_supply_class;
EXPORT_SYMBOL_GPL(power_supply_class);

ATOMIC_NOTIFIER_HEAD(power_supply_notifier);
EXPORT_SYMBOL_GPL(power_supply_notifier);

static struct device_type power_supply_dev_type;
......
// 当power supply设备属性值发生改变时的工作方法【重要】
static void power_supply_changed_work(struct work_struct *work)
{
	unsigned long flags;
	struct power_supply *psy = container_of(work, struct power_supply,
						changed_work); // 从工作队列中获取psy设备

	dev_dbg(&psy->dev, "%s\n", __func__);

	spin_lock_irqsave(&psy->changed_lock, flags); // 自旋锁锁定，保证对changed的修改原子性
	if (likely(psy->changed)) {
		psy->changed = false;
		spin_unlock_irqrestore(&psy->changed_lock, flags); // 释放自旋锁
		class_for_each_device(power_supply_class, NULL, psy,
				      __power_supply_changed_work);
		power_supply_update_leds(psy); // 更新电源指示灯
		atomic_notifier_call_chain(&power_supply_notifier,
				PSY_EVENT_PROP_CHANGED, psy);
		kobject_uevent(&psy->dev.kobj, KOBJ_CHANGE); // 向上层发送uevent变更事件
		spin_lock_irqsave(&psy->changed_lock, flags); // 自旋锁锁定，保证对changed的修改原子性
	}
	if (likely(!psy->changed))
		pm_relax(&psy->dev);
	spin_unlock_irqrestore(&psy->changed_lock, flags); // 释放自旋锁
}

void power_supply_changed(struct power_supply *psy)
{
	unsigned long flags;

	dev_dbg(&psy->dev, "%s\n", __func__);

	spin_lock_irqsave(&psy->changed_lock, flags);
	psy->changed = true;
	pm_stay_awake(&psy->dev);
	spin_unlock_irqrestore(&psy->changed_lock, flags);
	schedule_work(&psy->changed_work); // 调度工作队列
}
EXPORT_SYMBOL_GPL(power_supply_changed);

static void power_supply_deferred_register_work(struct work_struct *work)
{
	struct power_supply *psy = container_of(work, struct power_supply,
						deferred_register_work.work);

	if (psy->dev.parent) {
		while (!mutex_trylock(&psy->dev.parent->mutex)) { // 如果存在父设备节点，增加互斥锁访问，保证延时成功
			if (psy->removing)
				return;
			msleep(10); // 延时10ms再进行调度
		}
	}

	power_supply_changed(psy); // 调用power_supply_changed开始power supply数据更新的工作队列调度

	if (psy->dev.parent)
		mutex_unlock(&psy->dev.parent->mutex); // 释放互斥锁
}

......

// power supply设备注册方法【重要】
static struct power_supply *__must_check
__power_supply_register(struct device *parent,
				   const struct power_supply_desc *desc,
				   const struct power_supply_config *cfg,
				   bool ws)
{
	struct device *dev; // 定义Linux kernel device
	struct power_supply *psy;
	int rc;

	if (!parent)
		pr_warn("%s: Expected proper parent device for '%s'\n",
			__func__, desc->name);

	psy = kzalloc(sizeof(*psy), GFP_KERNEL); // 申请psy设备内存空间
	if (!psy)
		return ERR_PTR(-ENOMEM);

	dev = &psy->dev;
	// 初始化psy设备，完成设备引用计数器、信号量、设备访问锁等字段的初始化工作
	device_initialize(dev);

	dev->class = power_supply_class; // 配置 sysfs节点
	dev->type = &power_supply_dev_type; // 配置psy设备的type
	dev->parent = parent; // 配置父设备
	dev->release = power_supply_dev_release; // 设备空间释放方法
	dev_set_drvdata(dev, psy); //调用kernel api设置设备数据
	psy->desc = desc; //增加psy描述
	if (cfg) {
		psy->drv_data = cfg->drv_data;
		psy->of_node = cfg->of_node;
		psy->supplied_to = cfg->supplied_to;
		psy->num_supplicants = cfg->num_supplicants;
	}

	rc = dev_set_name(dev, "%s", desc->name);
	if (rc)
		goto dev_set_name_failed;

    // 注册数据变化时的工作队列，将工作和power_supply_changed_work方法绑定
	INIT_WORK(&psy->changed_work, power_supply_changed_work);
	// 注册延时工作队列，将工作和power_supply_deferred_register_work方法绑定
	INIT_DELAYED_WORK(&psy->deferred_register_work,
			  power_supply_deferred_register_work);

	rc = power_supply_check_supplies(psy);
	if (rc) {
		dev_info(dev, "Not all required supplies found, defer probe\n");
		goto check_supplies_failed;
	}

	spin_lock_init(&psy->changed_lock);
	rc = device_init_wakeup(dev, ws);
	if (rc)
		goto wakeup_init_failed;

	rc = device_add(dev); // 将psy设备链接到kernel device链表
    ......

	atomic_inc(&psy->use_cnt); // 通过原子增方法增加psy设备的引用计数
	psy->initialized = true; //标记psy设备初始化成功

	queue_delayed_work(system_power_efficient_wq,
			   &psy->deferred_register_work,
			   POWER_SUPPLY_DEFERRED_REGISTER_TIME);

	return psy;
......
}

......

// /sys/class/power_supply节点的创建
static int __init power_supply_class_init(void)
{
	power_supply_class = class_create(THIS_MODULE, "power_supply");

	if (IS_ERR(power_supply_class))
		return PTR_ERR(power_supply_class);
		
    // 配置uevent节点的处理方法
    power_supply_class->dev_uevent = power_supply_uevent;
    
	// 通过power_supply_sysfs中的power_supply_init_attrs方法对设备节点属性进行初始化
	power_supply_init_attrs(&power_supply_dev_type); 
	
	return 0;
}

// /sys/class/power_supply节点的销毁
static void __exit power_supply_class_exit(void)
{
	class_destroy(power_supply_class);
}
......
```

## 三、Power Manager IC

首先，我们都知道，电子设备通常会搭载一个电源管理芯片（PMIC，Power Manager IC）来进行主板供电系统的管理，通常一款PMIC能够提供电源电压转换、过压保护、电池温度检测等等，kernel中power\_supply子系统中的大部分信息也都是通过PMIC的驱动进行处理的。

### PMIC相关名词解释

*   LDO（Low Dropout Regulator）：低压差线性稳压器
    *   低压差：输入输出压差（Dropout Voltage）比较低，例如输入3.3V，输出可以达到3.2V
    *   线性： LDO内部的MOS管工作于线性电阻
    *   稳压器： LDO的用途是用来给电源稳压
*   PSRR（Power supply rejection ratio）：电源抑制比，是衡量电路对于输入电源中纹波抑制大小的重要参数，表示为输出纹波和输入纹波的对数比，单位为分贝（dB），它反映了LDO不受噪声和电压波动、保持输出电压稳定的能力。在特定频段内，PSRR越大越好。
*   ALDO：模拟LDO，PSRR要求比较高，PSRR的典型值为65dB
*   DLDO：数字LDO，PSRR要求比较低，典型值为40dB
*   AC-DC：交变电流转直流电压转换
*   DC-DC：直流电压转直流电压，严格来讲，LDO也是DC/DC的一种，但目前DC/DC多指开关电源
*   BOP（掉电保护）：防止电源单元由于不一致的电网电压突然下降而损坏。
*   OVP（过压保护）：当电压超过预设水平时关闭设备或暂停输出的电源功能。一般在电压输出超过110%～130%时才启动。
*   OPP（过功率保护）：防止因功率输出过大而造成损坏。这通常在连接组件的功率达到 130% 到 150% 时激活。
*   OTP（过温保护）：当内部温度超过最高安全工作温度时关闭电源。
*   OCP（过电流保护）：防止将过多电流推入 PSU 的潜在危险影响。这可能会导致设备过载或短路，从而可能产生故障电流并损坏电源设备或连接的组件，例如主板。当输出电流达到 130% 到 150% 时激活。
*   SCP（短路保护）：防止主板因高温输出而烧毁。

### 全志AXP803（AXP707）PMIC分析

根据官方文档，AXP707的内部结构如下图：

![](https://cdn.jsdelivr.net/gh/jonewan/MD_PNG@master/android_battery/axp707.png)

#### AXP803 Device Tree

将SSP中`/sys/firmware/fdt`导出并反编译后得到对应的设备树源码如下：

    ......
                pmu@34 {
                    compatible = "x-powers,axp803";
                    reg = <0x00000034>; // i2c寄存器地址
                    status = "okay";
                    interrupts = <0x00000000 0x00000008>; // 中断配置
                    interrupt-parent = <0x00000040>; // 上级中断控制器节点
                    x-powers,drive-vbus-en;
                    pmu_reset = <0x00000000>; // 电源键按下超过16s是否复位（0 - 不复位）
                    pmu_irq_wakeup = <0x00000001>;
                    pmu_hot_shutdown = <0x00000001>; // PMU过热保护是否开启（1 - 开启）
                    wakeup-source;
                    linux,phandle = <0x00000163>;
                    phandle = <0x00000163>;
                    ac-power-supply { // ac电源供应配置
                        compatible = "x-powers,axp803-ac-power-supply";
                        status = "okay";
                        pmu_ac_vol = <0x000011f8>; // ac输入电压限制 4600mV
                        pmu_ac_cur = <0x00000bb8>; // ac输入电流限制 3000mA
                        wakeup_ac_in; // ac插入唤醒
                        wakeup_ac_out; // ac拔出唤醒
                        linux,phandle = <0x00000164>;
                        phandle = <0x00000164>;
                    };
                    usb_power_supply { // usb电源供应配置
                        compatible = "x-powers,axp803-usb-power-supply";
                        status = "okay";
                        pmu_usbpc_vol = <0x000011f8>; // usb输入电压限制 4600mV
                        pmu_usbpc_cur = <0x000001f4>; // usb pc输入电流限制 500mA
                        pmu_usbad_vol = <0x000011f8>; // usb 适配器输入电压限制 4600mV
                        pmu_usbad_cur = <0x000009c4>; // usb适配器输入电流限制 2500mA
                        wakeup_usb_in; // usb插入唤醒
                        wakeup_usb_out; // usb拔出唤醒
                        linux,phandle = <0x0000006d>;
                        phandle = <0x0000006d>;
                    };
                    battery-power-supply { // 电池电源供应配置
                        compatible = "x-powers,axp803-battery-power-supply";
                        status = "okay";
                        pmu_chg_ic_temp = <0x00000000>; // TS引脚输入源始终开启 （0 - 关闭 1 - 开启）
                        pmu_battery_rdc = <0x0000005d>; // 电池内阻 93mΩ
                        pmu_battery_cap = <0x000013cb>; // 电池容量 5067mAh
                        pmu_runtime_chgcur = <0x000003e8>; // 运行时持续的充电电流限制 1000mA
                        pmu_suspend_chgcur = <0x000007d0>; // 休眠时持续的充电电流限制 2000mA
                        pmu_shutdown_chgcur = <0x000007d0>; // 关机时持续的充电电流限制 2000mA
                        pmu_init_chgvol = <0x00001068>; // 电池充满电压4200mV
                        pmu_battery_warning_level1 = <0x0000000f>; // 低电量告警1阈值 15%
                        pmu_battery_warning_level2 = <0x00000000>; // 低电量告警2阈值 0%
                        pmu_chgled_func = <0x00000000>;
                        pmu_chgled_type = <0x00000000>;
                        ocv_coulumb_100 = <0x00000001>;
                        // 电池曲线参数
                        pmu_bat_para1 = <0x00000000>;
                        ......
                        pmu_bat_para32 = <0x00000064>;
                        pmu_bat_temp_enable = <0x00000000>; // 电池温度检测 ntc是否使能 0-关闭 1-开启
                        pmu_bat_charge_ltf = <0x00000451>;
                        pmu_bat_charge_htf = <0x00000079>;
                        pmu_bat_shutdown_ltf = <0x00000565>;
                        pmu_bat_shutdown_htf = <0x00000059>;
                        pmu_bat_temp_para1 = <0x00000afe>; // 电池包-25℃对应的TS引脚电压 2814mV
                        ......
                        pmu_bat_temp_para16 = <0x00000042>; // 电池包80℃对应的TS引脚电压66mV
                        wakeup_bat_out;
                        linux,phandle = <0x00000165>;
                        phandle = <0x00000165>;
                    };
                    powerkey@0 { //电源键配置
                        status = "okay";
                        compatible = "x-powers,axp2101-pek";
                        pmu_powkey_off_time = <0x00001770>; // 按下后多长时间响应poweroff事件 6000ms
                        pmu_powkey_off_func = <0x00000000>; // poweroff事件功能 0 - poweroff
                        pmu_powkey_off_en = <0x00000001>;
                        pmu_powkey_long_time = <0x000005dc>;
                        pmu_powkey_on_time = <0x00000200>; // 按下后多久开机 512ms
                        wakeup_rising;
                        wakeup_falling;
                        linux,phandle = <0x00000166>;
                        phandle = <0x00000166>;
                    };
                    regulators@0 { // PMU内部调节器控制
                        linux,phandle = <0x00000167>;
                        phandle = <0x00000167>;
                        dcdc1 {
                            regulator-name = "axp803-dcdc1";
                            regulator-min-microvolt = <0x00186a00>; // 电源最小值
                            regulator-max-microvolt = <0x0033e140>; // 电源最大值
                            regulator-ramp-delay = <0x000000fa>; // 电源调压延时us
                            regulator-enable-ramp-delay = <0x000003e8>; // disable-enable的延时
                            regulator-boot-on;
                            linux,phandle = <0x00000027>;
                            phandle = <0x00000027>;
                        };
                        dcdc2 {......};
                        dcdc3 {......};
                        dcdc4 {......};
                        dcdc5 {......};
                        dcdc6 {......};
                        dcdc7 {......};
                        rtcldo {
                            regulator-name = "axp803-rtcldo";
                            ......
                        };
                        aldo1 {
                            regulator-name = "axp803-aldo1";
                            ......
                        };
                        aldo2 {......};
                        aldo3 {......};
                        dldo1 {
                            regulator-name = "axp803-dldo1";
                            ......
                        };
                        dldo2 {......};
                        dldo3 {......};
                        dldo4 {......};
                        eldo1 {......};
                        eldo2 {......};
                        eldo3 {......};
                        fldo1 {......};
                        fldo2 {......};
                        ldoio0 {......};
                        ldoio1 {......};
                        dc1sw {
                            regulator-name = "axp803-dc1sw";
                            ......
                        };
                        drivevbus {
                            regulator-name = "axp803-drivevbus";
                            ......
                        };
                    };
                    ......
                    axp_gpio@0 {
                        gpio-controller;
                        #size-cells = <0x00000000>;
                        #gpio-cells = <0x00000006>;
                        status = "okay";
                        linux,phandle = <0x00000168>;
                        phandle = <0x00000168>;
                    };
                };

#### PMIC regulator调试方法

kernel定义了针对PMIC regulator的调试框架，在sysfs下生成了一个`/sys/kernel/debug/regulator/regulator_summary`的调试节点，能够通过该节点获取各路电源的详细状态（以SSP为例）：

    ssp:/ # mount -t debugfs none /sys/kernel/debug
    ssp:/ # cat /sys/kernel/debug/regulator/regulator_summary
     regulator                      use open bypass voltage current     min     max
    -------------------------------------------------------------------------------
     regulator-dummy                  0    5      0     0mV     0mA     0mV     0mV
        uart1                                                           0mV     0mV
        twi3                                                            0mV     0mV
        twi1                                                            0mV     0mV
        twi0                                                            0mV     0mV
        twi6                                                            0mV     0mV
     usb1-vbus                        0    0      0  5000mV     0mA  5000mV  5000mV
     axp803-dcdc1                     6    8      0  3300mV     0mA  1600mV  3400mV
        1-0047                                                          0mV     0mV
        sdc0                                                            0mV     0mV
        sdc0                                                            0mV     0mV
        sdc0                                                            0mV     0mV
        sdc2                                                            0mV     0mV
        1-0015                                                       3300mV  3300mV
        uart0                                                           0mV     0mV
        reg-virt-consumer.1                                             0mV     0mV
     axp803-dcdc2                     0    2      0  1030mV     0mA   500mV  1300mV
        cpu0                                                         1030mV  1030mV
        reg-virt-consumer.2                                             0mV     0mV
     axp803-dcdc3                     0    1      0   900mV     0mA   500mV  1300mV
        reg-virt-consumer.3                                             0mV     0mV
     axp803-dcdc4                     0    2      0   950mV     0mA   500mV  1300mV
        gpu                                                             0mV     0mV
        reg-virt-consumer.4                                             0mV     0mV
     axp803-dcdc5                     0    1      0  1100mV     0mA   800mV  1840mV
        reg-virt-consumer.5                                             0mV     0mV
     axp803-dcdc6                     0    1      0   900mV     0mA   600mV  1520mV
        reg-virt-consumer.6                                             0mV     0mV
     axp803-dcdc7                     0    1      0  1280mV     0mA   800mV  1520mV
        reg-virt-consumer.7                                             0mV     0mV
     axp803-rtcldo                    0    1      0  1800mV     0mA  1800mV  1800mV
        reg-virt-consumer.8                                             0mV     0mV
     axp803-aldo1                     0    2      0  1800mV     0mA   700mV  3300mV 
        codec                                                        1800mV  1800mV
        reg-virt-consumer.9                                             0mV     0mV
     axp803-aldo2                     0    1      0  1800mV     0mA   700mV  3300mV
        reg-virt-consumer.10                                            0mV     0mV
     axp803-aldo3                     0    1      0  3300mV     0mA   700mV  3300mV
        reg-virt-consumer.11                                            0mV     0mV
     axp803-dldo1                     0    1      0  3300mV     0mA   700mV  3300mV
        reg-virt-consumer.12                                            0mV     0mV
     axp803-dldo2                     0    2      0  1800mV     0mA   700mV  4200mV
        reg-virt-consumer.13                                            0mV     0mV
        twi2                                                            0mV     0mV
     axp803-dldo3                     0    1      0  1800mV     0mA   700mV  3300mV
        reg-virt-consumer.14                                            0mV     0mV
     axp803-dldo4                     0    1      0  1800mV     0mA   700mV  3300mV
        reg-virt-consumer.15                                            0mV     0mV
     axp803-eldo1                     3    5      0  1800mV     0mA   700mV  1900mV
        codec                                                        1800mV  1800mV
        sdc0                                                            0mV     0mV
        sdc0                                                            0mV     0mV
        sdc2                                                            0mV     0mV
        reg-virt-consumer.16                                            0mV     0mV
     axp803-eldo2                     0    1      0  1800mV     0mA   700mV  1900mV
        reg-virt-consumer.17                                            0mV     0mV
     axp803-eldo3                     1    1      0  1800mV     0mA   700mV  1900mV
        reg-virt-consumer.18                                            0mV     0mV
     axp803-fldo1                     0    1      0   900mV     0mA   700mV  1450mV
        reg-virt-consumer.19                                            0mV     0mV
     axp803-fldo2                     0    1      0   800mV     0mA   700mV  1450mV
        reg-virt-consumer.20                                            0mV     0mV
     axp803-ldoio0                    1    2      0  3300mV     0mA   700mV  3300mV
        0-002e                                                       3300mV  3300mV
        reg-virt-consumer.21                                            0mV     0mV
     axp803-ldoio1                    0    1      0  3300mV     0mA   700mV  3300mV
        reg-virt-consumer.22                                            0mV     0mV
     axp803-dc1sw                     1    1      0     0mV     0mA     0mV     0mV
        reg-virt-consumer.23                                            0mV     0mV
     axp803-drivevbus                 0    1      0     0mV     0mA     0mV     0mV
        reg-virt-consumer.24                                            0mV     0mV

## 参考文献

*   [The Power Supply Subsystem](https://elinux.org/images/4/45/Power-supply_Sebastian-Reichel.pdf)
