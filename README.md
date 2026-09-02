# 硬件协议图解 · HW Protocol Lab

以交互式时序图与动画详解常见硬件通信协议，适合嵌入式工程师学习与查阅。

| 协议 | 内容 |
|------|------|
| [I²C](docs/i2c.html) | 起始/停止条件、应答机制、地址寻址、时钟拉伸、多主机仲裁、Linux i2c-tools 虚拟终端 |
| [I²S](docs/i2s.html) | Philips / LJ / RJ 数据格式对比、主从时钟关系、声道对齐 |
| [UART](docs/uart.html) | 起始位、数据位、校验位、停止位帧结构与波特率动画 |
| [IR 红外](docs/ir.html) | 38 kHz 载波调制、NEC/RC-5/SIRC 协议、波形实验室、解码状态机与 LIRC 实战 |
| [DMIC/PDM](docs/dmic.html) | 数字麦克风 1-bit 密度调制、L/R 时沿分离、÷64 抽取滤波、Linux/Android 实战 |
| [SDIO](docs/sdio.html) | WiFi 模块高速通道：48-bit 命令帧、CMD53 块读时序动画、枚举状态机、吞吐计算器与 Linux mmc 实战 |
| [SPI](docs/spi.html) | CPOL/CPHA 四模式波形实验室、移位寄存器环形交换动画、QSPI 变体与 spidev 虚拟终端 |
| [RS-485](docs/rs485.html) | A/B 差分长线与终端电阻、DE/RE 方向切换对错动画、Modbus RTU 帧动画与 CRC16 手算实验室 |
| [1-Wire](docs/onewire.html) | 复位/存在脉冲波形、读写时隙对比实验室、Search ROM 二叉树枚举、DS18B20 实战 |
| [CAN](docs/can.html) | 数据帧逐位示波器、双节点 ID 仲裁对决、位填充与错误计数状态机、CAN FD 与 SocketCAN 实战 |
| [TDM/PCM](docs/tdm.html) | 三种帧同步格式对比动画、交互式时隙分配器、BCLK 计算器与 ASoC set_tdm_slot 实战 |
| [eMMC/UFS](docs/emmc.html) | Boot/RPMB 分区与 HMAC 防回滚动画、DS→HS400 采样波形实验室、eMMC vs UFS 队列对决 |
| [DDR](docs/ddr.html) | 双沿采样示波器、Bank/Row/Column 寻址动画、tRCD/CL 时序实验室、六代演进与带宽计算器 |
| [RGB](docs/rgb.html) | PCLK/DE/HSYNC/VSYNC 时序、640×480 逐行扫描动画、六参数 porch 实验室与 PCLK 计算器 |
| [LVDS](docs/lvds.html) | 差分对电流动画、7:1 串行化实验室、JEIDA/VESA 位映射切换器、V-by-One 8b/10b 演进 |
| [MIPI DSI](docs/mipi-dsi.html) | D-PHY LP/HS 双态换挡灵魂动画、数据包 ECC+CRC 校验传送带、Video/Command 模式与带宽计算器 |
| [MIPI CSI-2](docs/csi2.html) | 帧与包逐字段拆解动画、RAW10 打包、4-lane 分发合并、VC 复用与 v4l2/media-ctl 实战 |
| [HDMI/DP](docs/hdmi.html) | TMDS 8b/10b 编码流水动画、扫描线里的 DE 与数据岛、4K60 带宽档位判定、DP MST 菊花链 |
| [USB](docs/usb.html) | NRZI+位填充三级流水动画、控制传输三阶段、枚举状态机、描述符树与 usbmon 抓包实战 |
| [PCIe](docs/pcie.html) | TLP 逐字段流入动画、LTSSM 链路训练状态机、credit 流控对照实验、lspci 解码实战 |
| [JTAG/SWD](docs/jtag.html) | 16 态 TAP 状态机模拟器、IDCODE 逐位移出时序、菊花链穿链演示、SWD 事务包与 OpenOCD 实战 |


## 在线浏览（GitHub Pages）

1. 将本仓库推送到 GitHub。
2. 仓库 **Settings → Pages**，Source 选择 **Deploy from a branch**，分支选 `main`、目录选 `/docs`，保存。
3. 稍等片刻即可通过 `https://<用户名>.github.io/hw-protocols/` 访问首页。

## 本地预览

页面均为纯静态 HTML，无需构建：

```bash
cd docs && python3 -m http.server 8000
# 浏览器打开 http://localhost:8000
```
