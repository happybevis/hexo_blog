---
title: 手机卡顿事件与Cpu频率大盗
date: 2018-04-17 14:37:02
tags: Linux & Android 技术
---

近来集中遇到不少user抱怨手机卡顿的case，能引起系统卡顿的原因可以成千上万，但最核心的起因通常避不开*CPU被限制频率*或*多核CPU部分cpu被关核*了。

本文将从几个工作中遇到的实际案例出发（只分析与CPU调控有关的案例），先尽可能地还原出完整的案发现场，再分析具体情况下CPU频率和核心的变化情况，最后和大家一起探索CPU之所以表现“异常”的原因和开发者设计思路。

## 案例分析
总结大量User反馈和描述，常遇到如下四大类卡顿问题：

> - *在手机电池电量较低时，会较为明显地感觉到系统顿挫感。*

> - *手机使用过程中，偶尔发生突然重启，此时系统剩余电量甚至大于20%。*

> - *拍照过程中，点击对焦框偶尔无响应，preview画面变化丢帧明显。*

> - *用户抱怨手机耗电量变大。*

粗略看看这些问题，似乎并没有明显的共同点。但随着研究的深入，隐藏在系统后面的系统调控体系行为逐步暴露出来，从而形成本文的后半部分。

本节将分别从上面四个问题入手，主要介绍问题分析和解决过程，中途可能会引入一些debug手段，最终和大家一起找出谋杀系统性能的凶手。

### 案例一：在手机电池电量较低时，会较为明显地感觉到系统顿挫感

这类问题相对来讲是比较好分析的，通过log分析电量低时是否有cpu 核心变化相关的log，或通过查看此时系统irq是否普遍被migration到某几个cpu，而剩余cpu上的irq响应次数越来越少甚至完全不再响应irq来进行确认(需kernel开启irq migration功能)。

若log讯息较少，不足以断定cpu情况，则有必要安装cpumonitor之类可以实时查看cpu频率和cpu核心运行状况的apk，同时进行放电观察即可确认。

PS：我通常喜欢写脚本做动态观察，具体直接cat /sys/devices/system/cpu/cpuX/cpufreq/scaling_cur_freq driver node来观察每颗cpu核心的频率状态。并通过/sys/devices/system/cpu/cpuX/online node返回值是否为0来确认该cpu核心是否被关闭。
当然，最最好用的小技巧当属直接向/sys/class/power_supply/battery/capacity node写入想要的电量，即可直接观察到不同电量下的cpu表现情况，由于省去了充放电的时间，效率十足。

对于该案例，最后通过脚本观察到系统达到或低于10%电量时，cpu4~7核会发生关闭。有了这个现象，后面我们只需要在电量达到10%的cpu关核的trace中加入dumpstack，即可很容易发现该关核情况为bcl参数设置导致。从bcl对应的code可以发现，该10%的阈值为dtsi中配置产生，修改可以改变其策略。(后文会有详尽的bcl控制讲解)

### 案例二：手机使用过程中，偶尔发生突然重启，此时系统剩余电量甚至大于20%

该案例一般倾向于由Pmic或charger的行为引发，例如charge由于某种原因或算法错误，误上报了0%的电量从而引发Android系统关机。或Pmic寄存器设置的cutoff电压较高，由于电池的物理特性，在电压较低时若有某硬件需要瞬时大电流供给，则电池两端会形成较为明显的压降。若瞬时压降触发到Pmic寄存器预设的关机电压值，就会表现为系统突然掉电(类似于强拔电池的效果)。

对于该案例，一般可以在下次开机时，读取Pmic寄存器中记录上次异常关机原因的寄存器，即可较容易知道真正的关机原因了。当然，我们应同步通知相关部分进行battery的老化检测，确认电池老化是否发生、是否为单体问题、有没可能为某批次共同问题等。除此之外，还要请相关部门量测造成瞬时大电流的组件(例如camera)是否存在优化的可能性。必须要彻底查清问题的本质，才能做出对得起用户的好产品。

### 案例三：拍照过程中，点击对焦框偶尔无响应，preview画面变化丢帧明显

该问题方向比较发散，由于所有user回报的前提都和使用camera相关，且及难复制，故起初一直将该问题倾向于camera模块算法相关。
知道一次偶然的机会，复制到一次在camera拍照瞬间，cpu后四核被关闭的情况，于是才转变注意力到cpu控制这一块来。

