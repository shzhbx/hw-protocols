# 学习型红外收发方案 · IR Learning Blaster

> 平台：ESP32（独立万能遥控网关）与 ARM Linux（树莓派 / 全志 / 瑞芯微等已有 Linux 板）
> 目标：学习任意遥控器按键 → 存储成库 → 稳定回放（发射），支持接入 Home Assistant / MQTT。
> 配套协议背景知识见 [IR 红外遥控协议详解](ir.html)。

---

## 0. 一页速览（TL;DR）

```
        学习流程                                回放流程
原遥控器 ──▶ [接收头] ──▶ 采集 raw 时序 ──▶ 归一化+校验 ──▶ JSON 存储
                                                      │
目标设备 ◀── [IR LED] ◀── 载波调制发射 ◀── 读库 ◀────┘（REST / MQTT 触发）
```

| 关键决策 | 结论 | 理由 |
|---|---|---|
| 学习方式 | **raw 时序采集为主**，协议解码为辅 | 万能学习必须兼容未知/私有协议（尤其空调） |
| ESP32 收发 | **RMT 外设**（或 IRremoteESP8266 库） | 硬件边沿时间戳，µs 精度，CPU 零占用 |
| Linux 收 | 内核 `gpio-ir-rx` 驱动 → `/dev/lirc0` | 用户态轮询/中断延迟不可控，必须内核时间戳 |
| Linux 发 | **`pwm-ir-tx`（首选）** / `gpio-ir-tx` | PWM 出载波最准；hrtimer 翻转 GPIO 次之 |
| 载波频率 | 默认 38 kHz 回放；宽带接收为升级项 | 90%+ 家电在 36~40 kHz，38 kHz 头可通吃大半 |
| 存储 | JSON（raw 数组 + 元信息） | 人可读、好 diff、两平台同构 |

---

## 1. 目标与边界

**做**：学习按键（raw）、命名存储、稳定回放、重复码处理、网络触发（HTTP/MQTT）、可选协议解码（NEC/RC-5/SIRC…）。

**不做（明确边界）**：
- 不做空中抓包干扰（发射时不学习，半双工即可）；
- 不承诺穿墙/超视距（红外物理特性）；
- 载波频率测量只在宽带接收升级硬件上提供（见 §3.1）。

**验收指标**（见 §9）：常见遥控学习成功率 ≥95%，回放正对距离 ≥8 m。

---

## 2. 总体架构

两平台共用同一套四层模型，仅硬件抽象层不同：

```
┌─────────────────────────────────────────────────┐
│ 应用层    REST / MQTT / 按键触发  ·  HA 集成      │
├─────────────────────────────────────────────────┤
│ 服务层    学习状态机 · 按键库(JSON) · 回放调度    │
├─────────────────────────────────────────────────┤
│ 算法层    归一化(50µs桶) · 一致性校验 · 重复检测  │
├──────────────────────┬──────────────────────────┤
│ HAL                  │ ESP32: RMT rx/tx(硬件)    │
│                      │ Linux : gpio-ir-rx /      │
│                      │        pwm-ir-tx(内核)    │
└──────────────────────┴──────────────────────────┘
```

采集与回放统一用 **pulse/space 微秒序列**（即 LIRC mode2 语义）作为中间表示——
这是整个方案的"通用货币"，ESP32 的 RMT item、Linux 的 lirc 数据都能无损互转。

---

## 3. 硬件设计

### 3.1 接收通道

**标配**：一体化解调接收头 **VS1838B / TSOP31238**（38 kHz）。
- 输出开漏/推挽（模块带上拉），空闲高，有载波拉低；
- VCC 就近 **100 nF 去耦**，串 100 Ω + 47 µF 更佳，远离 DC-DC 与 WiFi 天线；
- 引脚定义以手册为准（常见 VS1838B 面对透镜从左到右 OUT / GND / VCC）。

