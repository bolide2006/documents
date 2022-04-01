# 设备树（DTS）简介

[TOC]
## 前言
DTS即Device Tree Source 设备树源码, Device Tree是一种描述硬件的数据结构，它起源于OpenFirmware (OF)。
在Linux 2.6中，ARM架构的板极硬件细节过多地被硬编码在arch/arm/plat-xxx和arch/arm/mach-xxx，比如板上的 platform设备、resource、i2c_board_info、spi_board_info以及各种硬件的platform_data，这些 板级细节代码对内核来讲只不过是垃圾代码。而采用Device Tree后，许多硬件的细节可以直接透过它传递给Linux，而不再需要在kernel中进行大量的冗余编码。
Device Tree由一系列被命名的结点（node）和属性（property）组成，而结点本身可包含子结点。所谓属性，其实就是成对出现的name和 value。在Device Tree中，可描述的信息包括（原先这些信息大多被hard code到kernel中）：

- CPU的数量和类别
- 内存基地址和大小
- 总线和桥
- 外设连接
- 中断控制器和中断使用情况
- GPIO控制器和GPIO使用情况
- Clock控制器和Clock使用情况

## 1. 设备树的基本知识

### 1.1 dts

设备树的源文件的后缀名就是.dts， 每一款硬件平台可以单独写一份 xxxx.dts，所以在 Linux 内核源码中存在大量.dts 文件，对于 arm 架构可以在 arch/arm/boot/dts 找到相应的 dts。

### 1.2 dtsi
值得一提的是，对于一些相同的 dts 配置可以抽象到 dtsi 文件中，这个 dtsi 文件其实就类似于 C 语言当中的.h 头文件，可以通过 C 语言中使用 include 来包含一个.dtsi 文件，例如 arch/arm/boot/dts/zynq-zybo.dts文件有如下内容：

```c
// SPDX-License-Identifier: GPL-2.0+
/*
 *  Copyright (C) 2011 - 2015 Xilinx
 *  Copyright (C) 2012 National Instruments Corp.
 */
/dts-v1/;
#include "zynq-7000.dtsi"

/ {
        model = "Zynq ZYBO Development Board";
        compatible = "digilent,zynq-zybo", "xlnx,zynq-7000";

        aliases {
                ethernet0 = &gem0;
                serial0 = &uart1;
                spi0 = &qspi;
                mmc0 = &sdhci0;
        };

        memory@0 {
                device_type = "memory";
                reg = <0x0 0x20000000>;
        };

        chosen {
                bootargs = "";
                stdout-path = "serial0:115200n8";
        };

        usb_phy0: phy0 {
                #phy-cells = <0>;
                compatible = "usb-nop-xceiv";
                reset-gpios = <&gpio0 46 1>;
        };
};

&clkc {
        ps-clk-frequency = <50000000>;
};

&gem0 {
        status = "okay";
        phy-mode = "rgmii-id";
        phy-handle = <&ethernet_phy>;

        ethernet_phy: ethernet-phy@0 {
                reg = <0>;
                device_type = "ethernet-phy";
        };
};

&qspi {
        u-boot,dm-pre-reloc;
        status = "okay";
};

&sdhci0 {
        u-boot,dm-pre-reloc;
        status = "okay";
};

&uart1 {
        u-boot,dm-pre-reloc;
        status = "okay";
};

&usb0 {
        status = "okay";
        dr_mode = "host";
        usb-phy = <&usb_phy0>;
};
```

除了可以使用#include 除了可以包含.dtsi 文件之外，还可以包含.dts 文件以及 C 语言当中的.h 文
件，这些都是可以的，可以这么理解.dtsi 和.dts 文件语法各方面都是一样的，但是不能直接编译一个.dtsi 文
件。  

### 1.3 dtc

dtc 其实就是 device-tree-compiler，那就是设备文件.dts 的编译器嘛，将.c 文件编译为.o 文件需要用到
gcc 编译器，那么将.dts 文件编译为相应的二进制文件则需要 dtc 编译器， dtc 工具在 Linux 内核的scripts/dtc目录下。使用**make all**或**make dtbs**将dts编译成dtb文件。

### 1.4 dtb

dtb文件就是将.dts文件编译成二进制数据之后得到的文件，这就跟.c文件编译为.o文件是一样的道理，uboot 中使用 bootz 或 bootm 命令向 Linux 内核传递二进制设备树文件(.dtb)。

![在这里插入图片描述](..\images\19e86206b48c96265400749972940d59.png)

## 2. DTS语法
虽然我们基本上不会从头到尾重写一个.dts 文件，大多时候是直接在 SOC 厂商提供的.dts 文件上进行修改。但是 DTS 文件语法我们还是需要详细的学习一遍，因为我们肯定需要修改.dts 文件。大家不要看到要学习新的语法就觉得会很复杂， DTS 语法非常的人性化，是一种 ASCII 文本文件，不管是阅读还是修改都很方便。