如大家所知的，cpu若想引起关核，Qcom或Android在userspace提供了操作接口的方式常见如下几种：

1. 通过向perfd发送请求，让performance-framwork帮忙进行cpu控制。
2. 通过thermal-engine的温度节点和控制方式调控，让某颗sensor温度达到特定条件时进行关核操作。
3. 通过code直接对/sys/device/system 下cpu相关的控制节点进行操控来达到开关核的目的。该情况一般不会被code reviewer通过，且及其不符合设计规范。

有这么多控制方案，且该现象复制难度极大，又无固定复制手法，我们如何去确认该问题发生情景呢？
起初我们做了这样一些实验：

1. 关闭perfd service，让上层不再有机会调用其接口进行限核操作。
2. 关闭thermal-engine service，让thermal不再接管系统操控。

然后进行一次次的现象复制，结果依然复制到一次打开camera时引起的限核动作。
从逻辑上讲，不管我们是否愿意相信，的确可能真的有hard coding的上层service直接操控了cpu online node。搜索了所有的代码，和各种可能的字符串匹配情况，终于在camera包含的第三方库bin档中找到了相关字串。于是请camera team单独出test img，以便确认是否与该库的行为相关。最终事与愿违，拿掉该function的img依然复制到了问题，这下我们前面的思路彻底错乱了~

奇迹总是发生很突然，一次偶然机会，终于让我抓到了一次复制到问题时的完整systrace。
简单介绍一下systrace，就是通过systrace脚本，我们可以搜集Android子系统各个模块的ftrace状况，一些log中没有打印出来的信息，这里都有机会可以看到。顺便说一个小插曲，抓取systrace我们需要使用adb接口。所以最开始尝试抓取strace时都将手机通过usb线链接电脑复制，后来发现只要手机插着usb连着电脑，关核现象就更难复制到。于是我们使用了WiFi adb的方式抓取了这strace，wifi adb大家可能都有所耳闻，但真正用到刀刃上时，才真正完成了一次将知识转为了经验的过程.

通过strace的分析，发现关核是由vbat的interrupt触发，check相关代码，很容易就找到了真凶-->bcl的电压控制模块。bcl的具体细节后面会有专门的模块讲解，这里需要说明的是，kernel的dtsi中，Qcom默认将bcl vbat mitigation值设为了3.2v，前面讲过，电池电量较低时，打开camera瞬间或camera preview瞬间，电池电压被瞬间拉到bcl vbat阈值，而dtsi中定义的阈值行为就是关核。另外就可以解释，由于vbat电压只要恢复到3.3v，kernel立刻就移除了关核动作，所以有些机器电池老化现象不严重，自然电压恢复也快，导致卡顿持续就一瞬间。

自此，真相终于大白，解bug其实和艺术创造类似，有时也是需要一些灵感和运气的。

### 案例四：用户抱怨手机耗电量变大

该类问题从来都是最难解决的问题类别之一，导致好电的原因非常多，且软件硬件均会对其造成影响，故分析起来较为复杂。
整理了一下，引起耗电的常规情形：

1. 由于wake lock未移除或某只driver异常，导致cpu没有进入Deep Sleep mode从而耗电。deep sleep下耗电流通过不超过6mA，但若cpu未sleep，哪怕panel熄灭的情况下，耗电流均值通常也不会低于50mA.

2. Rtc(apk或进程设置alarm定期唤醒cpu)或某个sub-system过于频繁地唤醒cpu，导致cpu高频suspend/resume，从而导致耗电增加。

3. cpu在正常工作模式下，某个子系统的工作电压或clock过高，从而引起较高的耗电。此种情况又是最难分析的一种。

起初，我们分析了该机种在所有情形下的耗电状况，奇怪的是所有case均达标，于是陷入了百思不得其解的忧伤中。
直到一位用户开始抱怨，他的手机cpu频率为何最低也只会降到八百兆Hz而不是正常情况下的300M，此时该案例才有了转机。

从注意到这一现象开始，我们就开始观察身边RD的频率变化情况，经过一段时间的统计，我们发现了一些规律：

1. CPU最小频率被锁定，并不是每台机器都会发生。
2. 最小频率被锁定仅会发生在手机重启并使用过相当一段时间后才能复现。
3. 始终插着usb连接电脑的情况下，不会发生。(所以我们无法通过定时监控最小频率变化的方式解决问题)。
4. 最麻烦的是，发现在放一周能100%复制到现象的机器上，只要烧入我们加入log的boot img(哪怕随意加句log)，就再也无法复现现象。

