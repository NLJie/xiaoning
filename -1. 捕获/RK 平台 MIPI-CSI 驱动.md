#PARA/捕获

# 1. 调试驱动前的准备

## 1.1. 确认 Sensor 主控规格

确认当前 `Sensor` 所使用的主控平台支持的`最大分辨率`、`输入频率上限`、`最高每 Lane 的速率`、`支持的 Lane 数`、`数据输入的接口类型`等；

## 1.2. 阅读 Sensor DataSheet 

找 **Sensor** 原厂提供相关 **Sensor Datasheet**，方便查阅相关信息:
- 确认硬件各路电压、上下电时序。
- 确认 **Sensor Chip ID**、**Sensor probe** 时 **check** 使用。
- 确认曝光时间、增益如何设置，帧率如何修改。
- 确认是否支持Slave模式，多摄如何连接、如何配置相关寄存器。
- 测试模式是否支持。
- 确认是否支持高动态模式。
- 确认 **Mirror/Flip** 如何配置。

## 1.3. 确认 Sensor Initialize Settings

获取 **Sensor** 寄存器配置列表。

一般由 **Sensor** 原厂提供。明确具体需求，如**分辨率**、**帧率**、**lane数**、**每lane速率**、**VTS**、**HTS**、**位宽**、**主时钟频率**。

一般至少准备**标准分辨率序列**。根据实际项目需求，确定是否需要准备 **binning**、**Crop** 等类型的特殊分辨率序列。

例如思特威的寄存器配置文件：`cleaned_0x2b_SC530AL27Minput_1080Mbps_2lane_10bit_1920x1080_60fps.ini`,文件中会给出相关信息：

Sensor 名: **SC530AI**
- 主控提供给Sensor的MCLK：27Mhz
- Sensor 输入给主控的 MIPI速率为每Lane：1080Mbps
- Lane数：2
- 位宽：10bit
- 分辨率：1920x1080(1080p)
- 帧率：60pfs

下面给出向 Sensor 厂家申请配置模板：具体条目根据项目实际情况选择。

```c
sensor：XXXX  
mclk: 24MHz/27Mhz/37.5Mhz  
mode：raw8/raw10/raw12/raw16  
output:720p/1080p/1440p/...(分辨率)  
接口: mipi 1/2/4 Lane  
帧率：30/60/120 fps  
主控平台：RK3588/RV1106/RV1103B/RV1126B/...  
项目(产品)形态：⻔铃/⻔铃/IPC/... （可选）
```

# 2. 驱动开发

## 2.1. 驱动代码编写

编写新的 **Sensor** 驱动时，可基于一款规格接近的 `Sensor （kernel/drivers/media/i2c/)` 驱动或相同厂家型号相近的驱动修改 。

以下章节均以思特威 **SC200AI Sensor** 驱动开发为例，说明 **RK** 平台 Sensor 驱动开发的流程。
### 2.1.1. Sensor 重要参数

**Sensor mode** 结构体定义了 Sensor 的重要参数，不同分辨率都需填充准确的参数信息。

```c
struct sc200ai_mode {
	u32 bus_fmt;		// 输出像素格式
	u32 width;			// 输出宽度
	u32 height;			// 输出高度
	struct v4l2_fract max_fps;	// 最大输出帧率
	u32 hts_def;		// 默认行总周期
	u32 vts_def;		// 默认帧总周期
	u32 exp_def;		// 默认曝光值
	u32 mipi_freq_idx;	// 链接频率索引值
	u32 bpp;			// 每像素所用bit数
	const struct regval *reg_list;	// 寄存器列表
	u32 hdr_mode;		// 宽动态模式
	u32 vc[PAD_MAX];	// 虚拟通道数组
};
```

`supported_modes` 结构体数组保存了支持的不同分辨率的 `mode` 结构体，用来在 `probe` 初始化时或切换分辨率时遍 历、匹配。

以 **SC200AI** 为例：

```c
static const struct sc200ai_mode supported_modes[] = {
	{	// 1080p 30fps 线性配置参数
		.width = 1920,
		.height = 1080,
		.max_fps = {
			.numerator = 10000,
			.denominator = 300000,
		},
		.exp_def = 0x0080,
		.hts_def = 0x44C * 2,
		.vts_def = 0x0465,
		.bus_fmt = MEDIA_BUS_FMT_SBGGR10_1X10,
		.reg_list = sc200ai_linear_10_1920x1080_30fps_regs,
		.hdr_mode = NO_HDR,
		.bpp = 10,
		.mipi_freq_idx = 1,
		.vc[PAD0] = V4L2_MBUS_CSI2_CHANNEL_0,
	}, {	// 540p 120fps 线性配置参数
		.width = 960,
		.height = 540,
		.max_fps = {
			.numerator = 10000,
			.denominator = 1200000,
		},
		.exp_def = 0x0080,
		.hts_def = 0x420,
		.vts_def = 0x0249,
		.bus_fmt = MEDIA_BUS_FMT_SBGGR10_1X10,
		.reg_list = sc200ai_linear_10_960x540_120fps_regs,
		.hdr_mode = NO_HDR,
		.bpp = 10,
		.mipi_freq_idx = 1,
		.vc[PAD0] = V4L2_MBUS_CSI2_CHANNEL_0,
	}, {	// 1080p 30fps 宽动态配置参数
		.width = 1920,
		.height = 1080,
		.max_fps = {
			.numerator = 10000,
			.denominator = 300000,
		},
		.exp_def = 0x0080,
		.hts_def = 0x44C * 2,
		.vts_def = 0x08CC,
		.bus_fmt = MEDIA_BUS_FMT_SBGGR10_1X10,
		.reg_list = sc200ai_hdr_10_1920x1080_regs,
		.hdr_mode = HDR_X2,
		.bpp = 10,
		.mipi_freq_idx = 1,
		.vc[PAD0] = 1,
		.vc[PAD1] = 0,//L->csi wr0
		.vc[PAD2] = 1,
		.vc[PAD3] = 1,//M->csi wr2
	},
};
```

参数说明：
- **bus_fmt** 是 **Sensor** 输出的像素排列格式，一般在Sensor 给出的序列里面有多少bit(比如10bit)，BGGR是像素排列顺序，即Bayer格式，Bayerl顺序错了不影响出图(但会出现色偏)。

- **width**、**height** 是 **Sensor** 输出的有效宽、高，是最重要的参数，一般是小于 hts 和 vts。

- **max_fps** 是当前 **mode** 支持输出的最大帧率，是用 **denominato/numerator** 为帧率，一般 **numerator** 不改，如果是准确的30帧，denominator 就是 300000。

- **exp_def** 默认的曝光值（根据初始化寄存器定义的值填写）。

- **hts_def** 默认一行的总点数（根据初始化寄存器定义的值填写）。

- **vts_def** 默认一帧的总行数（根据初始化寄存器定义的值填写），此值不要写错，否则会影响输出帧率。

- **globalreg_list** 是 Sensor 的通用初始化寄存器，若没有，可以为空。

- **reg_list** 是 Sensor 厂商给的寄存器序列。

- **hdrmode** 是对应当前参数的hdr模式，NOHDR表示线性模式，HDRX2/HDRX3表示宽动态，使用2帧或3帧融合。

- **link_freq_idx** 对应下面 **link_freq_menu_items** 数组中的频率索引,一般厂家给的是每 Lane 速率 Mbps，因为 MIPI 双边沿采样的原因，MIPI 频率是采样率的一半，以第一组线性为例，厂家给的序列里写的速率是 720Mbps，MIPI频率则是360M，也就是link_freq链接频率是 360M。

**注意：** 重要提示:这个参数不对容易导致收图失败，要再三确认。
- **bpp** 是表明每个像素占用的 bit 位数。

**注意:** 线性与宽动态的配置区别有两处，一是hdr_mode的不同，二是vC虚拟通道数组的不同。

