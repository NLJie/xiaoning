#PARA/捕获


> [!在开发调试中，常会遇到下面的问题:]
> 1. 外设不工作
> 2. 驱动 probe 失败
> 3. 设备树被篡改
> 4. 引脚冲突、时钟不对、复位异常
> 5. 内核启动后和 DTS 对不上

以上这些绝大部分都能在 `/proc/device-tree` 里找到答案。

这篇文章不讲理论，全是一线排查知识、常用命令、真实案例。

# 1. 先搞懂：`/proc/device-tree` 到底是什么？

`/proc/device-tree` 是内核"最终使用"的设备树，不是我们写的 dts 源码。

1. DTS 源码 → 编译 → DTB；
2. 内核 /bootloader 可能会动态修改 DTB；
3. /proc/device-tree 是内核眼里真正生效的硬件配置；

所以：排查问题一定要看运行时 device-tree，不要只看源码。

# 2. 最实用的 8 条命令

1. 看平台型号(确认是否是A1平台)

```bash
cat /proc/device-tree/name
```

如君正A1系列环境下，

```bash
[root@Ingenic:base]# cat /proc/device-tree/compatible
```

根据此信息判断是不是我们当前的平台。

2. 看所有总线（AHB/APB 是君正核心）

```bash
ls /proc/device-tree/ahb0
```

3. 看时钟控制器（君正几乎所有问题都和时钟有关）

```bash
[root@Ingenic:]# ls /proc/device-tree/clock-controller@0x10000000
```

4. 看中断、定时器（驱动 probe 必查）

```bash
[root@Ingenic:]# ls /proc/device-tree/core-intc@0x12300000
```

5. 看别名、外部时钟（UART/I2C/SPI 常用） 

```bash
[root@Ingenic:]# ls /proc/device-tree/aliases
```

# 3. `/proc/device-tree` 能查哪些问题？(君正 A1)

1. 驱动不加载、probe 失败

排查方法：去对应总线看节点是否存在：

```bash
ls /proc/device-tree/apb/ | grep uart
```

节点不存在 → DTS 没编译进去或被 uboot 覆盖

节点存在但不加载 → 看compatible是否匹配驱动

2. 时钟配置错误（最常见坑）

【现象】：
- 外设时钟不输出
- UART 波特值错误
- SPI 频率不对

I2C 通信异常

【查】：
```bash
ls /proc/device-tree/clock-controller@0x10000000
```

看时钟节点是否存在、是否被正确引用。

3. 中断控制器异常（导致中断不触发）

```bash
ls /proc/device-tree/core-intc@0x12300000
```

intc 节点异常 → 所有外设中断都不能用。

4. 定时器异常（crash、调度异常）

```bash
ls /proc/device-tree/core-ost@0x12000000
```

OST（系统定时器）缺失 → 内核无法调度。

5. extclk /rtcclk 异常（看门狗、RTC 不工作）

```bash
cat /proc/device-tree/extclk
```

外部时钟、实时时钟配置错 → RTC、WDT、UART 精率都会错。

6. aliases 别名错误（串口、I2C 编号错乱）

```bash
ls /proc/device-tree/aliases
```

例如：
```
serial0 = &uart0
i2c0 = &i2c0
```

别名错 → 设备编号错乱，/dev/ttySx 不对。

# 4. 君正 A1 真实排查案例（最实用）

---

【案例 1】：UART 不工作，dts 加了也没用

排查： `ls /proc/device-tree/apb | grep uart`

发现没有 uart 节点。

结论：DTS 没有被正确编译进 dtb，或 uboot 加载了错误的设备树。

【案例 2】：I2C 偶尔通信失败

排查：

```
ls /proc/device-tree/apb | grep i2c
```

如果节点存在，再查时钟：

```
ls /proc/device-tree/clock-controller@0x10000000
```

发现 I2C 时钟未被正确引用。

【案例 3】：内核启动正常，但所有中断不触发

排查：

```
ls /proc/device-tree/core-intc@0x12300000
```

发现 intc 节点缺失。

结论：设备树中断控制器配置错误。

https://mp.weixin.qq.com/s/8xka9DFkreWLYFX20HX_Uw