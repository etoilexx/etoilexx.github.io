---
date: 2022-07-07 08:00:00
layout: post
title: Android设备树编译与反编译
subtitle: Android设备树编译与反编译
description: Android设备树编译与反译
image: /assets/img/markdown_img/image-title-android-linux-device-tree.jpg
optimized_image: /assets/img/markdown_img/image-title-android-linux-device-tree.jpg
category: Android System
tags:
    - Android System
    - Linux Device Tree
author: etoilexx
paginate: false
---

我们都知道，Linux为了将设备信息与驱动信息进行分离，创建了Linux设备树的机制，用来描述板卡硬件资源信息，我们来了解下设备树是如何编译与反编译的。

## DTS、DTSI、DTC、DTB的关系

首先来介绍几个概念名词：

* DTS（Device Tree Source）：设备树源文件，也就是在系统代码中的描述设备树的源码文件
* DTSI（Device Tree Source Include）：设备树源文件的可引入文件，设备厂商将一些描述SOC的外设信息进行提炼，可在多个dts中引用
* DTC（Device Tree Compiler）：将DTS的源码文件编译成Linux识别的二进制文件
* DTB（Device Tree Binary）：`dtb`文件是`dts`被 DTC 编译后的二进制格式的设备树文件，它可以被linux内核解析。

### DTC(Device Tree Compiler)

`DTC`的可执行文件为`dtc`，在`Linux Kernel`编译时会调用`dtc`命令将设备的`dts`文件编译成为对应的`dtb`文件，`dtc`的命令解析如下：

```sh
Usage: dtc [options] <input file>

Options: -[qI:O:o:V:d:R:S:p:a:fb:i:H:sW:E:@Ahv]
  -q, --quiet                
        Quiet: -q suppress warnings, -qq errors, -qqq all
  -I, --in-format <arg>      
        Input formats are:
                dts - device tree source text
                dtb - device tree blob
                fs  - /proc/device-tree style directory
  -o, --out <arg>            
        Output file
  -O, --out-format <arg>     
        Output formats are:
                dts - device tree source text
                dtb - device tree blob
                asm - assembler source
  -V, --out-version <arg>    
        Blob version to produce, defaults to 17 (for dtb and asm output)
  -d, --out-dependency <arg> 
        Output dependency file
  -R, --reserve <arg>        
        Make space for <number> reserve map entries (for dtb and asm output)
  -S, --space <arg>          
        Make the blob at least <bytes> long (extra space)
  -p, --pad <arg>            
        Add padding to the blob of <bytes> long (extra space)
  -a, --align <arg>          
        Make the blob align to the <bytes> (extra space)
  -b, --boot-cpu <arg>       
        Set the physical boot cpu
  -f, --force                
        Try to produce output even if the input tree has errors
  -i, --include <arg>        
        Add a path to search for include files
  -s, --sort                 
        Sort nodes and properties before outputting (useful for comparing trees)
  -H, --phandle <arg>        
        Valid phandle formats are:
                legacy - "linux,phandle" properties only
                epapr  - "phandle" properties only
                both   - Both "linux,phandle" and "phandle" properties
  -W, --warning <arg>        
        Enable/disable warnings (prefix with "no-")
  -E, --error <arg>          
        Enable/disable errors (prefix with "no-")
  -@, --symbols              
        Enable generation of symbols
  -A, --auto-alias           
        Enable auto-alias of labels
  -h, --help                 
        Print this help and exit
  -v, --version              
        Print version and exit
```

所以设备树的编译与反编译方法如下：

```sh
# 设备树编译
dtc -I dts -O dtb -o <device_name>.dtb <device_name>.dts

# 设备树文件反编译
dtc -I dtb -O dts -o <device_name>.dts <device_name>.dtb
```
## ADB导出设备树的方法

当我们在无法知道源码中具体使用了哪个设备树文件时我们其实可以通过adb导出设备中使用的设备树文件进行解析。

在kernel启动后设备树信息会挂载至`/sys/firmware/`下，在该目录的结构如下：

```
console:/sys/firmware # ls -l
total 0
drwxr-xr-x 3 root root      0 2022-06-27 17:27 devicetree
-r-------- 1 root root 174350 2022-07-06 17:38 fdt
```