**升级位（真"万能学习"）**：宽带接收（无解调）**TSMP58000** 或 光电二极管 + 比较器：
- 用途①：实测载波 36~56 kHz，回放频率分毫不差；
- 用途②：学习非标准 / 极快编码。
- PCB 预留焊位即可，首版可不上。

> 38 kHz 接收头对 36/40 kHz 遥控灵敏度会降但多半可用——学习时靠近到 10 cm 基本都能抓到。

### 3.2 发射通道

大功率 940 nm IR LED + 三极管驱动，载波由 GPIO/PWM 提供给基极（栅极）：

```
ESP32/Linux GPIO(PWM 38kHz, duty 1/3)
   │
   ├─ 1kΩ ──▶ NPN(MMBT3904/S8050) 基极        或 AO3400 NMOS: 栅极串100Ω+10k下拉
   │           C ──▶[ IR LED ]──▶ 8.2Ω ── 3V3
   │           E ── GND
```

**电流估算**（3.3 V 轨，Vf≈1.4 V，Vce(sat)≈0.2 V）：
- 峰值电流 ≈ (3.3 − 1.4 − 0.2) / 8.2 Ω ≈ **210 mA**，1/3 占空下平均 ≈70 mA；
- 更远距离：两颗 LED 串联（Vf≈2.8 V，R 改 3.9 Ω）或双管并联分流；
- ESP32 GPIO 本身只能 source ~40 mA，**必须用三极管/MOSFET**，不要直推 LED。

**布局**：发射管大角度朝外（或两颗指向不同方向成"泛射"）；接收头与发射管保持 >3 cm 并加挡光，防止自发自收。

### 3.3 两平台接线

| 信号 | ESP32 | ARM Linux（示例） |
|---|---|---|
| 接收 OUT | GPIO14（避开 strapping 脚 0/2/12/15） | GPIO17（`gpio-ir-recv`） |
| 发射驱动 | GPIO4 | GPIO18（`pwm-ir-tx`，注意与音频 PWM 冲突时换脚） |
| 电源 | 3V3 / GND 共地 | 3V3 / GND 共地 |

---

## 4. ESP32 实现

三条路线，按投入从低到高：

### 路线 A：ESPHome（零固件代码，HA 原生集成）

```yaml
remote_receiver:
  pin: { number: GPIO14, inverted: true }
  dump: raw            # 学习：看日志拿 raw 数组
  tolerance: 5%

remote_transmitter:
  pin: GPIO4
  carrier_duty_percent: 33%

switch:
  - platform: template
    name: "电视电源"
    turn_on_action:
      remote_transmitter.transmit_raw:
        code: [9000, -4500, 600, -550, 600, -1650, ...]   # 正=载波时长, 负=空闲时长(µs)
```
学习流程：按住原遥控按键 → 日志复制 raw → 填进 YAML。适合个人一两个房间的轻量需求。

### 路线 B：Arduino + IRremoteESP8266（推荐，平衡）

库自带 50+ 协议收发 + raw 采集，ESP32 后端自动走 RMT：

```cpp
#include <IRremoteESP8266.h>
#include <IRrecv.h>
#include <IRsend.h>
#include <IRutils.h>

const uint16_t kRecvPin = 14, kSendPin = 4;
IRrecv irrecv(kRecvPin, 50, 1024);   // 断帧超时 50ms，缓冲 1024 边沿(空调长帧够用)
IRsend  irsend(kSendPin);

void setup() {
  Serial.begin(115200);
  irrecv.enableIRIn();
  irsend.begin();
}

void loop() {
  decode_results r;
  if (irrecv.decode(&r)) {
    if (r.repeat) { irrecv.resume(); return; }        // 先忽略重复帧
    Serial.printf("proto=%s bits=%u value=0x%llX\n",
                  typeToString(r.decode_type).c_str(), r.bits,
                  (unsigned long long)r.value);
    // raw 回放：rawbuf 以 50µs tick 记录，×50 还原成 µs
    uint16_t raw[600]; size_t n = 0;
    for (uint16_t i = 1; i < r.rawlen && n < 600; i++)
      raw[n++] = r.rawbuf[i] * 50;
    // irsend.sendRaw(raw, n, 38);                   // 38kHz 载波回放
    irrecv.resume();
  }
}
```