- 线性：
```
hdr_mode = NO_HDR,Vc[PAD0] = V4L2_MBUS_CSI2_CHANNEL_0;
```

- 宽动态：
```
hdr_mode = HDR_X2/HDR_X3,
Vc[PAD0] = V4L2_MBUS_CSI2_CHANNEL_0;
Vc[PAD1] = V4L2_MBUS_CSI2_CHANNEL_0;
Vc[PAD2] = V4L2_MBUS_CSI2_CHANNEL_1;
Vc[PAD3] = V4L2_MBUS_CSI2_CHANNEL_1;
```

### 2.1.2. Sensor 重要寄存器

以下简单罗列并介绍了 **Sensor** 驱动开发中可能使用到的一些重要寄存器，根据 **Sensor** 厂家、型号不同，可能存在一些差异。

#### 2.1.2.1. Chip id

**Sensor** 芯片ID，是每款 Sensor 的身份证，用来在驱动加载时识别 Sensor 是否被正确加载。Sensor probe 接口中会 check 每款芯片的Chip id，若读取到该 Sensor Chip id，则认为 Sensor 设备识别正常，I2C 读写通讯正常，各路电压正常。
```c
#define CHIP_ID				0xcb1c
#define SC200AI_REG_CHIP_ID		0x3107
```

注意：
有些 Sensor 没有 Chipid，无法验证 Sensor 设备是否正常识别和挂载时，可以选取某个(某些个)只读寄存器读取，验证读取结果是否是默认值来确认。
#### 2.1.2.2. Ctrol mode

**Sensor** 控制起流、停流的控制寄存器，一般等Sensor 所有寄存器都写入成功后，写入该寄存器可以控制Sensor 出流或停流。
```c
#define SC200AI_REG_CTRL_MODE		0x0100
#define SC200AI_MODE_SW_STANDBY		0x0
#define SC200AI_MODE_STREAMING		BIT(0)
```

注意：请不要在寄存器序列中配置该寄存器的值，我们使用起流和停流的接口调用单独控制该寄存器的写入。

#### 2.1.2.3. Exposure reg

曝光控制寄存器，根据线性/宽动态模式，可能会配置不同的曝光控制寄存器，具体根据 Sensor 手册中说明。

```c
#define SC200AI_REG_EXPOSURE_H		0x3e00	// 长曝光最高位
#define SC200AI_REG_EXPOSURE_M		0x3e01	// 长曝光中间位
#define SC200AI_REG_EXPOSURE_L		0x3e02	// 长曝光最低位
#define SC200AI_REG_SEXPOSURE_H		0x3e22	// 短曝光最高位	
#define SC200AI_REG_SEXPOSURE_M		0x3e04	// 短曝光中间位
#define SC200AI_REG_SEXPOSURE_L		0x3e05	// 短曝光最低位
#define	SC200AI_EXPOSURE_MIN		1		// 最新曝光行，一般不改，除非sensor有最小曝光限制
#define	SC200AI_EXPOSURE_STEP		1		// 曝光步进，一般不用修改
```

注意：
- 有些Sensor的曝光寄存器可能不止16bit的(比如高字节+低字节，如这里是24bit，高、中、低三个字节)，需要分多次写入。也有可能是长帧或短帧的，需要根据实际情况写入。
- 增益值可能需要按照特定公式计算，比如：
	 - 模拟增益可能是线性值，也可能是查表；
	 - 数字增益可能是浮点倍数，但写入的是定点数(如Q8.8格式)；

#### 2.1.2.4. Gain reg
增益控制寄存器，分为模拟增益控制寄存器和数字增益控制寄存器。

每种增益控制寄存器有时分主增益(Coarsegain，粗增益，大粒度的，一般对应的是增益的整数部分，用来对增益的粗调)和精细增益(Finegain，精增益，细粒度的，一般对应的是增益的小数部分，用来对增益的微调)。

下面简单介绍下相关概念：

**模拟增益(Analog Gain)**
- 是在模拟信号域(ADC之前)对传感器输出的电压信号进行放大；
- 优点:对图像噪声影响相对小一些(相比数字增益)；
- 缺点:放大倍数有限，过高会引入更多噪声和非线性失真；
- 对应寄存器:ANA_GAIN(主)、ANA_FINE_GAIN(微调)；

**数字增益(Digital Gain)**
- 是在数字信号域(ADC之后)对图像的原始数字值(比如10bit/12bit数据)进行乘法放大；
- 优点:实现简单，灵活；
- 缺点:本质是对数字值做乘法，会放大噪声和量化误差，尤其在低光下容易产生彩色噪点；
- 对应寄存器:DIG_GAIN(主)、DIG_FINE_GAIN(微调)；

**增益调节的目标是什么?**
- 在低光环境下提升画面亮度(通过提高增益)；
- 在自动曝光(AE)算法中，动态调整增益与曝光时间(shutter/linecount)来达到目标亮度；
- 控制图像信噪比(SNR)、动态范围等；

```c
#define SC200AI_REG_DIG_GAIN		0x3e06	// 数字主增益
#define SC200AI_REG_DIG_FINE_GAIN	0x3e07	// 数组精增益
#define SC200AI_REG_ANA_GAIN		0x3e08	// 模拟主增益
#define SC200AI_REG_ANA_FINE_GAIN	0x3e09	// 模拟精增益
#define SC200AI_REG_SDIG_GAIN		0x3e10	// 短帧的数字主增益
#define SC200AI_REG_SDIG_FINE_GAIN	0x3e11	// 短帧的数组精增益
#define SC200AI_REG_SANA_GAIN		0x3e12	// 短帧的模拟主增益
#define SC200AI_REG_SANA_FINE_GAIN	0x3e13	// 短帧的模拟精增益
#define SC200AI_GAIN_MIN			0x0040	//增益最小值
#define SC200AI_GAIN_MAX			(54 * 32 * 64)       //53.975*31.75*64	增益最大值
#define SC200AI_GAIN_STEP			1
#define SC200AI_GAIN_DEFAULT		0x0800	//默认增益值
#define SC200AI_LGAIN				0		//⻓帧增益
#define SC200AI_SGAIN				1		//短帧增益
```

**注意：** 在支持HDR高动态模式时，设置 **Sensor** 增益的时候，可能需要设置长帧和短帧增益，此时定义了 `SC200AI_LGAIN`、`SC200AI_SGAIN` 来区分。

#### 2.1.2.5. Group hold


**Group Hold（组保持/群组保持）** 是CMOS图像传感器中用于控制**曝光(shutter)**、**增益等参数同步更新**的一种机制，主要目的是:

> 在同一帧图像的采集过程中，保证所有相关参数(如模拟增益、数字增益、行曝光时间等)保持一致，避免参数中途变化导致图像出现异常(如竖纹、明暗不均、部分过曝/欠曝等)。

具体来说：在CMOS传感器中，很多关键参数（比如曝光时间、模拟/数字增益等）是通过寄存器写入来配置的;

但这些寄存器的修改，**通常并不会立即生效**，而是要等到**下一个帧/行周期开始时才生效**；如果你在**一帧图像的采集过程中，修改了这些参数**，就可能导致**图像的一部分用旧参数、另一部分用新参数**，出现明显的分界线或异常；**Group Hold 就是用来“冻结这些参数的更新时机”，让它们在指定的时刻统一生效，从而保证一帧图像内参数的一致性**；

```c
#define SC200AI_REG_GROUP_HOLD      0x3812  // Group hold 控制寄存器
#define SC200AI_GROUP_HOLD_START    0x00    // 开启 Group hold（冻结参数更新）即：Group hold Start
#define SC200AI_GROUP_HOLD_END      0x30    // 关闭 Group hold（释放参数更新）即：Group hold End
```

#### 2.1.2.6. VTS

在 CMOS 图像传感器（CIS）或摄像头模组的上下文中，`HTS` 和 `VTS` 是两个非常关键的时序参数。本节重点介绍VTS。