我们真的没有办法了吗？不是，后来使用ftrace观察cpu freq trace的mitigation状况，总结出一个规律，cpu最小核被限制的情况通通发生在cpu从deep sleep到resume过程中。再次check相关irq的通知状况，最终确定cpu最小核被锁定时，有一颗thermal sensor会在resume过程中发送irq，读取该sensor此时的数据值发现，返回值居然是零下二十度，而正常情况下该值都是室温。

继续追踪相关代码，发现当系统检测到手机温度过低时，会启动最小核锁定的做法，让系统尽可能地多发热来解决低温电池不稳定的情况。当然，这些温度阈值和调节策略也同样可以通过dtsi进行调节。回报该问题给chip厂商，厂商确认为suspend/resume过程中硬件有bug，最后我们拿掉该thermal zone的检测，从而彻底解决了该问题。

这些案例是我曾遇过的一些相对比较经典的问题，希望能抛砖引玉，从思路上给大家一些启发。

后面将详细分析Thermal和bcl的条件过程及策略，系统的了解Thermal和bcl对系统时时性能的控制和管理逻辑。

## thermal与bcl框架剖析

Thermal和系统稳定性管理一直是移动设备的重中之重，随着Kernel版本的升级和Android从N到O的转型，在部分高端平台上，旧的KTM(kernel thermal monitor)模式已被新的Thermal-core模式所替代。

> - BCL（Battery Current Limit ）
 
> - 常规Thermal管理（Linux Kernel Thermal-Core & User Space Thermal-Engine管理）

下面我们将分别从BCL和Thermal两个方向出发，深入学习新框架下BCL和Thermal的管理及配置方法。


### BCL（Battery Current Limit）介绍


> PMIC芯片中的BCL硬件会对系统的供给电压和电流进行采样，当其达到预设安全阈值时，强行呼叫操作系统采取拉低系统功耗的行为（Ex. 降低cpu/Gpu频率）来保证电池和系统的安全性，避免Under-Voltage Lockout (UVLO) 和 Over-Current 的发生。


首先来几个常见的名词解释：

> * Battery current (Ibat)

> * Battery voltage (Vbat)

> * Battery State-of-Charge (SoC)


在未引入Thermal-Core架构之前，Qcom平台可直接通过修改如下几个dtsi节点值来调整BCL阈值及行为。

```
qcom,bcl {
compatible = "qcom,bcl";
qcom,bcl-enable;
qcom,bcl-framework-interface;
qcom,bcl-freq-control-list = <&CPU4 &CPU5 &CPU6 &CPU7>;
qcom,bcl-hotplug-list = <&CPU6 &CPU7>;
qcom,bcl-soc-hotplug-list = <&CPU4 &CPU5 &CPU6 &CPU7>;
qcom,ibat-monitor {
qcom,low-threshold-uamp = <3400000>; //瞬间电流低于3.4A时，撤销hotplug操作。
qcom,high-threshold-uamp = <4200000>; //瞬间电流大于4.2A时，启动hotplug操作。
qcom,mitigation-freq-khz = <768000>;
qcom,vph-high-threshold-uv = <3500000>; //在电压恢复到大于3.5V时撤销cpu hotplug动作。
qcom,vph-low-threshold-uv = <3300000>; //在瞬间电压低于3.3V时启动bcl-soc-hotplug-list对应的hotplug动作
qcom,soc-low-threshold = <10>;  //在电量低于10%时启动bcl-soc-hotplug-list对应的cpu hotplug动作。
qcom,thermal-handle = <&msm_thermal_freq>;
}
```

新的Thermal-Core架构将上面的dtsi控制方式做了较大的改动，但BCL的主要参数配置方式依然是通过dtsi完成。

完整的BCL参数如下(下节Thermal Core介绍中会讲解每个参数的具体意义和使用方式，这里将主要用来介绍调节BCL参数过程中可能会修改到的位置):