### 路线 C：自研固件（ESP-IDF，RMT 直驱）

- **RX**：`rmt_new_rx_channel`（IDF5），分辨率 1 µs，多 mem block + ping-pong 收空调长帧；
- **TX**：`rmt_new_tx_channel` + `tx_carrier`（38 kHz / duty 33%），把 µs 时序转 RMT item 流式发送；
- 通道数随型号不同（ESP32 8 通道；C3 2+2；S3 4+4），规划好收发各占一路；
- 网络：`esp_http_server`（REST）或 MQTT；存储：LittleFS `/ir/codes.json` + NVS 存配置；
- OTA：`esp_https_ota` 必备（现场设备别让自己够不着）。

REST API 设计（两平台通用，见 §8）。

---

## 5. ARM Linux 实现

### 5.1 内核层（关键：不要在用户态 bit-bang）

用户态 GPIO 翻转受调度延迟影响（毫秒级抖动），38 kHz 载波和 µs 级时宽都做不了。全部依赖内核驱动：

**接收** — `gpio-ir-rx`（边沿中断 + ktime 时间戳 → mode2 数据）：

```bash
# 树莓派 /boot/firmware/config.txt:
dtoverlay=gpio-ir-recv,gpio_pin=17
# 重启后出现 /dev/lirc0，验证：
sudo ir-ctl -d /dev/lirc0 -r -m        # 对着接收头按遥控，打印 pulse/space 微秒序列
```

**发射（首选 PWM）** — `pwm-ir-tx`：载波由 PWM 硬件生成，精度最高、CPU 零占用：

```bash
dtoverlay=pwm-ir-tx                    # 默认占用音频 PWM 通道，与模拟音频互斥
```

**发射（备选 GPIO）** — `gpio-ir-tx`：任意 GPIO，hrtimer 软件翻转载波（精度略低、占用 CPU，短帧无感）：

```bash
dtoverlay=gpio-ir-tx,gpio_pin=18
```

通用发行版（非树莓派）在设备树里挂 `gpio-ir-recv` / `gpio-ir-tx` / `pwm-ir-tx` 节点即可，驱动均在主线内核。

### 5.2 用户态：ir-ctl 三板斧

```bash
# ① 学习（raw）—— timeout 控制学习窗口，窗口内所有帧都会被打印
timeout 6 sudo ir-ctl -d /dev/lirc0 -r -m

# ② 回放 raw 文件（驱动自动调制载波）
sudo ir-ctl -d /dev/lirc0 -s /var/lib/ir-learn/tv_power.raw

# ③ 已知协议直接发扫描码
sudo ir-ctl -d /dev/lirc0 -S nec:0x0045
```

raw 文件格式（人可读，可手编）：

```
carrier 38000
duty 33
pulse 9024
space 4497
pulse 562
space 1685
...
```

### 5.3 守护进程骨架（Python）

学习/回放核心逻辑，套一层 FastAPI/Flask 即成服务；也可直接挂到 Home Assistant 的 MQTT：