**VTS** 是 Vertical Total Time或 Vertical Total Lines 的简称。中文俗称:帧总周期/帧总行数。它表示传感器输出一帧完整图像(包括所有有效行+消隐行)所用的总行数(或总时间)。通常是行数(Iines)，也可以换算为时间(比如ms)，取决于帧率与VTS。
包含的内容:

1. Active Lines(有效行数):即实际图像数据的行数，比如1080(对于1920x1080分辨率中的高度)。
2. VSYNC(场同步信号)
3. 帧消隐区(Vertical Blank/前廊/后廊):用于传感器切换帧、内部处理等。

VTS=有效行数+垂直消隐行数

**HTS**、**VTS** 与图像分辨率、帧率的关系:

1. 帧率(FPS)的计算

帧率关键计算公式是:

![[resource/Pasted image 20260427201522.png]]

其中：  
- **PCLK**：Pixel Clock，即每个像素的时钟频率（单位通常是 MHz 或 Hz），表示每秒传输多少个像素时钟。
- **HTS**：每行总时钟周期数（决定一行花多长时间）；
- **VTS**：一帧总行数（决定一帧花多少行）；

所以，HTS 和 VTS 共同决定了传感器的帧率！如果你**想提高帧率（FPS）**，你可以：

- 减少 **VTS （总行数，即减少垂直消隐）**、或提高 **PCLK （像素时钟）**；

反之，**如果想降低帧率**（比如低光下需要更长曝光），可以：

- 增加 **VTS** 或 降低 **PCLK**；

**HTS** 与 **VTS** 的核心区别：

| 项目     | HTS（Horizontal Total）    | VTS（Vertical Total）  |
| ------ | ------------------------ | -------------------- |
| 方向     | 水平方向（一行内）                | 垂直方向（一帧内）            |
| 含义     | 一行总的时钟周期数（包含有效像素+消隐）     | 一帧总行数（包含有效行+消隐）      |
| 包含内容   | 有效像素 + HSYNC + 水平消隐区     | 有效行数 + VSYNC + 垂直消隐区 |
| 影响     | 行频、数据传输时许                | 帧率、帧时间               |
| 与帧率的关系 | FPS = PCLK / (HTS x VTS) | 增大 VTS 会降低频率，减小则提高   |
| 单位     | 时钟周期数（或像素时钟数）            | 行数（lines）            |
| 典型配置位置 | 传感器寄存器（行时序）              | 传感器寄存器（帧时序）          |

```c
#define SC200AI_REG_VTS_H 0x320e  
#define SC200AI_REG_VTS_L 0x320f
```

#### 2.1.2.7. Mirror / Flip

镜像（**Mirror**）翻转（**Flip**）控制寄存器。

在图像传感器(Sensor)的上下文中，Mirror(镜像)和Flip(翻转)是两个用于控制图像输出方向或方位的功能，它们影响的是Sensor输出的图像内容的左右或上下方向，属于图像几何变换的范畴。

1. **Mirror(镜像)** 是指将图像在水平方向(X轴)上做左右对称翻转，类似于人照镜子时看到的效果。就像你照镜子时，你的左侧在镜子里显示在右侧，这就是 **水平镜像(Mirror)**。
在Sensor中的作用:
- 通过设置Sensor 寄存器中的 Mirror 位(或 Horizontal Flip)，可以让 Sensor 输出的图像在水平方向翻转。
- 常用于:
	- Sensor横向安装反了，导致图像左右相反
	- 镜头安装方向导致需要水平翻转图像

2. **Flip(翻转**) 是指将图像在垂直方向(Y轴)上做上下对称翻转，也就是将图像上下倒置。就像你把一张照片旋转180°上下倒置，或者从天花板往下看图像，就是 **垂直翻转(Flip)** 的效果。

在Sensor中的作用：

- 通过设置Sensor 寄存器中的 **Flip** 位(或 **Vertical Flip)**，可以让 Sensor 输出的图像在垂直方向翻转。
- 常用于:
	- **Sensor 倒装 (比如摄像头倒着装在设备中)**；
	- 需要调整图像方向以匹配显示要求；

Mirror 和 Flip 的组合效果：

| 设置                | 效果     | 实际表现                       |
| ----------------- | ------ | -------------------------- |
| 无 Mirror / 无 Flip | 正常输出   | 图像无翻转，和人眼看到的一致（默认方向）       |
| Mirror（水平镜像）      | 左右翻转   | 图像左右对调，像照镜子一样              |
| Flip（垂直翻转）        | 上下翻转   | 图像上下颠倒，像倒置的照片              |
| Mirror + Flip     | 旋转180度 | 图像整体看起来像是旋转了 180 度（上下左右全反） |
在大多数 CMOS 图像传感器 **（如 Sony、OmniVision、SmartSens、GalaxyCore、Onsemi 等）中，Mirror 和  Flip 功能通常是通过寄存器配置的** 。
```c
#define SC200AI_FLIP_MIRROR_REG		0x3221	// 镜像翻转控制器

#define SC200AI_FETCH_MIRROR(VAL, ENABLE)	(ENABLE ? VAL | 0x06 : VAL & 0xf9)	// 设置镜像
#define SC200AI_FETCH_FLIP(VAL, ENABLE)		(ENABLE ? VAL | 0x60 : VAL & 0x9f)	// 设置翻转
```

3. v4l2-ctl 命令设置镜像与翻转
```bash
v4l2-ctl --set-ctrl=horizontal_flip=1 -d /dev/v4l-subdev2 # 启用水平镜像  
v4l2-ctl --set-ctrl=horizontal_flip=0 -d /dev/v4l-subdev2 # 禁用水平镜像  

v4l2-ctl --set-ctrl=vertical_flip=1 -d /dev/v4l-subdev2 # 启用垂直翻转  
v4l2-ctl --set-ctrl=vertical_flip=0 -d /dev/v4l-subdev2 # 禁用垂直翻转  

v4l2-ctl --set-ctrl=horizontal_flip=1 --set-ctrl=vertical_flip=1 -d /dev/v4l-subdev2  # 同时启用镜像和翻转  
v4l2-ctl --set-ctrl=horizontal_flip=0 --set-ctrl=vertical_flip=0 -d /dev/v4l-subdev2  # 同时禁用镜像和翻转
```

#### 2.1.2.8. Test pattern

测试图案寄存器，是指 图像传感器（如 CMOS/CCD Sensor）在特定模式下，不采集真是光学图像，而是直接输出由传感器内部生成的固定图案（如彩色条纹、渐变、棋盘格、单色块等）的一种功能；

简单来说就是：让摄像头不拍真实场景，而是输出一个传感器内部生成的”假图像”，用于测试、调试或验证目的。

这些图案通常是:

- 彩色条纹(Color Bars)
- 渐变色(Gradient)
- 棋盘格(Checkerboard)
- 单色(纯红/绿/蓝/黑/白)
- 网格(Grid)
- 固定图案噪声(Fixed Pattern Noise,FPN)