```
&thermal_zones {
	ibat-high {
	 polling-delay-passive = <0>;
	 polling-delay = <0>;
	 thermal-governor = "step_wise";
	 thermal-sensors = <&bcl_sensor 0>;

	 trips {
		 ibat_high: low-ibat {
			 temperature = <5000>;
			 hysteresis = <200>;
			 type = "passive";
		 };
	  };
  };
	ibat-vhigh {
	  polling-delay-passive = <0>;
	  polling-delay = <0>;
	  thermal-governor = "step_wise";
	  thermal-sensors = <&bcl_sensor 1>;

	  trips {
		 ibat_vhigh: ibat_vhigh {
		         temperature = <6000>;
			 hysteresis = <100>;
			 type = "passive";
		  };
	  };
	};
	vbat {
	  polling-delay-passive = <100>;
	  polling-delay = <0>;
	  thermal-governor = "low_limits_cap";
	  thermal-sensors = <&bcl_sensor 2>;
	  tracks-low;

	     trips {
		 low_vbat: low-vbat {
		 temperature = <3200>; //对应上面的qcom,vph-low-threshold-uv，
		 电压低于3.2V时，对下方定义的cooling-maps设备进行Thermal Max LIMIT限制操作。
		 hysteresis = <100>; //迟滞值，在governer为"low_limits_cap"情况下，
		 撤销前面对cpu进行Thermal限制操作的阈值为temperature + hysteresis，即3.3V。
		 注意此处的temperature其实就是电压，bcl driver为了能通用到Thermal core框架，
		 在get temperature的实现中对其进行了转换。
			 type = "passive";
		 };
	 };
	    cooling-maps {
		  vbat_cpu4 {
			  trip = <&low_vbat>;
			  cooling-device =
				  <&CPU4 THERMAL_MAX_LIMIT
					  THERMAL_MAX_LIMIT>;
		  };
		  vbat_cpu5 {
			  trip = <&low_vbat>;
			  cooling-device =
				  <&CPU5 THERMAL_MAX_LIMIT
					  THERMAL_MAX_LIMIT>;
		  };
		  vbat_map6 {
			  trip = <&low_vbat>;
			  cooling-device =
				  <&CPU6 THERMAL_MAX_LIMIT
					  THERMAL_MAX_LIMIT>;
		  };
		  vbat_map7 {
			  trip = <&low_vbat>;
			  cooling-device =
				  <&CPU7 THERMAL_MAX_LIMIT
					  THERMAL_MAX_LIMIT>;
		  };
	  };
	};
	vbat_low {
	  polling-delay-passive = <0>;
	  polling-delay = <0>;
	  thermal-governor = "low_limits_cap";
	  thermal-sensors = <&bcl_sensor 3>;
	  tracks-low;

	  trips {
		  low-vbat {
			  temperature = <2800>;
			  hysteresis = <0>;
			  type = "passive";
		  };
	  };
	};
	vbat_too_low {
	     polling-delay-passive = <0>;
	     polling-delay = <0>;
	     thermal-governor = "low_limits_cap";
	     thermal-sensors = <&bcl_sensor 4>;
	     tracks-low;

	  trips {
		  low-vbat {
			  temperature = <2600>;
			  hysteresis = <0>;
			  type = "passive";
		  };
	   };
	};
	soc {
	   polling-delay-passive = <100>;
	   polling-delay = <0>;
	   thermal-governor = "low_limits_cap";
	   thermal-sensors = <&bcl_sensor 5>;
	   tracks-low;

	  trips {
		  low_soc: low-soc {
			  temperature = <10>; //对应于旧ktm时代dtsi中的qcom,soc-low-threshold，
			  即当电量低于10%时对cooling-maps中的设备进行THERMAL_MAX_LIMIT的最大限制。
			  hysteresis = <0>; //迟滞值为0，即电量大于10%时将立即撤销对cpu的限制。
			  type = "passive";
		  };
	  };
	  cooling-maps {
		  soc_cpu4 {
			  trip = <&low_soc>;
			  cooling-device =
				  <&CPU4 THERMAL_MAX_LIMIT
					  THERMAL_MAX_LIMIT>;
		  };
		  soc_cpu5 {
			  trip = <&low_soc>;
			  cooling-device =
				  <&CPU5 THERMAL_MAX_LIMIT
					  THERMAL_MAX_LIMIT>;
		  };
		  soc_map6 {
			  trip = <&low_soc>;
			  cooling-device =
				  <&CPU6 THERMAL_MAX_LIMIT
				  	THERMAL_MAX_LIMIT>;
		  };
		  soc_map7 {
			  trip = <&low_soc>;
			  cooling-device =
				  <&CPU7 THERMAL_MAX_LIMIT
					  THERMAL_MAX_LIMIT>;
		  };
	  };
	};
```

通过查看bcl driver code和上面的dtsi值可以发现，在Thermal-Core的架构下(以后如不特别指出，默认都将dtsi指代为Thermal-Core新架构下的dtsi配置)通过dtsi配置，BCL逻辑会接收如下五种中断：

