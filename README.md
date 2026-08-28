# 硬件协议图解 · HW Protocol Lab

以交互式时序图与动画详解常见硬件通信协议，适合嵌入式工程师学习与查阅。

| 协议 | 内容 |
|------|------|
| [I²C](docs/i2c.html) | 起始/停止条件、应答机制、地址寻址、时钟拉伸、多主机仲裁、Linux i2c-tools 虚拟终端 |
| [I²S](docs/i2s.html) | Philips / LJ / RJ 数据格式对比、主从时钟关系、声道对齐 |
| [UART](docs/uart.html) | 起始位、数据位、校验位、停止位帧结构与波特率动画 |
| [IR 红外](docs/ir.html) | 38 kHz 载波调制、NEC/RC-5/SIRC 协议、波形实验室、解码状态机与 LIRC 实战 |
| [DMIC/PDM](docs/dmic.html) | 数字麦克风 1-bit 密度调制、L/R 时沿分离、÷64 抽取滤波、Linux/Android 实战 |


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