1. **Test pattern 的用途和作用**
	1. Sensor 功能验证、调试

	在 Sensor 开发、验证阶段，工程师可以通过 Test pattern 快速确认：
	- Sensor 是否正常输出图像数据；
	- MIPI CSI-2 接口是否工作正常（数据通路是否 OK）；
	- 图像的分辨率、色彩格式（如 RAW、RGB、YUV）是否正确；
	- ISP 或 下游模块是否能这却接收和处理数据流；

	2. ISP 调试与算法验证

	在 ISP （图像信号处理器）开发阶段，Test pattern 可以用来：
	- 验证 ISP 各模块功能（如 Demosaic、3A、Gamma、CSC、Sharpening 等）；
	- 测试不同输入图案下 ISP 的处理效果；
	- 检查色彩还原、亮度、对比度、噪声抑制算法是否正常工作；

	因为Test Pattern是已知的、可控的输入，所以非常适合用来做定量分析和算法调优。
	
	比如：你输入一个纯色块（比如全红），看ISP输出是否也是纯红，可以验证白平衡/颜色校正模块是否异常;
	
	输入渐变图案，可以用来测试Gamma、动态范围、tone mapping 等功能。

	3. 摄像头模组调试 & 校准

	在摄像头模组生产 / 校准阶段，Test pattern 可以用于：
	- 验证镜头模组 + Sensor + ISP 的数据通路；
	- 做自动对焦（AF）、自动曝光（AE）、自动白平衡（AWB）算法的模拟测试；
	- 在无真实光源的环境下进行基本功能测试；

	4. 故障排查 / 异常分析

	当摄像头输出图像异常（如偏色、花屏、条纹、全黑、全白）时，你可以：
	- 先开启 Test pattern ，看是否仍然异常
		- 如果 Test pattern 输出正常：说明可能出现在光学部分，如镜头脏污、无光等；
		- 如果 Test pattern 输出异常：说明问题可能在 Sensor 、MIPI 通路、ISP 或 电源等；
	 
	所以 Test pattern 是要给非常重要的排查工具，用来区分问题是出现在 光学链路还是 电信号链路上； 

	5. 工厂生产测试

	在 摄像头/Sensor 的工厂量产测试环节，Test Pattern 可用于：
	- 自动化测试Sensor输出是否正常；
	- 检查坏点(Dead Pixel)、亮点(Hot Pixel)、固定噪声等；
	- 验证数据完整性、色彩准确性、动态范围等指标；


| Test pattern 类型        | 描述                                      | 用途                |
| ---------------------- | --------------------------------------- | ----------------- |
| Solid Color （单色）       | 整幅图像为纯色（如 纯红 / 绿 / 蓝 / 黑 / 白）           | 检查色彩通道、坏点         |
| Color Bars（彩色条纹）       | 类似电视侧视图的彩色横条（如 纯红 / 绿 / 蓝 / 黑 / 白 等的组合） | 验证色彩输出、ISP 处理     |
| Gradient（渐变）           | 从黑到白或红绿蓝的平滑过渡                           | 测试动态范围、Gamma、色调映射 |
| Checkerboard（棋盘格）      | 黑白相间的方格图案                               | 测试锐利度、边缘处理、压缩算法   |
| Moving Pattern（动态图案）   | 一些 Sensor 支持动态变化的测试图                    | 测试时序、缓存、延迟        |
| Random / Noise Pattern | 模拟噪声图案                                  | 测试噪声抑制算法          |
| Cross Hatch / Grid     | 交叉线网格                                   | 检查几何失真、对齐         |

```c
#define SC200AI_REG_TEST_PATTERN	0x4501		// 测试图案控制寄存器
#define SC200AI_TEST_PATTERN_BIT_MASK	BIT(3)	// 测试图案开机掩码
```

## 2.2. 驱动框架与说明

Carmera Sensor 采用 I2C 与主控进行交互，目前 Sensor 的驱动按照 I2C 设备驱动方式实现，Sensor 的驱动同事采用 V4l2 Subdev 的方式实现与 host driver 之间的交互；

### 2.2.1. 数据类型说明

#### 2.2.1.1. struct i2c_driver

【说明】：定义 I2C 设备驱动信息；

【定义】：`kernel-6.1/include/linux/i2c.h`
```c
struct i2c_driver {
	...
	int (*probe)(struct i2c_client *client, const struct i2c_device_id *id);
	void (*remove)(struct i2c_client *client);
	...
	struct device_driver driver;
	const struct i2c_device_id *id_table;
	...
};
```
【关键成员】：

| 成员名称       | 数据类型                              | 作用描述                                                                                      |
| ---------- | --------------------------------- | ----------------------------------------------------------------------------------------- |
| `probe`    | `int (\*)(struct i2c_client \*）`  | 设备探测函数，当驱动匹配到设备时调用；                                                                       |
| `remove`   | `void (\*)(struct i2c_client \*)` | 设备移除清理函数；                                                                                 |
| `driver`   | `struct device_driver`            | 核心驱动结构,当 `of_match_table` 中的 `compatible` 域和 `dts` 文件的 `compatible` 域匹配时，`.probe` 函数才会被调用 |
| `id_table` | `const struct i2c_device_id *`    | 设备匹配表, 如果 `kernel` 没有使用 `of_match_table` 和 `dts` 注册设备进行进行匹配，则 `kernel` 使用该 `table` 进行匹配   |
【示例】：
```c
#if IS_ENABLED(CONFIG_OF)
static const struct of_device_id sc200ai_of_match[] = {
	{ .compatible = "smartsens,sc200ai" },
	{},
};
MODULE_DEVICE_TABLE(of, sc200ai_of_match);
#endif

static const struct i2c_device_id sc200ai_match_id[] = {
	{ "smartsens,sc200ai", 0 },
	{ },
};

static struct i2c_driver sc200ai_i2c_driver = {
	.driver = {
		.name = SC200AI_NAME,
		.pm = &sc200ai_pm_ops,
		.of_match_table = of_match_ptr(sc200ai_of_match),
	},
	.probe		= &sc200ai_probe,
	.remove		= &sc200ai_remove,
	.id_table	= sc200ai_match_id,
};

static int __init sensor_mod_init(void)
{
	return i2c_add_driver(&sc200ai_i2c_driver);
}

static void __exit sensor_mod_exit(void)
{
	i2c_del_driver(&sc200ai_i2c_driver);
}
```

#### 2.2.1.2. struct v4l2_subdev_ops

【说明】：`struct v4l2_subdev_ops` 是 Linux V4L2 框架中子设备（Sub-device）操作集的核心结构体，定义了各类硬件模块（如 Sensor、ISP、解码器）的功能接口；

【定义】：`kernel-6.1/include/media/v4l2-subdev.h`
```c
struct v4l2_subdev_ops {
	const struct v4l2_subdev_core_ops	*core;
	const struct v4l2_subdev_tuner_ops	*tuner;
	const struct v4l2_subdev_audio_ops	*audio;
	const struct v4l2_subdev_video_ops	*video;
	const struct v4l2_subdev_vbi_ops	*vbi;
	const struct v4l2_subdev_ir_ops		*ir;
	const struct v4l2_subdev_sensor_ops	*sensor;
	const struct v4l2_subdev_pad_ops	*pad;
};
```
【关键成员】：

| 成员类型  | 结构体                                          | 功能描述               | 适用硬件                       |
| ----- | -------------------------------------------- | ------------------ | -------------------------- |
| 核心操作  | `const struct v4l2_subdev_core_ops	*core;`   | 基础控制（电源/中断/日志等）    | 所有子设备                      |
| 视频流控制 | `const struct v4l2_subdev_video_ops	*video;` | 视频流参数（帧率/格式/HDR 等） | Sensor / ISP / 编码器         |
| 面板控制  | `const struct v4l2_subdev_pad_ops	*pad;`     | 像素格式/分辨率/裁剪设置 (关键) | Sensor / ISP /MIPI-CSI 接收器 |
【示例】：
```c
static const struct v4l2_subdev_ops sc200ai_subdev_ops = {
	.core	= &sc200ai_core_ops,
	.video	= &sc200ai_video_ops,
	.pad	= &sc200ai_pad_ops,
};
```

#### 2.2.1.3. struct v4l2_subdev_core_ops

【说明】：V4L2 子设备驱动中用于实现核心控制功能的操作集。