>	* "bcl-low-vbat"
>	* "bcl-very-low-vbat"
>	* "bcl-crit-low-vbat"
>	* "bcl-high-ibat"
>	* "bcl-very-high-ibat"

但代码逻辑中，bcl driver目前将只会监控vbat(bcl-low-vbat)的interrupt，对于其他四种的interrupt("bcl-very-low-vbat"
/"bcl-crit-low-vbat"/"bcl-very-high-ibat"/bcl-high-ibat/bcl-very-high-ibat)均在bcl driver probe时代码中强行跳过了irq的注册监听。

有趣的是，代码中有这样一条注释：

> The LMH-DCVSh will handle and mitigate for the rest of the ibat/vbat interrupts.

由于BCL driver 在Probe时，会调用bcl_configure_lmh_peripheral() 函数，配置LMh与Pmic相关的寄存器，结合注释可以猜测，系统会在bcl硬件的相关配置后，调用LMh的相关处理逻辑来对系统进行一定的控制（限制clock/power rail
之类）。

顺带介绍下LMh(Limits Management Hardware )，在Qcom的高端系列芯片中，高通为了能更高效地监控并管理系统重loading下的稳定性，特意加入了一个硬件电路LMh用来动态管理峰值情况下的CPU运行电压/频率，以保证大功耗情况下的系统的稳定性。


### Thermal Management In Kernel(Thermal-Core)

Thermal-Core的管理框架，宏观逻辑上讲，是

> Thermal sensor(Thermal Zone) --> Thermal-Core --> Thermal cooling device 

如此的管理方式。其中Thermal sensor(Thermal Zone)对应物理上的Thermal Sensor，Thermal cooling device对应于所有在driver中注册过的可以用来通过各种方式进行Thermal Mitigation的设备(例如cpu可以通过限频来缓解Thermal问题)。
 
我们在BCL一节中已经描述了几个完整的Thermal dtsi配置实例，下面将通过一个例子来介绍】Thermal dtsi item表的的含义：

#### Example Thermal Zone in dtsi

```
&thermal_zones {
	vbat { //该Thermal Zone反应到Thermal Zone driver node下的name。
       polling-delay-passive = <100>; //当被动散热的cooling device达到迟滞值扫描区间时，将进行polling值为100ms间隔的时时监控。
	polling-delay = <0>;
	thermal-governor = "low_limits_cap"; //vbat Thermal Zone想使用的thermal governor为"low_limits_cap"策略（系统支援的thermal policy常见如下几种：
	low_limits_floor、low_limits_cap、user_space、 step_wise，
	其中user_space策略表示该Thermal Zone的kernel部分，将仅仅在达到Thermal阈值时发送一个uevent供userspace处理，
	kernel部分除此之外什么也不做，userspace的处理者通常指的就是thermal-engine）。
       thermal-sensors = <&bcl_sensor 2>; //该Thermal Zone所绑定的实际Thermal Sensor。
	  trips {  //阈值触发点设置域。
		 low_vbat: low-vbat {
		 temperature = <3200>; //温度(对该Thermal Zone而言，返回值反应的其实是电压值)
      		hysteresis = <100>; //迟滞值
	       type = "passive"; //将要绑定到该Thermal Zone的cooling device的类型，
	       目前有三种可选：passive、active、critical。详见[补充1].
	      };
	   };
	   cooling-maps { //将要绑定的cooling device
		  vbat_cpu4 {
		  	trip = <&low_vbat>; //触发源，对应于trips域。
		  	cooling-device =
		  		<&CPU4  THERMAL_MAX_LIMIT
		  			THERMAL_MAX_LIMIT>;  //实际对应的cooling device。详见[补充2]。
	  		};
	  		vbat_cpu5 {
	  			trip = <&low_vbat>;
	  			cooling-device =
		  			<&CPU5 THERMAL_MAX_LIMIT
		  				THERMAL_MAX_LIMIT>;
 			};
 			vbat_map6 {
 				trip = <&low_vbat>;
 				cooling-device =
 					<&CPU6 THERMAL_MAX_LIMIT
 						THERMAL_MAX_LIMIT>;
 			};
 			vbat_map7 {
 				trip = <&low_vbat>;
 				cooling-device =
 					<&CPU7 THERMAL_MAX_LIMIT
 						THERMAL_MAX_LIMIT>;
 			};
 		};
 		```