其中`fdt`是设备树编译后的源文件（dtb二进制文件），而`devicetree`目录则包含整个设备树的结构信息，目录的根节点对应base目录, 每一个节点对应一个目录, 每一个属性对应一个文件：

```
console:/sys/firmware # ls -lR devicetree/                                     
devicetree/:
total 0
drwxr-xr-x 536 root root 0 2022-06-27 17:27 base

devicetree/base:
total 0
-r--r--r--   1 root root  4 2022-07-06 17:50 #address-cells
-r--r--r--   1 root root  4 2022-07-06 17:50 #size-cells
drwxr-xr-x 100 root root  0 2022-07-06 17:50 1000b000.pinctrl
drwxr-xr-x   2 root root  0 2022-07-06 17:50 __symbols__
......

devicetree/base/1000b000.pinctrl/alspsdefaultcfg:
total 0
-r--r--r-- 1 root root 16 2022-07-06 17:50 name
-r--r--r-- 1 root root  4 2022-07-06 17:50 phandle
......
```

想要导出整个设备树进行查看可以直接将`fdt`文件导出来，之后通过Linux`dtc`命令进行设备树源码还原：

```bash
adb pull /sys/firmware/fdt
# 使用dtc命令将二进制设备树数据转换成为dts格式
dtc -I dtb -O dts fdt -o <device_name>.dts
```

## SSP与W2设备树导出与解析

* W2 MT8788方案设备树反编译

```
/dts-v1/;

/ {
	model = "MT8183"; // 芯片方案
	compatible = "mediatek,mt6771"; // 厂商名称，模块名称
	interrupt-parent = <0x1>;
	#address-cells = <0x2>;
	#size-cells = <0x2>;

    ......
    
	chosen { // 传递给Kernel启动时的参数
	    ......
		bootargs = "console=tty0 console=ttyS0,921600n1 vmalloc=400M slub_debug=OFZPU swiotlb=noforce page_owner=on cgroup.memory=nosocket,nokmem androidboot.hardware=mt6771 firmware_class.path=/vendor/firmware loop.max_part=7 has_battery_removed=0 androidboot.boot_devices=bootdevice,11230000.mmc root=/dev/ram androidboot.vbmeta.device=PARTUUID=8c68cd2a-ccc9-4c5d-8b57-34ae9b2dd481 androidboot.vbmeta.avb_version=1.1 androidboot.vbmeta.device_state=unlocked androidboot.veritymode=disabled androidboot.veritymode.managed=yes androidboot.slot_suffix=_a androidboot.slot=a androidboot.verifiedbootstate=orange bootopt=64S3,32N2,64N2 buildvariant=userdebug ddr_name= ddr_speed=0 androidboot.atm=disabled androidboot.force_normal_boot=1 androidboot.meta_log_disable=0 printk.disable_uart=0 bootprof.pl_t=1469 bootprof.lk_t=8993 bootprof.logo_t=1775 androidboot.serialno=YLH2144100004 androidboot.bootreason=rtc gpt=1 usb2jtag_mode=0 mrdump_ddrsv=yes mrdump_cb=0x11e000,0x2000 mrdump.lk=MRDUMP08 androidboot.dtb_idx=0 androidboot.dtbo_idx=0";
        ......
    }; 
    ......
    
	cpus { // cpu每个核心的信息描述
		#address-cells = <0x1>;
		#size-cells = <0x0>;

		cpu@000 { // cpu0
			device_type = "cpu";
			compatible = "arm,cortex-a53";
			reg = <0x0>;
			enable-method = "psci";
			clock-frequency = <0x8b799100>; // 时钟主频 2.2GHz
			cpu-idle-states = <0x2 0x3 0x4 0x5 0x6 0x7 0x8>;
			phandle = <0x9>;
		};
		......
	};
	......
	i2c@11011000 {// i2c描述信息
		compatible = "mediatek,i2c";
		id = <0x1>;
		reg = <0x0 0x11011000 0x0 0x1000 0x0 0x11000480 0x0 0x80>;
		interrupts = <0x0 0x55 0x8>;
		clocks = <0x15 0x3a 0x15 0x2b>;
		clock-names = "main", "dma";
		clock-div = <0x5>;
		mediatek,hs_only;
		mediatek,skip_scp_sema;
		phandle = <0x57>;
		#address-cells = <0x1>;
		#size-cells = <0x0>;
		clock-frequency = <0x186a0>;
		mediatek,use-open-drain;
        ......
    };
	
	memory { // 内存信息
		items_reserved_mem = <0xa07e 0x0 0x2000 0x0>;
		tee_reserved_mem = <0xe0bf 0x0 0x400 0x0>;
		lca_reserved_mem = <0x0 0x0 0x0 0x0>;
        ......
		device_type = "memory";
		reg = <0x0 0x40000000 0x1 0x80000000>; // 两个内存区块，2G+4G=6G DDR
	};
	......
};
```