【定义】：`kernel-6.1/include/media/v4l2-subdev.h`
```c
struct v4l2_subdev_core_ops {
	int (*log_status)(struct v4l2_subdev *sd);
	int (*s_io_pin_config)(struct v4l2_subdev *sd, size_t n,
				      struct v4l2_subdev_io_pin_config *pincfg);
	int (*init)(struct v4l2_subdev *sd, u32 val);
	int (*load_fw)(struct v4l2_subdev *sd);
	int (*reset)(struct v4l2_subdev *sd, u32 val);
	int (*s_gpio)(struct v4l2_subdev *sd, u32 val);
	long (*command)(struct v4l2_subdev *sd, unsigned int cmd, void *arg);
	long (*ioctl)(struct v4l2_subdev *sd, unsigned int cmd, void *arg);
#ifdef CONFIG_COMPAT
	long (*compat_ioctl32)(struct v4l2_subdev *sd, unsigned int cmd,
			       unsigned long arg);
#endif
#ifdef CONFIG_VIDEO_ADV_DEBUG
	int (*g_register)(struct v4l2_subdev *sd, struct v4l2_dbg_register *reg);
	int (*s_register)(struct v4l2_subdev *sd, const struct v4l2_dbg_register *reg);
#endif
	int (*s_power)(struct v4l2_subdev *sd, int on);
	int (*interrupt_service_routine)(struct v4l2_subdev *sd,
						u32 status, bool *handled);
	int (*subscribe_event)(struct v4l2_subdev *sd, struct v4l2_fh *fh,
			       struct v4l2_event_subscription *sub);
	int (*unsubscribe_event)(struct v4l2_subdev *sd, struct v4l2_fh *fh,
				 struct v4l2_event_subscription *sub);
};
```
【关键成员】：

| 成员函数             | 函数原型                                                                       | 功能描述                     | 在 RK Sensor 驱动中的典型实现                                          |
| ---------------- | -------------------------------------------------------------------------- | ------------------------ | ------------------------------------------------------------- |
| `ioctl`          | `long (\*)(struct v4l2_subdev \*sd, unsigned int cmd, void \*arg);`        | 处理私有自定义命令<br>（非标准V4L2命令） | 实现特有功能，响应系统 ioctl 调用                                          |
| `compat_ioctl32` | `long (\*)(struct v4l2_subdev \*sd, unsigned int cmd, unsigned long arg);` |                          | 当 32bit 应用使用 64 位 kernel 时，实现私有功能<br>为了解决从用户空间传递过来/传递到用户空间的数据 |
| `s_power`        | `int (\*)(struct v4l2_subdev \*sd, int on);`                               | 控制电源状态                   | 控制 PWDN / RESET 引脚的电平序列，管理 PMIC 供电时序<br>（上电：on=1，掉电：on=0）     |
【示例】：
```c
static const struct v4l2_subdev_core_ops sc200ai_core_ops = {
	.s_power = sc200ai_s_power,
	.ioctl = sc200ai_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl32 = sc200ai_compat_ioctl32,
#endif
};
```
目前使用了如下的私有ioctl命令，用来实现**模组信息的查询**和**OTP信息的查询设置**。

| 私有ioctl                        | 描述                                                                                                                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| `RKMODULE_GET_MODULE_INFO`     | 获取模组信息                                                                                                                         |
| `PREISP_CMD_SET_HDRAE_EXP`     | hdr 曝光设置                                                                                                                       |
| `RKMODULE_SET_HDR_CFG`         | 设置 hdr 模式，可实现 normal（线性）和 hdr（高动态）模式切换，<br>需要驱动适配 normal 和 hdr2 组配置信息                                                          |
| `RKMODULE_GET_HDR_CFG`         | 获取当前hdr模式                                                                                                                      |
| `RKMODULE_SET_CONVERSION_GAIN` | 设置线性模式的 `conversion gain`，如 `imx347`、`os04a10` Sensor 带  <br>有 `conversion gain` 的功能，若当前 Sensor 不支持 conversion  <br>gain，可不实现。 |

#### 2.2.1.4. struct v4l2_subdev_video_ops

【说明】：视频设备控制接口。

【定义】：`kernel-6.1/include/media/v4l2-subdev.h`
```c
struct v4l2_subdev_video_ops {
	int (*s_routing)(struct v4l2_subdev *sd, u32 input, u32 output, u32 config);
	int (*s_crystal_freq)(struct v4l2_subdev *sd, u32 freq, u32 flags);
	int (*g_std)(struct v4l2_subdev *sd, v4l2_std_id *norm);
	int (*s_std)(struct v4l2_subdev *sd, v4l2_std_id norm);
	int (*s_std_output)(struct v4l2_subdev *sd, v4l2_std_id std);
	int (*g_std_output)(struct v4l2_subdev *sd, v4l2_std_id *std);
	int (*querystd)(struct v4l2_subdev *sd, v4l2_std_id *std);
	int (*g_tvnorms)(struct v4l2_subdev *sd, v4l2_std_id *std);
	int (*g_tvnorms_output)(struct v4l2_subdev *sd, v4l2_std_id *std);
	int (*g_input_status)(struct v4l2_subdev *sd, u32 *status);
	int (*s_stream)(struct v4l2_subdev *sd, int enable);
	int (*g_pixelaspect)(struct v4l2_subdev *sd, struct v4l2_fract *aspect);
	int (*g_frame_interval)(struct v4l2_subdev *sd,
				struct v4l2_subdev_frame_interval *interval);
	int (*s_frame_interval)(struct v4l2_subdev *sd,
				struct v4l2_subdev_frame_interval *interval);
	int (*s_dv_timings)(struct v4l2_subdev *sd,
			struct v4l2_dv_timings *timings);
	int (*g_dv_timings)(struct v4l2_subdev *sd,
			struct v4l2_dv_timings *timings);
	int (*query_dv_timings)(struct v4l2_subdev *sd,
			struct v4l2_dv_timings *timings);
	int (*s_rx_buffer)(struct v4l2_subdev *sd, void *buf,
			   unsigned int *size);
	int (*pre_streamon)(struct v4l2_subdev *sd, u32 flags);
	int (*post_streamoff)(struct v4l2_subdev *sd);
};
```
【关键成员】：

| 成员名称               | 描述                                                  |
| ------------------ | --------------------------------------------------- |
| `s_stream`         | 通知驱动视频流启动或停止的关键接口；                                  |
| `g_frame_interval` | （获取帧率）`VIDIOC_SUBDEV_G_FRAME_INTERVAL` ioctl 的回调函数； |
| `s_frame_interval` | （设置帧率）动态修改 Sensor 的输出帧率（帧间隔），是调整帧率的核心回调函数；          |
【示例】：
```c
static const struct v4l2_subdev_video_ops sc200ai_video_ops = {
	.s_stream = sc200ai_s_stream,
	.g_frame_interval = sc200ai_g_frame_interval,  
	.s_frame_interval = sc200ai_s_frame_interval,
};
```
#### 2.2.1.5. struct v4l2_subdev_pad_ops

【说明】：此结构体定义子设备在 **像素格式协商、裁剪、缩放和帧布局** 等操作的回调函数集合；在 RK 等平台的 Sensor 驱动中，他是配置图像处理流水（Sensor -> ISP -> 内存）的关键接口；

