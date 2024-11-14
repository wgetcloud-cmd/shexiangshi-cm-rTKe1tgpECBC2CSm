


---


　　大家好，我是痞子衡，是正经搞技术的痞子。今天痞子衡给大家介绍的是**在FDCB里配置串行NOR Flash多个寄存器的注意事项**。


　　关于使用 i.MXRT 启动头 FDCB 来设置 Flash 内部寄存器，痞子衡写过如下两篇文章，在进入本文之前，建议大家先阅读下这两篇文章，有个初步了解。



> * [《在FDCB里设置Flash的Dummy Cycle》](https://github.com)
> * [《在FDCB里切换Flash模式至QPI/OPI》](https://github.com)


　　我们知道 Flash 内部常常有多个状态/配置寄存器，这些寄存器有些是易失性的，有些是非易失性的。当芯片被指定从 Flash 启动的时候，我们如果希望 BootROM 能够根据不同应用需求来提前设置好这些 Flash 寄存器，那么在应用程序里就不用再额外配置了（涉及 Flash 工作状态变化的配置，如果是 XIP 程序去操作，需要考虑代码重定向问题）。


　　对于使用 FDCB 来配置 Flash 一个寄存器的操作，相信大家都很了解，在恩智浦 SDK 包里默认 FDCB 启动头里都有成功示例。最近痞子衡同事尝试使用 FDCB 去配置镁光 MT35X 的两个寄存器（地址为 0x000000 的寄存器切至 OPI DDR、地址为 0x000003 的寄存器设 Drive Strength）发现有一个寄存器设置没生效，这是怎么回事？今天我们来聊一聊：



> * Note: 本文适用于 i.MXRT500/600/1010/1020/1040/1050/1060/1160/1170


### 一、FDCB提供的Flash寄存器配置能力


　　我们先来看一下 FDCB 结构里跟 Flash 配置相关的成员，痞子衡整理如下，简单来说，就是有一条 deviceModeSeq 和三条 configCmdSeqs，所以最多能配置 Flash 里 4 个不同命令下对应的寄存器（有些 Flash 里一条配置命令能连续写入多个寄存器，这种情况下就能配置不止 4 个寄存器），这对于大部分应用场景都完全够用了。



> * Note 1: BootROM 执行这四个配置的顺序分别是 deviceModeSeq、configCmdSeqs\[0]、configCmdSeqs\[1]、configCmdSeqs\[2]，记住这个顺序。
> * Note 2: deviceModeSeq 与 configCmdSeq 实现的配置功能几乎没有区别，两者能做的事情是一样的，可以互换。



```
//!@brief FlexSPI Memory Configuration Block
typedef struct _FlexSPIConfig
{
    // ...

    //!< [0x010-0x010] Device Mode Configure enable flag, 1 - Enable, 0 - Disable
    uint8_t deviceModeCfgEnable;
    //!< [0x011-0x011] Specify the configuration command type:Quad Enable, DPI/QPI/OPI switch, Generic configuration, etc.
    uint8_t deviceModeType; 
    //!< [0x012-0x013] Wait time for all configuration commands, unit: 100us
    uint16_t waitTimeCfgCommands;
    //!< [0x014-0x017] Device mode sequence info, [7:0] - LUT sequence id, [15:8] - LUt sequence number, [31:16] Reserved
    flexspi_lut_seq_t deviceModeSeq;
    //!< [0x018-0x01b] Argument/Parameter for device configuration
    uint32_t deviceModeArg;

    //!< [0x01c-0x01c] Configure command Enable Flag, 1 - Enable, 0 - Disable 
    uint8_t configCmdEnable;
    //!< [0x01d-0x01f] Configure Mode Type, similar as deviceModeTpe 
    uint8_t configModeType[3];
    //!< [0x020-0x02b] Sequence info for Device Configuration command, similar as deviceModeSeq
    flexspi_lut_seq_t configCmdSeqs[3];
    //!< [0x030-0x03b] Arguments/Parameters for device Configuration commands
    uint32_t configCmdArgs[3];

    // ...

    //!< [0x07c-0x07d] Busy offset, valid value: 0-31
    uint16_t busyOffset;
    //!< [0x07e-0x07f] Busy flag polarity, 0 - busy flag is 1 when flash device is busy, 1 - busy flag is 0 when flash device is busy
    uint16_t busyBitPolarity;
    //!< [0x080-0x17f] Lookup table holds Flash command sequences
    uint32_t lookupTable[64];

    // ...

} flexspi_mem_config_t;

```

　　在 [《在FDCB里设置Flash的Dummy Cycle》](https://github.com) 一文最后，痞子衡已经分享了 BootROM 解析执行 configCmdSeq 的代码流程，在这个流程里我们能看到和 deviceModeType/configModeType\[]、waitTimeCfgCommands 成员相关的逻辑代码，这里有必要进一步解释一下。



```
//!@brief Flash Configuration Command Type
enum
{
    kDeviceConfigCmdType_Generic,    //!< Generic command, for example: configure dummy cycles, drive strength, etc
    kDeviceConfigCmdType_QuadEnable, //!< Quad Enable command
    kDeviceConfigCmdType_Spi2Xpi,    //!< Switch from SPI to DPI/QPI/OPI mode
    kDeviceConfigCmdType_Xpi2Spi,    //!< Switch from DPI/QPI/OPI to SPI mode
    kDeviceConfigCmdType_Spi2NoCmd,  //!< Switch to 0-4-4/0-8-8 mode
    kDeviceConfigCmdType_Reset,      //!< Reset device command
};

```

　　当 deviceModeType/configModeType 成员被设置为 kDeviceConfigCmdType\_Spi2Xpi 时；如果 FDCB 本身被获取时 BootROM 用得是 DPI/QPI/OPI 命令（根据 efuse 配置决定），那么此条 Flash 寄存器配置会被直接忽略；如果 BootROM 用得是普通一线 SPI 模式读取的 FDCB，那么这个 Flash 寄存器配置仍然生效。


　　kDeviceConfigCmdType\_Spi2Xpi 等三个跟 Flash 命令模式切换相关的配置类型，顾名思义就是告诉 BootROM 这三种配置会导致 Flash 工作模式变化，而一旦 Flash 工作模式发生变化，用于判断配置是否完成的 READ\_STATUS 命令也随之变得不可用（因为 SPI 模式下与 DPI/QPI/OPI 模式下的命令序列不同），这种情况下就需要借助 waitTimeCfgCommands 成员来实现软件延时以等待对 Flash 的配置真正生效（如果不等生效就直接进入后续流程，可能会导致启动问题）。


　　kDeviceConfigCmdType\_Generic 配置类型，则是用于跟工作模式切换无关的 Flash 寄存器配置，这种情况下 BootROM 可以使用 READ\_STATUS 命令来判断对 Flash 寄存器配置是否已经生效，那么就不需要 waitTimeCfgCommands 实现的延时等待（需要查 Flash 数据手册作相应设置，比较麻烦，而且手册里是给了典型值和最大值，取最大值会导致启动时间变长，典型值不能保证适用所有情况）。


### 二、配置Flash多个寄存器注意点


　　Flash 配置寄存器的写入流程通常分三步：一、WRITE\_ENABLE 使能写操作；二、具体的 CONFIG\_REG 操作；三、READ\_STATUS 或者软件延时确保配置已完成。这些命令序列全部存储在 FDCB 里的 lookupTable\[64] 成员里。


　　关于 Flash 寄存器的配置操作，从寄存器属性上来看，分为易失性和非易失性两种，前者的操作一般是立即生效的，后者的操作不是立即生效（Flash 状态寄存器 WIP 位会反映进度）。从命令模式角度来看，分为非模式切换操作（比如设置 Dummy Cycle、Drive Strength）以及切换 SPI 与 DPI/QPI/OPI 模式操作两种，同样前者操作是立即生效，后者操作不是立即生效。


　　现在回到文章开头痞子衡同事遇到的问题，如果 deviceModeSeq 用于切换至 OPI DDR 模式，configCmdSeqs\[0] 用于设 Drive Strength，请问哪一个操作没有生效？这里就不卖关子了，设 Drive Strength 没有生效，因为第一个配置是切换至 OPI DDR 模式，当 Flash 切换到该模式时，用于第二/三/四个配置的 WRITE\_ENABLE 命令变得不可用了（还是因为 SPI 模式下与 DPI/QPI/OPI 模式下的命令序列不同），当然对应 Flash 寄存器设置就无效了。


　　那么如何避免这个问题？有一个一劳永逸的方法，那就是永远用 configCmdSeqs\[2] 去做 SPI 与 DPI/QPI/OPI 模式切换操作，这样就不会影响前面三个配置。前面讲了，此时判断命令模式是否切换完成不能用 READ\_STATUS 命令，那么就一定需要 waitTimeCfgCommands 来做软延时，如果延时时间不够，并且后续 BootROM 执行到验证 IVT 头时模式切换仍未完成，这会导致 IVT 启动头无法正确获取从而导致启动失败。


　　假设 waitTimeCfgCommands 延时设置对于命令模式切换已足够，这样问题就一定解决了吗？其实还不一定，如果此时已将 deviceModeSeq 用于设 Drive Strength，但是选择得是写入 Flash 非易失性存储器，这需要确保 waitTimeCfgCommands 延时对于写入非易失性寄存器也足够。翻看 MT35X 的数据手册可以发现，写入非易失性寄存器 cycle time 典型值是 0\.2s，最大值是 1s。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT_FDCB_MultiCfg_MT35X_AC.PNG)


　　再对比看看旺宏以及华邦家的 OctalFlash 非易失性寄存器写入时间，发现旺宏最短(\<\=60us)，华邦其次(\<\=15ms)，镁光有点略长了（\<\=1s）。越短的写入时间，启动时间也越短。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT_FDCB_MultiCfg_MX25UM_AC.PNG)


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT_FDCB_MultiCfg_W35T51_AC.PNG)


　　如果此时 waitTimeCfgCommands 设置对写入非易失性寄存器来说不够，即没等到 Drive Strength 配置完成就去做切换 OPI 模式操作，这会导致模式切换失败（Flash 在 Busy 期间不能接受任何写入命令），当然也就无法正常启动了。


　　以上介绍得都是比较复杂的 Flash 寄存器配置场景，如果是单纯的易失性寄存器配置操作且不涉及模式切换，那么 FDCB 里这四个配置顺序也就不重要了，而且也不需要启用 waitTimeCfgCommands，这样也不需要过多考虑了。


　　至此，在FDCB里配置串行NOR Flash多个寄存器的注意事项痞子衡便介绍完毕了，掌声在哪里\~\~\~


### 欢迎订阅


文章会同时发布到我的 [博客园主页](https://github.com):[MeoMiao 萌喵加速](https://biqumo.org)、[CSDN主页](https://github.com)、[知乎主页](https://github.com)、[微信公众号](https://github.com) 平台上。


微信搜索"**痞子衡嵌入式**"或者扫描下面二维码，就可以在手机上第一时间看了哦。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/github/pzhMcu_qrcode_258x258.jpg)