* SSP Allwinner A133方案设备树反编译

```
......
/ {
    model = "sun50iw10";
    compatible = "allwinner,a133", "arm,sun50iw10p1";
    interrupt-parent = <0x00000001>;
    #address-cells = <0x00000002>;
    #size-cells = <0x00000002>;
    ......
    cpus { // cpu信息
        #address-cells = <0x00000002>;
        #size-cells = <0x00000000>;
        cpu@0 { // cpu0
            device_type = "cpu";
            compatible = "arm,cortex-a53", "arm,armv8";
            reg = <0x00000000 0x00000000>;
            enable-method = "psci";
            clocks = <0x000000fb>;
            clock-latency = <0x001e8480>; // 时钟主频1.2GHz
            clock-frequency = <0x4ead9a00>;
            dynamic-power-coefficient = <0x000000be>;
            operating-points-v2 = <0x000000fc>;
            cpu-idle-states = <0x000000fd 0x000000fe>;
            #cooling-cells = <0x00000002>;
            cpu-supply = <0x00000041>;
            linux,phandle = <0x000000ee>;
            phandle = <0x000000ee>;
        };
        ......
    };
    ......
    memory@40000000 { //内存信息
        device_type = "memory";
        reg = <0x00000000 0x40000000 0x00000000 0x40000000>; //共1GB DDR
    };
    ......
    soc@03000000 {
        dtbo_version = <0x00000001>;
        compatible = "simple-bus";
        #address-cells = <0x00000002>;
        #size-cells = <0x00000002>;
        ranges;
        device_type = "soc";
        linux,phandle = <0x0000012b>;
        phandle = <0x0000012b>;
        ......    
        s_twi@0x07081400 {
            #address-cells = <0x00000001>;
            #size-cells = <0x00000000>;
            compatible = "allwinner,sun50i-twi";
            reg = <0x00000000 0x07081400 0x00000000 0x00000200>;
            ......
            pmu@34 { //PMIC的信息描述，需参考AXP707 DataSheet
                compatible = "x-powers,axp803"; // 采用全志AXP803（实际搭载AXP707，设备信息与AXP707相同）
                reg = <0x00000034>;
                status = "okay"; //标记可用
                ......
                ac-power-supply { // 插入ac电源的信息描述
                    compatible = "x-powers,axp803-ac-power-supply";
                    status = "okay";
                    pmu_ac_vol = <0x000011f8>; // ac供电的电压限制
                    pmu_ac_cur = <0x00000bb8>; // ac供电的电流限制
                    ......
                };
                usb_power_supply { // 插入usb供电的信息描述
                    compatible = "x-powers,axp803-usb-power-supply";
                    status = "okay";
                    pmu_usbpc_vol = <0x000011f8>; // usb供电的电压限制
                    pmu_usbpc_cur = <0x000001f4>; // usb供电的电流限制
                    ......
                };
                battery-power-supply { // 电池供电的信息描述
                    compatible = "x-powers,axp803-battery-power-supply";
                    status = "okay";
                    ......
                    pmu_bat_temp_enable = <0x00000000>; // 电池是否启用NTC采集温度
                    ......
                    pmu_bat_temp_para1 = <0x00000afe>; // 电池包-25‘C时的TS引脚电压，单位mV
                    pmu_bat_temp_para2 = <0x0000089a>; // 电池包-15‘C时的TS引脚电压，单位mV
                    ......
                };
                ......
            };
            ......
        };
        ......
    };
};
151007