【定义】：`kernel-6.1/include/media/v4l2-subdev.h`
```c
struct v4l2_subdev_pad_ops {
	int (*init_cfg)(struct v4l2_subdev *sd,
			struct v4l2_subdev_state *state);  // 将焊盘配置初始化为默认值
	int (*enum_mbus_code)(struct v4l2_subdev *sd,
			      struct v4l2_subdev_state *state,
			      struct v4l2_subdev_mbus_code_enum *code);  // 枚举媒体总线格式代码
	int (*enum_frame_size)(struct v4l2_subdev *sd,
			       struct v4l2_subdev_state *state,
			       struct v4l2_subdev_frame_size_enum *fse);  // 枚举帧尺寸
	int (*enum_frame_interval)(struct v4l2_subdev *sd,
				   struct v4l2_subdev_state *state,
				   struct v4l2_subdev_frame_interval_enum *fie);  // 枚举帧间隔
	int (*get_fmt)(struct v4l2_subdev *sd,
		       struct v4l2_subdev_state *state,
		       struct v4l2_subdev_format *format);  // 获取格式
	int (*set_fmt)(struct v4l2_subdev *sd,
		       struct v4l2_subdev_state *state,
		       struct v4l2_subdev_format *format);  // 设置格式
	int (*get_selection)(struct v4l2_subdev *sd,
			     struct v4l2_subdev_state *state,
			     struct v4l2_subdev_selection *sel);  // 获取选择区域
	int (*set_selection)(struct v4l2_subdev *sd,
			     struct v4l2_subdev_state *state,
			     struct v4l2_subdev_selection *sel);  // 设置选择区域
	int (*get_edid)(struct v4l2_subdev *sd, struct v4l2_edid *edid);  // 获取 EDID 信息
	int (*set_edid)(struct v4l2_subdev *sd, struct v4l2_edid *edid);  // 设置 EDID 信息
	int (*dv_timings_cap)(struct v4l2_subdev *sd,
			      struct v4l2_dv_timings_cap *cap);  // 获取数字视频时序能力
	int (*enum_dv_timings)(struct v4l2_subdev *sd,
			       struct v4l2_enum_dv_timings *timings);  // 枚举数字视频时序
#ifdef CONFIG_MEDIA_CONTROLLER
	int (*link_validate)(struct v4l2_subdev *sd, struct media_link *link,
			     struct v4l2_subdev_format *source_fmt,
			     struct v4l2_subdev_format *sink_fmt);  // 检查管道链路是否可用于流传输
#endif /* CONFIG_MEDIA_CONTROLLER */
	int (*get_frame_desc)(struct v4l2_subdev *sd, unsigned int pad,
			      struct v4l2_mbus_frame_desc *fd);  // 获取底层媒体总线帧参数
	int (*set_frame_desc)(struct v4l2_subdev *sd, unsigned int pad,
			      struct v4l2_mbus_frame_desc *fd);  // 设置底层媒体总线帧参数
	int (*get_mbus_config)(struct v4l2_subdev *sd, unsigned int pad,
			       struct v4l2_mbus_config *config);  // 获取远端子设备的媒体总线配置
};
```

【关键成员】：

| 成员函数                  | 功能描述                      | Rockchip 平台典型应用                                                                    |
| --------------------- | ------------------------- | ---------------------------------------------------------------------------------- |
| `enum_mbus_code`      | 枚举支持的媒体总线格式（像素格式）         | 报告 Sensor 支持的 RAW / YUV 格式（如 MEDIA_BUS_FMT_SBGGR10_1X10）                           |
| `get_fmt`             | 获取当前设置的格式（分辨率 + 像素格式）     | 查询 Sensor 当前输出配置（如1920x1080@SRGGB10）；                                              |
| `set_fmt`             | 设置格式（分辨率 + 像素格式）重要函数      | 配置 Sensor 输出分辨率（如 1920 x 1080）和格式（如 NV12），影响MIPI数据流                                |
| `enum_frame_size`     | 枚举指定像素格式下支持的分辨率           | 列出特定格式（如 UYVY）可用的分辨率（如 640x480，1280x720）                                           |
| `enum_frame_interval` | 枚举指定格式和分辨率下支持的帧率          | 报告当前模式下支持的分辨率、帧率等；                                                                 |
| `get_selection`       | 获取裁剪 / 缩放区域（读写区域，输出尺寸）    | 实现是吗变焦，或者设置低功耗模式下的小分辨率区域；                                                          |
| `set_selection`       | 设置裁剪 / 缩放区域               | 比如 在预览模式中使用 1080p 区域；                                                              |
| `get_edid /set_edid`  | 处理 EDID（显示器扩展表示数据）        | HDMI 接收芯片专用，Sensor 驱动通常忽略；                                                         |
| `dv_timings_ca`       | 获取支持的数字视频时序标准             | HDMITX/RX 芯片专用；                                                                    |
| `enum_dv_timings`     | 枚举支持的数字视频时序               | HDMITX/RX 芯片专用；                                                                    |
| `get_mbus_config`     | 获取媒体总线配置（如 MIPI D-PHY 参数） | 报告：<br>- 通道数：如 `V4L2_MBUS_CSI2_4LAN`；<br>- 时钟模式：`V4L2_MBUS_CSI2_CONTINUOUS_CLOCK`； |
| `set_mbus_config`     | 设置媒体总线配置                  | 动态调整 MIPI 参数（如切换 2-lane/4-lane 模式）                                                 |
【示例】：
```c
static const struct v4l2_subdev_pad_ops sc200ai_pad_ops = {
	.enum_mbus_code = sc200ai_enum_mbus_code,
	.enum_frame_size = sc200ai_enum_frame_sizes,
	.enum_frame_interval = sc200ai_enum_frame_interval,
	.get_fmt = sc200ai_get_fmt,
	.set_fmt = sc200ai_set_fmt,
	.get_mbus_config = sc200ai_g_mbus_config,
};
```

#### 2.2.1.6. struct v4l2_ctrl_ops

【说明】：`struct v4l2_ctrl_ops` 是 V4L2 控制框架的核心操作集，用于实现传感器参数（如曝光、增益、白平衡 等）的动态控制。在 Rockchip 平台 Sensor 驱动开发中，它提供了标准化接口来处理应用层对传感器参数的配置请求，是连接用户空间控制与硬件寄存器操作的关键枢纽；

【定义】：`kernel-6.1/include/media/v4l2-ctrls.h`
```c
struct v4l2_ctrl_ops {
	int (*g_volatile_ctrl)(struct v4l2_ctrl *ctrl);
	int (*try_ctrl)(struct v4l2_ctrl *ctrl);
	int (*s_ctrl)(struct v4l2_ctrl *ctrl);
};
```
【关键成员】：

| 成员函数     | 功能描述          | Rockchip 典型实现案例                |
| -------- | ------------- | ------------------------------ |
| `s_ctrl` | 设置控制值（最重要的函数） | 将用户空间设置的值写入硬件寄存器（如设置曝光时间 / 增益） |
【示例】：
```c
static int sc200ai_set_ctrl(struct v4l2_ctrl *ctrl)
{
	struct sc200ai *sc200ai = container_of(ctrl->handler,
					       struct sc200ai, ctrl_handler);
	struct i2c_client *client = sc200ai->client;
	
	if (!pm_runtime_get_if_in_use(&client->dev))
		return 0;

	switch (ctrl->id) {
	case V4L2_CID_EXPOSURE:
		dev_dbg(&client->dev, "set exposure value 0x%x\n", ctrl->val);
		...
		break;
	case V4L2_CID_ANALOGUE_GAIN:
		dev_dbg(&client->dev, "set gain value 0x%x\n", ctrl->val);
		...
		break;
	case V4L2_CID_VBLANK:
		dev_dbg(&client->dev, "set blank value 0x%x\n", ctrl->val);
		...
		break;
	case V4L2_CID_TEST_PATTERN:
		...
		break;
	case V4L2_CID_HFLIP:
		...
		break;
	case V4L2_CID_VFLIP:
		...
		break;
	default:
		dev_warn(&client->dev, "%s Unhandled id:0x%x, val:0x%x\n",
			 __func__, ctrl->id, ctrl->val);
		break;
	}

	pm_runtime_put(&client->dev);

	return ret;
}

static const struct v4l2_ctrl_ops sc200ai_ctrl_ops = {
	.s_ctrl = sc200ai_set_ctrl,
};
```

### 2.2.2. API 介绍

#### 2.2.2.1. xxxx_set_fmt

【描述】：设置 Sensor 输出格式。这⾥指的是⼀个 Sensor 驱动内⽀持多种分辨率。通过 `set_fmt` 下发分辨率，Sensor 驱动内 遍历去获得最符合的分辨率。 

HDR 和线性模式的切换不通过这个函数配置，通过 ioctl 实现私有命令，实现切换。需要注意的是 HDR 和 线性模式的分辨率要⼀致才能实现切换。