```
[补充1]：
cooling device type目前有有三种：passive、active、critical。

passive 对应于 THERMAL_TRIP_PASSIVE --> 被动散热设备常见于CPU、GPU等通过调节频率来被动散热的设备.
active 对应于 THERMAL_TRIP_ACTIVE --> 主动散热设备，例如风扇。
critical 对应于 THERMAL_TRIP_CRITICAL –-> 达到阈值时，Thermal Core会调用系统函数对设备进行强行关机。

[补充2]：
cooling-device =<&<mitigation_device> <perf_ceiling> <perf_floor>>;
例如: cooling-device =<&CPU4 7 8>;

perf_ceiling ：设置该cooling device最高可允许的performance级别.
perf_floor ：设置该cooling device最低可允许的performance级别.
数值越低代表对cooling device的限制越小，即越放任系统走更高的performance。数值越大，则说明对cooling device限制越大，系统performance将会变得相对较差。
若perf_ceiling和perf_floor被设为一样，则表明该cooling device将始终运行在该mitigation值上不改变(因为没有提供给系统可条件区间)。

```
	
#### Thermal Core的核心文件组成

```
> thermal_core.c 
> of_thermal.c 
> gov_low_limit.c
> step_wise.c
> devfreq_cooling.c 
> qcom_lmh_dcvs.c 
> bcl_peripheral.c
```

#### 使用Thermal Core框架所需用到的核心函数

```
1. thermal_of_cooling_device_register(parent->of_node, (char *)dev_name(&bd->dev), bd, &bd_cdev_ops); //可用来注册cooling device
				
	 Ex: static struct thermal_cooling_device_ops bd_cdev_ops = {
	  .get_max_state = bd_cdev_get_max_brightness,
	  .get_cur_state = bd_cdev_get_cur_brightness,
	  .set_cur_state = bd_cdev_set_cur_brightness,
	};

2. thermal_register_governor(&thermal_gov_low_limits_cap); //可用来注册自定义的governor。

	ex: static struct thermal_governor thermal_gov_low_limits_cap = {
	  .name		= "low_limits_cap",
	  .throttle	= low_limits_throttle,
	};

3. thermal_zone_of_sensor_register(&pdev->dev, BCL_SOC_MONITOR, soc_data, &soc_data->ops); //用来注册新的thermal zone。

4. thermal_zone_device_update(soc_data->tz_dev, THERMAL_DEVICE_UP); //用来更新Thermal Zone Device对应的各项Thermal返回值或初始化thermal zone中的trip值等操作。

```

#### 流程分析

某个sensor driver probe时，我们首先至少需要实现thermal_zone_of_device_ops结构体中的get_temp和set_trip function用以响应dtsi中的trip值的设置操作,紧接着我们就要开始注册thermal Zone Sensor。

```
struct thermal_zone_of_device_ops {
	int (*get_temp)(void *, int *);
	int (*get_trend)(void *, int, enum thermal_trend *);
	int (*set_trips)(void *, int, int);
	int (*set_emul_temp)(void *, int);
	int (*set_trip_temp)(void *, int, int);
};
```

```
	vbat->ops.get_temp = bcl_read_vbat_and_clear;
	vbat->ops.set_trips = bcl_set_vbat;
	vbat->tz_dev = thermal_zone_of_sensor_register(&pdev->dev,
			  type, vbat, &vbat->ops);
```

接着我们需要调用device update函数，对该Thermal Zone进行首次的硬件寄存器trip值初始化

> thermal\_zone\_device\_update(vbat->tz_dev, THERMAL\_DEVICE\_UP);

>> update\_temperature()

>>> thermal\_zone\_get\_temp() 

>>>> tz->ops->get\_temp(tz, temp) //更新当前温度

>> thermal\_zone\_set\_trips() 

>>> {	
>>>     tz->ops->get\_trip\_temp(tz, i, &trip\_temp);
>>>     tz->ops->get\_trip\_hyst(tz, i, &hysteresis);
>>>
>>>    trip\_low = trip\_temp - hysteresis;
>>> }  //此处统一为用减法方式算出low trip值，在后面具体governor的trip过程中，不同governor会有不同的加减算法. Ex.加 low\_limits\_floor/low\_limits\_cap, 减step\_wise.
>>>    tz->ops->set\_trips(tz, low, high); 

>>>>  -> 调用各thermal\_zone\_device注册时分别实现的set_trip函数，Ex. vbat->ops.set\_trips = bcl\_set\_vbat;