设备树的语法specification的可以从参考【1】下载得到。

### 2.1 设备树的结构

设备树用树状结构描述设备信息，组成设备树的基本单元是 node（设备节点）， 这些 node 被组织成树状结构，有如下一些特征：  

- 一个 device tree 文件中只有一个 root node（根节点）

- 除了 root node，每个 node 都只有一个 parent node（父节点）

- 一般来说，开发板上的每一个设备都能够对应到设备树中的一个 node 

- 每个 node 中包含了若干的 property-value（键-值对，当然也可以没有 value） 来描述该 node 的一些
  特性

- 每个 node 都有自己的 node name（节点名字）

- node 之间可以是平行关系，也可以嵌套成父子关系，这样就可以很方便的描述设备间的关系

```c
//设备树结构示意
/{ // 根节点
	node1{ // node1 节点
		property1=value1; // node1 节点的属性 property1
		property2=value2; // node1 节点的属性 property2
		...
	};

	node2{ // node2 节点
		property3=value3; // node2 节点的属性 property3
		...
		node3{ // node2 的子节点 node3
			property4=value4; // node3 节点的属性 property4
			...
		};
	};
};
```

第 1 行当中的’/’就表示设备树的 root node（根节点），所以可知 node1 节点和 node2 节点的父节点都是root node，而 node3 节点的父节点则是 node2， node2 与 node3 之间形成了父子节点关系。 Root node 下面的子节点 node1 和 node2 可以表示为 SoC 上的两个控制器，而 node3 则可以表示挂在 node2 控制器上的某个设备，例如 node2 表示 ZYNQ PS 的一个 I2C 控制器，而 node3 则表示挂在该 I2C 总线下的某个设备，例如eeprom、 RTC 等。

### 2.2 节点和属性
#### 2.2.1 节点
在设备树文件中如何定义一个节点，节点的命名有什么要求呢？在设备树中节点的命名格式如下：
```
[label:]node-name[@unit-address] {
[properties definitions]
[child nodes]
};
```

“[]”中的内容表示可选的，可有也可以没有；节点名字前加上”label”则方便在 dts 文件中被其他的节点引用；其中“node-name”是节点名字，为 ASCII 字符串，节点名字应该能够清晰的描述出节点的功能，比如“uart1”就表示这个节点是 UART1 外设。“unit-address”一般表示设备的地址或寄存基地址，如果某个节点没有地址或者寄存器的话“unit-address”可以不要，比如“cpu@0”、“interruptcontroller@00a01000”。
#### 2.2.2属性
- compatible 属性
compatible 属性也叫做“兼容性”属性，这是非常重要的一个属性！ compatible 属性的值是一个字符串列表， compatible 属性用于将设备和驱动绑定起来。
I.MX6U-ALPHA 开发板上的音频芯片采用的欧胜(WOLFSON)出品的 WM8960， sound 节点的 compatible 属性值如下：compatible = "fsl,imx6ul-evk-wm8960","fsl,imx-audio-wm8960";

- model 属性
  model 属性值也是一个字符串，一般 model 属性描述设备模块信息，比如名字什么的，比
  如：model = "wm8960-audio";

- status 属性
  status 属性看名字就知道是和设备状态有关的， status 属性值也是字符串，字符串是设备的
  状态信息，可选的状态如下表所示：

|     值     |                             描述                             |
| :--------: | :----------------------------------------------------------: |
|   “okay”   |                     表明设备是可操作的。                     |
| "diabled"  | 表明设备当前是不可操作的，但是在未来可以变为可操作的，比如热插拔设备插入以后。至于 disabled 的具体含义还要看设备的绑定文档。 |
|   "fail"   | 表明设备不可操作，设备检测到了一系列的错误，而且设备也不大可能变得可操作。 |
| "fail-sss" |    含义和“fail”相同，后面的 sss 部分是检测到的错误内容。     |

  #address-cells 和#size-cells 属性