【语法】：
```c
static int xxxx_set_fmt(struct v4l2_subdev *sd,
						struct v4l2_subdev_pad_config *cfg,
						struct v4l2_subdev_format *fmt)
```
【参数】：

| 参数名称 | 描述                               | 输入输出 |
| ---- | -------------------------------- | ---- |
| sd   | v4l2 subdev 结构体指针                | 输入   |
| cfg  | subdev pad information 结构体指针     | 输入   |
| fmt  | Pad-level media bus format 结构体指针 | 输入   |
【返回值】：

0：成功、非0：失败

【注意事项】

kernel 6.1版本中 `struct v4l2_subdev_pad_config *cfg` 改为了 `struct v4l2_subdev_state *sd_state`

#### 2.2.2.2. xxxx_get_fmt

【描述】：

获取 Sensor 输出格式，获取到的是当前使⽤的配置，如前⾯描述的 struct xxx_mode ⾥⾯可能配置多个 不同分辨率的配置，驱动初始化的时候会选中其中⼀个，或者 set_fmt 切换后，可通过 get_fmt 来确认当 前的 format。 

对于 RK3588 以前的芯⽚，这边还会通过 `fmt->reserved[0]` 这个保留参数上传当前设备使⽤的 vc 通道信 息，也就是可以通过这个参数来指定设备的vc通道，否则按默认值。 

对于RK3588及后⾯的芯⽚，则通过 `ioctl` 实现 `RKMODULE_GET_CHANNEL_INFO`，来获取通道信息。

【语法】：
```c
static int xxxx_get_fmt(struct v4l2_subdev *sd,
						 struct v4l2_subdev_pad_config *cfg,
						 struct v4l2_subdev_format *fmt)
```

【参数】：

| 参数名称  | 描述                                 | 输入输出 |
| ----- | ---------------------------------- | ---- |
| `sd`  | `v4l2 subdev` 结构体指针                | 输入   |
| `cfg` | `subdev pad information` 结构体指针     | 输入   |
| `fmt` | `Pad-level media bus format` 结构体指针 | 输出   |
【返回值】：

0：成功、非0：失败

【注意事项】：

kernel 6.1版本中 `struct v4l2_subdev_pad_config *cfg` 改为了 `struct v4l2_subdev_state *sd_state`

#### 2.2.2.3. xxxx_enum_mbus_code

【描述】：

枚举 Sensor 输出 `bus format`，驱动中会根据 `struct xxx_mode` 结构定义⼀个静态结构体变量，⾥⾯填充 Sensor 驱动⽀持的各个分辨率，这个函数可以根据结构体变量⾥⾯填充的 `bus format`，返回枚举值。

【语法】：

```c
static int xxxx_enum_m bus_code(struct v4l2_subdev *sd,  
								struct v4l2_subdev_pad_config *cfg,  
								struct v4l2_subdev_mbus_code_enum *code)
```
【参数】：

| 参数名称   | 描述                                   | 输入输出 |
| ------ | ------------------------------------ | ---- |
| `sd`   | `v4l2 subdev` 结构体指针                  | 输入   |
| `cfg`  | `subdev pad information` 结构体指针       | 输入   |
| `code` | `media bus format enumeration` 结构体指针 | 输出   |
【返回值】：

0：成功、非0：失败

【注意事项】：

kernel 6.1版本中 `struct v4l2_subdev_pad_config *cfg` 改为了 `struct v4l2_subdev_state *sd_state`
#### 2.2.2.4. xxxx_enum_frame_sizes

【描述】：

枚举 Sensor 输出大小。驱动中会根据 `struct xxx_mode` 结构定义一个静态结构体变量，里面填充 Sensor  驱动支持的各个分辨率，这个函数可以根据结构体变量里面**填充的分辨率，返回枚举值**。

【语法】：

```c
static int xxxx_enum_frame_sizes(struct v4l2_subdev *sd,  
								struct v4l2_subdev_pad_config *cfg,  
								struct v4l2_subdev_frame_size_enum *fse)
```
【参数】：

| 参数名称  | 描述                             | 输入输出 |
| ----- | ------------------------------ | ---- |
| `sd`  | `v4l2 subdev` 结构体指针            | 输入   |
| `cfg` | `subdev pad information` 结构体指针 | 输入   |
| `fse` | `media bus frame size` 结构体指针   | 输出   |
【返回值】：

0：成功、非0：失败

【注意事项】：

kernel 6.1版本中 `struct v4l2_subdev_pad_config *cfg` 改为了 `struct v4l2_subdev_state *sd_state`
#### 2.2.2.5. xxxx_g_frame_interval

【描述】：

获取 Sensor 输出帧间隔。驱动中会根据 `struct xxx_mode` 结构定义—个静态结构体变量 ，里面的配置的 帧率是最高帧率 ，驱动通过**增加 vblank 的值来降低帧率** ，如果需要获取实际帧率 ，这边返回给应用的值 ，需要将**默认的 vblank** 和**当前的 vblank** 值进行换算 ，以获取最新的帧间隔 ，上传给应用。

【语法】：

```c
static int xxxx_g_frame_interval(struct v4l2_subdev *sd,  
								struct v4l2_subdev_frame_interval *fi)
```
【参数】：

| 参数名称 | 描述                           | 输入输出 |
| ---- | ---------------------------- | ---- |
| `sd` | `v4l2 subdev` 结构体指针          | 输入   |
| `fi` | `pad-level frame rate` 结构体指针 | 输出   |
【返回值】：

0：成功、非0：失败

#### 2.2.2.6. xxxx_s_stream

【描述】：

设置 `stream` 输入输出。  

函数—般根据**传入的参数 on 实现 start_stream/stop_stream 函数**。`start_stream` 里面实现初始化寄存器数组的配置，初始化曝光参数配置 ，开启数据流。 `stop_stream` 里面实现 `stream off` 的寄存器配置。

【语法】：

```c
static int xxxx_s_stream(struct v4l2_subdev *sd, int on)  
```
【参数】：

| 参数名称 | 描述                           | 输入输出 |
| ---- | ---------------------------- | ---- |
| `sd` | `v4l2 subdev` 结构体指针          | 输入   |
| `on` | 1: 启动stream输出; 0: 停止stream输出 | 输入   |
【返回值】：

0：成功、非0：失败

#### 2.2.2.7. xxxx_runtime_resume

【描述】：

Sensor上电时的回调函数。  

函数里面主要做 `Sensor` 上电操作 ，将上电操作放在这个函数 ，方便 Sensor 驱动其他地方或者上层驱动调用 `pm_runtime_get_sync` 来上电。

【语法】：

```c
static int xxxx_runtime_resume(struct device *dev)
```
【参数】：

| 参数名称  | 描述             | 输入输出 |
| ----- | -------------- | ---- |
| `dev` | `device` 结构体指针 | 输入   |
【返回值】：

0：成功、非0：失败

#### 2.2.2.8. xxxx_runtime_suspend

【描述】：

`Sensor` 下电时的回调函数。

函数里面做 `Sensor` 的下电操作 ，方便 `Sensor` 驱动其他地方或者上层驱动调用 `pm_runtime_put` 来下电。

【语法】：

```c
static int xxxx_runtime_suspend(struct device *dev)
```
【参数】：

| 参数名称  | 描述             | 输入输出 |
| ----- | -------------- | ---- |
| `dev` | `device` 结构体指针 | 输入   |
【返回值】：

0：成功、非0：失败

#### 2.2.2.9. xxxx_set_ctrl

【描述】：

设置各个 `v4l2 control` 的值。

注册 `v4l2 control` 命令时 ，调用这个回调函数 ，应用设置 `control` 参数下来时 ，通过这个回调函数来配置 `Sensor`。  