>> handle\_thermal\_trip() 

>>> {
>>> if (type == THERMAL\_TRIP\_CRITICAL || type == THERMAL\_TRIP\_HOT)
>>>
>>>   handle\_critical\_trips(tz, trip, type);    //若dtsi中定义了 thermal trip critical 温度,对于上报的trip type 为THERMAL\_TRIP\_CRITICA type，则此后会立即调用orderly\_poweroff()进行关机行为。
而对于trip source为THERMAL\_TRIP\_HOT type，将只会调用thermal\_zone\_device注册进来的notify函数做对应操作。
>>>
>>> else 
>>>
>>> handle\_non\_critical\_trips(tz, trip, type);//常规运行路径 

>>>> **tz->governor->throttle(tz, trip)** 
>>>> 该函数会会调用到该Thermal zone采用的Thermal governor所对应的throttle函数 --> Ex. low\_limits\_cap governor的throttle函数 .throttle = low\_limits\_throttle (对于user\_space governor的throttle函数来说，其仅仅只会发送一个uevent而已) 

>>>>>	
>>>>>   thermal\_zone\_trip\_update(tz, trip); 

>>>>>> 
>>>>>>   tz->ops->get\_trip\_temp(tz, trip, &trip\_temp);
>>>>>> 
>>>>>>   tz->ops->get\_trip\_type(tz, trip, &trip\_type);
>>>>>> 
>>>>>>   if (tz->ops->get\_trip\_hyst) {
>>>>>> 
>>>>>>   tz->ops->get\_trip\_hyst(tz, trip, &trip\_hyst);
   
>>>>>>   trip\_hyst = **trip\_temp + trip\_hyst;**
>>>>>> 
>>>>>>   } else {
>>>>>> 
>>>>>>   trip\_hyst = trip\_temp;
>>>>>>   }
>>>>>>   //对于low\_limits\_cap governor来说，它的hysteresis是做的加法，换句话说，电池电量大于trip值+hysteresis值后，才会debounce限制。其他governor大部分都是做减法。 Tips：虽然tsensor设备首次probe时调用的是thermal\_zone\_device\_update函数中所包含的thermal\_zone\_set\_trips()，且该函数中所设置的trip\_low = trip\_temp - hysteresis只有减法操作，但只要trip\_high一旦触发trip动作，则governor所对应的 thermal\_zone\_trip\_update 函数会再次根据实际需要，立即修正hysteresis的计算方式。
>>>>>> 
		
>>>>>   list\_for\_each\_entry(instance, &tz->thermal\_instances, tz\_node)
>>>>> 
>>>>>   thermal\_cdev\_update(instance->cdev);
``` 
调用对应colling device注册时注入的cooling_ops
Ex.对于devfreq colling设备，会调用cdev->ops->set_min_state(cdev, min_target)，
这样就会根据dtsi定义的state范围进行最大限制了。
	 static struct thermal_cooling_device_ops devfreq_cooling_ops = {
	  .get_max_state = devfreq_cooling_get_max_state,
	  .get_cur_state = devfreq_cooling_get_cur_state,
	  .set_cur_state = devfreq_cooling_set_cur_state,
	  .get_min_state = devfreq_cooling_get_min_state,
	  .set_min_state = devfreq_cooling_set_min_state,
	};
```
 
>>>>> monitor\_thermal\_zone(tz); //达到trip点，开始polling
>>>>> 
    
#### Thermal Zone对应的driver node管理

我们若想查看当前Thermal调控策略下，每一个colling device此时的mitigation状况，可以通过sysfs中colling_device的driver node获取。
 
 例如：
 ```
/sys/class/thermal # ls
cooling_device0
cooling_device1
cooling_device2
cooling_device3
cooling_device4
...
thermal_zone0
thermal_zone1
thermal_zone2
thermal_zone3
...
```


以BCL的soc thermal Zone为例,可以看到如下资讯：
```
/sys/class/thermal # ls thermal_zone63 
available_policies
cdev0
cdev0_lower_limit
cdev0_trip_point
cdev0_upper_limit
cdev0_weight
cdev1
cdev1_lower_limit
cdev1_trip_point
cdev1_upper_limit
cdev1_weight
cdev2
cdev2_lower_limit
cdev2_trip_point
cdev2_upper_limit
cdev2_weight
cdev3
cdev3_lower_limit
cdev3_trip_point
cdev3_upper_limit
cdev3_weight
integral_cutoff
k_d
k_i
k_po
k_pu
mode
offset
passive_delay
policy
polling_delay
power
slope
subsystem
sustainable_power
temp
trip_point_0_hyst
trip_point_0_temp
trip_point_0_type
type
uevent
```

其中可以找到很多的cdev（0~3）绑定到该Thermal zone中，

接下来我们就以BCL dtsi中所注册的Soc Thermal Zone为例（thermal_zone63），介绍下Thermal Zone中关键driver node与dtsi的对应关系：

```
soc {
	  polling-delay-passive = <100>;
	  polling-delay = <0>;
	  thermal-governor = "low_limits_cap";
	  thermal-sensors = <&bcl_sensor 5>;
	  tracks-low;

	  trips {
		  low_soc: low-soc {
			  temperature = <10>;
			  hysteresis = <0>;
			  type = "passive";
		  };
	  };
	  cooling-maps {
		  soc_cpu4 {
			  trip = <&low_soc>;
			  cooling-device =
				  <&CPU4 THERMAL_MAX_LIMIT
					  THERMAL_MAX_LIMIT>;
		  };
		  soc_cpu5 {
			  trip = <&low_soc>;
			  cooling-device =
				  <&CPU5 THERMAL_MAX_LIMIT
					  THERMAL_MAX_LIMIT>;
		  };
		  soc_map6 {
			  trip = <&low_soc>;
			  cooling-device =
				  <&CPU6 THERMAL_MAX_LIMIT
					  THERMAL_MAX_LIMIT>;
		  };
		  soc_map7 {
			  trip = <&low_soc>;
			  cooling-device =
				  <&CPU7 THERMAL_MAX_LIMIT
				  THERMAL_MAX_LIMIT>;
   		 };
	  };
	};
```
driver node中各节点的含义如下：
```
type [soc] : Thermal Zone Name.
Policy [low_limits_cap] : 该Thermal Zone 对应的policy.
trip_point_0_type [passive] : 对应于dtsi中所定义Thermal Zone trips域中的type（可以有passive active和critical）.
trip_point_0_temp [10] : 对应于dtsi中所定义Thermal Zone trips域中的temperature.
trip_point_0_hyst [0] : 对应于dtsi中所定义Thermal Zone trips域中的 hysteresis.
temp : 当前Thermal sensor读出的温度（bsc driver将其转为了电量值，便于使用Thermal框架统一管理）.
available_policies [low_limits_floor low_limits_cap user_space step_wise]: 为系统支援的thermal policy.
```

下面我们关注下cdev node，该node对应于dtsi中绑定的colling device，就soc而言，对应的自然就是CPU4～7。
进入cdev0的node中可查看到如下node：

```
 cur_state  ： 0
 max_state  ： 24
 min_state  ： 0
 type ： thermal-cpufreq-4
```
   
   这里的min和max state分别对应于dtsi中传入的mitigating范围（对应的值越大，限制效果越强）
   
   对于使用了Lmh硬件的Cpu来讲，我们最好是通过cur_state来获取当前cpu的真实频率状况，而尽量去避免使用/sys/devices/system/cpu/cpu4/cpufreq/scaling_cur_freq来获取频率。因LMh硬件有时候会进行后台调频，但此种情况下并没有调用到cpufreq的update函数，有机会导致此处的读取值不准。

当然我们也同样可以通过/sys/class/thermal/cooling_device* 文件夹获取每一个cooling device的状态。

```
Ex：ls  /sys/class/thermal/cooling_device7
cur_state  0 
max_state  24 
min_state 24
type ： thermal-cpufreq-4  --> cooling device name
```

对于dtsi中定义了colling device的thermal Zone，我们均可以在cooling_device node下找到其对应的state状态。

### Thermal Management In UserSpace

User space中的Thermal管理，主要是通过Thermal-Engine完成，这部分相对以前没有任何改动，这部分资料很多，大家自行了解即可。

但需要特别强调的是：新的Thermal-Core机制下，只有governor为userspace的Thermal Zone，才能保证在Thermal Zone达到trip值后，kernel部分只进行uevent的发送，而将完整的Thermal控制权转交给上层。


## 总结

本文从实际工作案例出发，介绍了工作中debug问题的思路和一些debug方法。同时还分析了BCL和Thermal Core的工作机制，目的是让大家能尽快对Thermal-Core框架快速上手，已达到举一反三的目的。