这两个属性的值都是无符号 32 位整形， #address-cells 和#size-cells 这两个属性可以用在任何拥有子节点的设备中，用于描述子节点的地址信息。 #address-cells 属性值决定了子节点 reg 属性中地址信息所占用的字长(32 位)， #size-cells 属性值决定了子节点 reg 属性中长度信息所占的字长(32 位)。#address-cells 和#size-cells 表明了子节点应该如何编写 reg 属性值，一般 reg 属性都是和地址有关的内容，和地址相关的信息有两种：起始地址和地址长度， reg 属性的格式一为：
```
reg = <address1 length1 address2 length2 address3 length3……>
```
每个“address length”组合表示一个地址范围，其中 address 是起始地址， length 是地址长度， #address-cells 表明 address 这个数据所占用的字长， #size-cells 表明 length 这个数据所占用的字长，比如:
```
spi4 {
	compatible = "spi-gpio";
	#address-cells = <1>;
	#size-cells = <0>;

    gpio_spi: gpio_spi@0 {
        compatible = "fairchild,74hc595";
        reg = <0>;
    };
};

aips3: aips-bus@02200000 {
    compatible = "fsl,aips-bus", "simple-bus";
    #address-cells = <1>;
    #size-cells = <1>;

    dcp: dcp@02280000 {
    compatible = "fsl,imx6sl-dcp";
    reg = <0x02280000 0x4000>;
    };
};
```
- reg 属性
reg 属性的值一般是(address， length)对。 reg 属性一般用于描述设备地址空间资源信息，一般都是某个外设的寄存器地址范围信息。

- ranges 属性
ranges属性值可以为空或者按照(child-bus-address,parent-bus-address,length)格式编写的数字
矩阵， ranges 是一个地址映射/转换表， ranges 属性每个项目由子地址、父地址和地址空间长度
这三部分组成。
>child-bus-address：子总线地址空间的物理地址，由父节点的#address-cells 确定此物理地址
所占用的字长。
>parent-bus-address： 父总线地址空间的物理地址，同样由父节点的#address-cells 确定此物
理地址所占用的字长。
>length： 子地址空间的长度，由父节点的#size-cells 确定此地址长度所占用的字长。

- name 属性
name 属性值为字符串， name 属性用于记录节点名字， name 属性已经被弃用，不推荐使用name 属性，一些老的设备树文件可能会使用此属性。
- device_type属性
  device_type 属性值为字符串， IEEE 1275 会用到此属性，用于描述设备的 FCode，但是设
  备树没有 FCode，所以此属性也被抛弃了。此属性只能用于 cpu 节点或者 memory 节点。
-  interrupts属性
用到了中断的设备结点透过它指定中断号、触发方法等，具体这个属性含有多少个cell，由它依附的中断控制器结点的#interrupt-cells属性决定。
```c
/*7为中断号，3为trigger方式*/
/**
 *1为上升沿触发
 *2为下降沿触发
 *4为高电平触发
 *8为低电平触发
 */
interrupts = < 7 3 >;
```

#### 2.3 特殊节点
- aliases 子节点
单词 aliases 的意思是“别名”，因此 aliases 节点的主要功能就是定义别名，定义别名的目的就是为了方便访问节点。不过我们一般会在节点命名的时候会加上 label，然后通过&label来访问节点，这样也很方便，而且设备树里面大量的使用&label 的形式来访问节点。
```
aliases {
    can0 = &flexcan1;
    can1 = &flexcan2;
    ethernet0 = &fec1;
    ethernet1 = &fec2;
    gpio0 = &gpio1;
    gpio1 = &gpio2;
    ......
    spi0 = &ecspi1;
    spi1 = &ecspi2;
    spi2 = &ecspi3;
    spi3 = &ecspi4;
    usbphy0 = &usbphy1;
    usbphy1 = &usbphy2;
};
```
- chosen 子节点
chosen 并不是一个真实的设备， chosen 节点主要是为了 uboot 向 Linux 内核传递数据，重点是 bootargs 参数。
>注：uboot会在启动时通过修改内存中dtb中的chosen的值，向kernel传递bootargs参数。

## 3. DTS 常用of函数
### 3.1 查找节点的of函数

#### 3.1.1 of_parse_phandle函数

```c
/*通过当前节点的 phandle 属性，获得一个 device node 节点。*/
struct device_node *of_parse_phandle(const struct device_node *np,
							const char *phandle_name, int index);
```
#### 3.1.2 of_find_node_by_name 函数
```c
/*通过节点名字查找指定的节点*/
struct device_node *of_find_node_by_name(struct device_node *from,
										const char *name);
```
#### 3.1.3 of_find_node_by_type函数
```c
/*通过 device_type 属性查找指定的节点*/
struct device_node *of_find_node_by_type(struct device_node *from,
										const char *type);
```
#### 3.1.4 of_find_compatible_node函数
```c
/*根据 device_type 和 compatible 这两个属性查找指定的节点*/
struct device_node *of_find_compatible_node(struct device_node *from,
											const char *type,
											const char *compatible);
```
#### 3.1.5 of_find_matching_node_and_match函数
```c
/*通过 of_device_id 匹配表来查找指定的节点*/
struct device_node *of_find_matching_node_and_match(struct device_node 								*from,const struct of_device_id *matches,
							const struct of_device_id **match);
```
#### 3.1.6 of_find_node_by_path函数
```c
/*通过路径来查找指定的节点*/
inline struct device_node *of_find_node_by_path(const char *path);
```
### 3.2 查找父子节点的of函数
#### 3.2.1 of_get_parent函数
```c
/*用于获取指定节点的父节点*/
/**
 * of_get_parent - get the parent node of specified node
 * @node: the client
 *
 * Retrun the parent node ponter.
 */
struct device_node *of_get_parent(const struct device_node *node);
```