```python
import subprocess, json, re, time, statistics

DEV = "/dev/lirc0"
LIB = "/var/lib/ir-learn"

def capture(timeout=6):
    """采集 raw，返回帧列表，每帧 [pulse, space, pulse, ...]（µs）"""
    p = subprocess.run(["ir-ctl", "-d", DEV, "-r", "-m"],
                       capture_output=True, text=True, timeout=timeout + 2)
    frames, cur = [], []
    for m in re.finditer(r"(pulse|space)\s+(\d+)", p.stdout):
        kind, us = m.group(1), int(m.group(2))
        if kind == "space" and us > 100_000:   # >100ms 静默 = 帧边界
            if cur: frames.append(cur)
            cur = []
        else:
            cur.append(us)
    if cur: frames.append(cur)
    return normalize(frames)

def normalize(frames, bucket=50):
    """去毛刺(<200µs 合并) + 50µs 桶化"""
    out = []
    for f in frames:
        g = [max(bucket, round(x / bucket) * bucket) for x in f if x >= 100]
        if g: out.append(g)
    return out

def learn(name, times=3):
    """多次采集做一致性校验，全部相似才入库"""
    shots = [capture() for _ in range(times)]
    base = shots[0]
    for s in shots[1:]:
        if len(s) != len(base) or len(s[0]) != len(base[0]):
            raise RuntimeError("多次采集不一致，请重新学习")
        for a, b in zip(s[0], base[0]):
            if abs(a - b) > max(a, b) * 0.25:
                raise RuntimeError("时序偏差过大，请重新学习")
    raw = base[0]
    with open(f"{LIB}/{name}.json", "w") as fp:
        json.dump({"name": name, "carrier_hz": 38000,
                   "raw": raw, "learned_at": time.strftime("%FT%T%z")}, fp)

def send(name):
    d = json.load(open(f"{LIB}/{name}.json"))
    lines = [f"carrier {d['carrier_hz']}", "duty 33"]
    lines += [f"{'pulse' if i % 2 == 0 else 'space'} {us}" for i, us in enumerate(d["raw"])]
    fn = f"/tmp/{name}.raw"
    open(fn, "w").write("\n".join(lines) + "\n")
    subprocess.run(["sudo", "ir-ctl", "-d", DEV, "-s", fn], check=True)
```

> 高级：不走子进程，直接 `open("/dev/lirc0")` + `LIRC_MODE_MODE2` 读 int32 字（`LIRC_MODE2_PULSE|µs`）效率更高，功能一致。
> 宽带学习：支持 learning 模式的 USB 接收棒（mceusb 等）配 `ir-ctl -r -w`，可同时测出载波频率。

---

## 6. 学习算法（两平台通用）

1. **进入学习**：5~10 s 窗口，指示灯提示"请按原遥控按键"；
2. **帧边界**：静默 > **100~150 ms** 判定一帧结束（经验值 ≥ 2.5× 协议最大 space）；
3. **去毛刺**：<100~200 µs 的窄段并入邻近段（日光灯/开关电源噪声）；
4. **归一化**：全部时长 **四舍五入到 50 µs 桶**，抗抖动、利于比对与压缩；
5. **一致性校验**：要求用户连按 2~3 次，序列长度相等且逐段容差 **±25%** 才通过，否则提示重学；
6. **重复结构识别**：
   - 后续帧与首帧完全相同 → 记 `repeat_mode=frame`（SIRC 风格连发）；
   - 后续是短帧（如 NEC 9+2.25+0.56 ms）→ 单独存 `repeat` 数组 + `repeat_gap`；
   - 只有一帧（多数空调）→ 无重复；
7. **双表示存储**：命中已知协议时额外存 `protocol + scancode`（紧凑可读），raw 始终保留（保真回放）。

---

## 7. 数据格式与存储

`/ir/codes.json`（ESP32: LittleFS；Linux: `/var/lib/ir-learn/`）：

```json
{
  "version": 1,
  "buttons": [
    {
      "id": "tv_power",
      "name": "电视·电源",
      "carrier_hz": 38000,
      "duty": 33,
      "source": "raw",
      "raw": [9024, 4497, 562, 561, 562, 1685],
      "repeat": [9024, 2250, 562],
      "repeat_gap_ms": 110,
      "protocol": "nec",
      "scancode": "0x0045",
      "learned_at": "2026-08-17T12:00:00+08:00"
    }
  ]
}
```

导入导出即整文件拷贝；两平台格式完全一致，ESP32 学的码可直接拷给 Linux 发（反之亦然）。

---

## 8. 控制 API（REST 示例，MQTT 同构）