需要实现的 `v4l2 control` 命令可以参考 3.2.1.6 参考[[-1. 捕获/RK 平台 MIPI-CSI 驱动#2 2 1 2 struct v4l2_subdev_ops]] 说明实现。

【语法】：

```c
static int xxxx_set_ctrl(struct v4l2_ctrl *ctrl)
```
【参数】：

| 参数名称   | 描述                | 输入输出 |
| ------ | ----------------- | ---- |
| `ctrl` | `v4l2_ctrl` 结构体指针 | 输入   |
【返回值】：

0：成功、非0：失败

#### 2.2.2.10. xxxx_enum_frame_interval

【描述】：

枚举 `Sensor` 支持的帧间隔参数。  

驱动中会根据 `struct xxx_mode` 结构定义—个静态结构体变量，可以将里面支持的几种分辨率下的帧间 隔枚举上去。这边—般用途不大，对应帧率调整还是通过调整 vblank。

【语法】：

```c
static int xxxx_enum_frame_interval(struct v4l2_subdev *sd,  
									struct v4l2_subdev_pad_config *cfg,  
									struct v4l2_subdev_frame_interval_enum *fie)
```
【参数】：

| 参数名称  | 描述                  | 输入输出 |
| ----- | ------------------- | ---- |
| `sd`  | `v4l2 subdev` 结构体指针 | 输入   |
| `cfg` | pad配置参数             | 输入   |
| `fie` | 帧间隔参数               | 输出   |
【返回值】：

0：成功、非0：失败

【注意事项】：

kernel 6.1版本中 `struct v4l2_subdev_pad_config *cfg` 改为了 `struct v4l2_subdev_state *sd_state`
#### 2.2.2.11. xxxx_g_mbus_config

【描述】：

获取支持的总线配置 ，总线类型存在 **并口/MIPI/LVDS** 等几种类型 ，并口里面存在 **BT601/BT656/BT1120** ，MIPI 存在 **DPHY/CPHY** 协议 ，控制器通过这个接口获取当前 Sensor 使用的总线参 数 ，来确认控制器的采集方式。  

比如使用 MIPI 时 ， 当 Sensor 支持多种 MIPI 传输模式时 ，可以根据 Sensor 当前使用的 MIPI 模式上传参数。

【语法】：

```c
static int xxxx_g_m bus_config(struct v4l2_subdev *sd,  
								struct v4l2_m bus_config *config)
```
【参数】：

| 参数名称     | 描述                  | 输入输出 |
| -------- | ------------------- | ---- |
| `sd`     | `v4l2 subdev` 结构体指针 | 输入   |
| `config` | 总线配置参数              | 输出   |
【返回值】：

0：成功、非0：失败

#### 2.2.2.12. xxxx_get_selection

【描述】：

配置裁剪参数 ，**ISP 输入的宽度要求16对⻬ ，高度8对⻬** ，对于 Sensor 输出的分辨率不符合对⻬或 Sensor 输出分辨率不是标准分辨率 ，可实现这个函数对输入 ISP 的分辨率做裁剪。

【语法】：

```c
static int xxxx_get_selection(struct v4l2_subdev *sd,  
								struct v4l2_subdev_pad_config *cfg,  
								struct v4l2_subdev_selection *sel)
```
【参数】：

| 参数名称  | 描述                  | 输入输出 |
| ----- | ------------------- | ---- |
| `sd`  | `v4l2 subdev` 结构体指针 | 输入   |
| `cfg` | pad配置参数             | 输入   |
| `sel` | 裁剪参数                | 输出   |
【返回值】：

0：成功、非0：失败

【注意事项】：

kernel 6.1版本中 `struct v4l2_subdev_pad_config *cfg` 改为了 `struct v4l2_subdev_state *sd_state`

#### 2.2.2.13. xxxx_ioctl

【描述】：

`ioctl` 私有命令实现。

【语法】：

```c
static long xxxx_ioctl(struct v4l2_subdev *sd, unsigned int cmd, void *arg)
```
【参数】：

| 参数名称  | 描述                  | 输入输出  |
| ----- | ------------------- | ----- |
| `sd`  | `v4l2 subdev` 结构体指针 | 输入    |
| `cmd` | 私有命名值               | 输入    |
| `arg` | 传递的参数               | 输入/输出 |
【返回值】：

0：成功、非0：失败

## 2.3. 驱动移植步骤

### 2.3.1. 实现标准I2C子设备驱动部分

### 2.3.2. 实现 v4l2 子设备驱动

主要实现以下三个成员：
```c
struct v4l2_subdev_core_ops
struct v4l2_subdev_video_ops
struct v4l2_subdev_pad_ops
```

1. 参考 `struct v4l2_subdev_core_ops  ` 说明实现其回调函数，主要实现以下回调：
- `.s_power`：`s_power` 实现 `Sensor` 的上下电操作 ，对于一些寄存器数组较⻓的 Sensor ，可以归纳出公共部分的寄存器，在上电后执行 ，s_stream 只配置不同分辨率配置必须的寄存器 ，加快切换速度。
- `.ioctl`：ioctl 主要实现私有控制命令；例如以下：

| 成员名称                           | 描述                                                                                                                                                                  |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `RKMODULE_GET_MODULE_INFO`     | DTS 文件定义的模组信息(模组名称等) ，通过该命令上 传给 `camera_engine` ，用于设备匹配                                                                                                             |
| `RKMODULE_AWB_CFG`             | 模组 `OTP` 信息使能情况下 ，`camera_engine` 通过该命 令传递典型模组 `AWB` 标定值 ，`CIS` 驱动负责与当前模组 `AWB` 标定值比较后 ，生成 `R/B Gain` 值设置到 `CIS MWB` 模块中；                                           |
| `RKMODULE_LSC_CFG`             | 模组 `OTP` 信息使能情况下 ，`camera_engine` 通过该命 令控制 `LSC` 标定值生效使能；                                                                                                           |
| `PREISP_CMD_SET_HDRAE_EXP`     | HDR 曝光设置                                                                                                                                                            |
| `RKMODULE_SET_HDR_CFG`         | 设置 HDR 模式 ，可**实现 normal 和 hdr 切换** ，需要驱动 适配  `HDR` 和 `normal` 2组配置信息                                                                                                |
| `RKMODULE_GET_HDR_CFG`         | 获取当前 `HDR` 模式；                                                                                                                                                      |
| `RKMODULE_SET_CONVERSION_GAIN` | 设置线性模式的 `conversion gain` ，如 `imx347`、 `os04a10` Sensor 带有 `conversion gain` 的功能 ，高转换 的 `conversion gain` 可以在低照度下获得更好的信噪 比 ，如 `Sensor` 不支持 `conversion gain` ，可不实现； |
| `RKMODULE_SET_QUICK_STREAM`    | 配置 `Sensor` 的 `stream on/off` 寄存器 ，用于异常复位;                                                                                                                          |
| `RKMODULE_GET_CHANNEL_INFO`    | 获取通道信息 ，默认 `vicap` 的 `id0~id3` 对应 `vc0~vc3` ，如果有特殊需求可通过这个命令配置通道信 息 ，如 `SPD/EBD` 数据采集。RK3588 以前芯片仍通过 `get_fmt` 获取通道信息。                                               |

2. 参考 `struct v4l2_subdev_video_ops` 说明实现其回调函数。

主要实现以下回调：
- `.enum_m bus_code`：枚举当前 CIS 驱动支持数据格式；
- `.enum_frame_size`：枚举当前 CIS 驱动支持分辨率；
- `.get_fmt`：RKISP driver 通过该回调获取CIS输出的数据格式 ，务必实现；
- `.set_fmt`：设置CIS驱动输出数据格式以及分辨率 ，务必实现；
- `.enum_frame_interval`：枚举sensor支持的帧间隔 ，包含分辨率；
- `.get_selection`：配置裁剪参数 ，ISP 输入的宽度要求16对⻬ ，高度8对⻬。

3. 参考 `struct v4l2_subdev_pad_ops` 说明实现其回调函数。

主要实现以下回调：
- `.s_ctrl`：RKISP driver、camera_engine 通过设置不同的命令来实现 CIS 曝光控制；