#### 3.2.2 of_get_next_child函数
```c
/*用迭代的方式查找子节点*/
struct device_node *of_get_next_child(const struct device_node *node,
									struct device_node *prev);
```
### 3.3 查找节点属性的of函数
#### 3.3.1 of_find_property函数
```c
/*用于查找指定的属性*/
property *of_find_property(const struct device_node *np,
							const char *name,
							int *lenp);
```
#### 3.3.2 of_property_count_elems_of_size函数
```c
/*用于获取属性中元素的数量，比如 reg 属性值是一个数组，那么使用此函数可以获取到这个数组的大小*/
int of_property_count_elems_of_size(const struct device_node *np,
									const char *propname,
									int elem_size);
```
#### 3.3.3 of_property_read_u32_index函数
```c
/*用于从属性中获取指定标号的 u32 类型数据值(无符号32位)，比如某个属性有多个 u32 类型的值，那么就可以使用此函数来获取指定标号的数据值.*/
int of_property_read_u32_index(const struct device_node *np,
                                        const char *propname,
                                        u32 index,
                                        u32 *out_value);
```
#### 3.3.4 of_property_read_u32_array 函数
```c
/*这 4 个函数分别是读取属性中 u8、u16、 u32 和 u64 类型的数组数据，比如大多数的reg 属性都是数组数据，可以使用这 4 个函数一次读取出 reg 属性中的所有数据。*/
int of_property_read_u8_array(const struct device_node *np,
							const char *propname,
							u8 *out_values,
							size_t sz);
int of_property_read_u16_array(const struct device_node *np,
							const char *propname,
							u16 *out_values,
							size_t sz);
int of_property_read_u32_array(const struct device_node *np,
							const char *propname,
							u32 *out_values,
							size_t sz);
int of_property_read_u64_array(const struct device_node *np,
							const char *propname,
							u64 *out_values
							size_t sz);

```
#### 3.3.5 of_property_read_u32函数
```c
/*这 4 个函数分别是读取属性中 u8、 u16、 u32 和 u64 类型的数组数据，比如大多数的reg 属性都是数组数据，可以使用这 4 个函数一次读取出 reg 属性中的所有数据。*/
int of_property_read_u8(const struct device_node *np,
                            const char *propname,
                            u8 *out_value);
int of_property_read_u16(const struct device_node *np,
                            const char *propname,
                            u16 *out_value);
int of_property_read_u32(const struct device_node *np,
                            const char *propname,
                            u32 *out_value);
int of_property_read_u64(const struct device_node *np,
                            const char *propname,
                            u64 *out_value);
```
#### 3.3.6 of_property_read_string函数
```c
/*用于读取属性中字符串值*/
int of_property_read_string(struct device_node *np,
							const char *propname,
							const char **out_string);
```
#### 3.3.7 of_n_addr_cells函数
```c
/*用于获取 #address-cells 属性值*/
int of_n_addr_cells(struct device_node *np);
```

#### 3.3.8 of_n_size_cells函数
```c
/*用于获取 #size-cells 属性值*/
int of_n_size_cells(struct device_node *np);
```
### 3.4 其他常用的of函数
#### 3.4.1 of_get_address 函数
```c
/*用于获取地址相关属性，主要是“reg”或者“assigned-addresses”属性值*/
const __be32 *of_get_address(struct device_node *dev,
							int index,
							u64 *size,
							unsigned int *flags);
```
#### 3.4.2 of_iomap函数
```c
/*of_iomap 函数用于直接内存映射，以前我们会通过 ioremap 函数来完成物理地址到虚拟地址的映射，采用设备树以后就可以直接通过 of_iomap 函数来获取内存地址所对应的虚拟地址，不需要使用 ioremap 函数了。*/
void __iomem *of_iomap(struct device_node *np,
						int index);
```

## 参考

1. https://github.com/devicetree-org/devicetree-specification
1. https://blog.csdn.net/weixin_45309916/article/details/109880928
1. https://www.cnblogs.com/aaronLinux/p/5496546.html
1. https://e-mailky.github.io/2019-01-14-dts-1
1. https://e-mailky.github.io/2019-01-14-dts-2
1. https://www.cnblogs.com/tureno/articles/6403946.html
1. 《I.MX6U 嵌入式 Linux 驱动开发指南 V1.5.1》