| 方法 | 路径 | 功能 |
|---|---|---|
| POST | `/api/learn/start?name=tv_power&times=3` | 进入学习窗口 |
| GET | `/api/learn/status` | 学习中 / 成功 / 失败原因 |
| GET | `/api/buttons` | 列出按键库 |
| POST | `/api/send/tv_power?hold=0` | 回放（hold=秒数 → 循环重发/重复码） |
| DELETE | `/api/buttons/tv_power` | 删除 |
| POST | `/api/buttons/import` · `/api/buttons/export` | 库导入导出 |

MQTT 话题设计：`ir/blaster/{device}/learn/start`、`ir/blaster/{device}/send`（payload 为按钮 id），状态回报 `ir/blaster/{device}/status`。

---

## 9. 测试与验收

- [ ] 学习：NEC 电视、RC-5 机顶盒、SIRC 音响、变频空调（长帧 ≥200 边沿）各学 20 次，成功率 ≥95%；
- [ ] 回放：正对 8 m、偏角 ±30° 4 m 可控；被控设备响应率 100%（各 50 次）；
- [ ] 重复码：长按"音量+"持续增；点按只发一帧；
- [ ] 抗扰：日光灯下、正午窗边各复测学习与回放；
- [ ] 并发：回放期间收到学习请求应排队或拒绝，不交错；
- [ ] 断电恢复：重启后按键库完整，回放一致；
- [ ] （Linux）`htop` 观察回放期间 CPU 占用 <5%（pwm-ir-tx 应接近 0）。

---

## 10. 平台对比与选型

| 维度 | ESP32 | ARM Linux |
|---|---|---|
| BOM 成本 | ~¥20（模块+收发元件） | 复用已有板子则近乎 ¥5 |
| 时序精度 | RMT 硬件，±1 µs | 内核 hrtimer（µs~十µs 级）；PWM 载波精确 |
| 开发形态 | 固件（Arduino/IDF/ESPHome） | 设备树 + Python，无固件烧写 |
| 生态 | MQTT/HA 生态成熟 | lircd/HA/脚本全家桶 |
| 功耗 | 可电池/休眠 | 一般常电 |
| 长帧（空调） | 注意 RMT 缓冲与分块 | 内核驱动天然支持 |
| 适用 | 独立万能遥控网关、批量部署 | 已有 Linux 网关/盒子顺手加能力 |

**建议**：两台设备共用同一 JSON 库与同一套学习算法语义；先在 Linux 上把学习算法用 Python 调通（迭代快），再移植到 ESP32 固件。

---

## 11. 里程碑

| 阶段 | 内容 | 完成标志 |
|---|---|---|
| M1 | 硬件点亮 | 收到原始脉冲可见、LED 能闪出载波 |
| M2 | 学习/回放闭环 | 学一个电视电源键并成功控制电视 |
| M3 | 库与 API | JSON 存储 + REST/MQTT 全通 |
| M4 | 算法加固 | 一致性校验、重复码、去毛刺、空调长帧 |
| M5 | 体验与验收 | 距离/角度/抗扰过 §9 清单，外壳安装 |

---

## 附录：常见问题速查

| 症状 | 病因 | 处置 |
|---|---|---|
| 学不到任何脉冲 | 载波不匹配（36/40 kHz 遥控） | 靠近重试；上宽带接收升级位 |
| 学到的帧每次都不一样 | 环境光噪声 / 原遥控电池弱 | 遮光、换电池；放宽一致性到 ±30% 再观察 |
| 回放设备无反应 | 载波频率偏差、发射电流不足 | 核对 `carrier` 值；减小限流电阻/加 LED |
| 空调码偶尔不识别 | 长帧被截断 | 加大采集缓冲（ESP32 bufsize / RMT mem block） |
| Linux 回放偶发变形 | 用了 gpio-ir-tx 且系统繁忙 | 换 pwm-ir-tx；或降低负载 |
| 自发自收 | 收发头距离过近 | 挡光、拉开距离；软件上回放时关接收 |
