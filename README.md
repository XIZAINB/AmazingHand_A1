# AmazingHand_A1

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-AidLux%20Rhino%20Pi%20A1%20%7C%20Ubuntu-orange)]()
[![Python](https://img.shields.io/badge/Python-3.10-3776AB)]()

在 **AidLux Rhino Pi A1（Ubuntu）** 上驱动 [Pollen Robotics AmazingHand](https://github.com/pollen-robotics/AmazingHand) 灵巧手的适配层与工具集。

> 8 自由度开源灵巧手 · 8 × Feetech SCS0009 总线舵机 · Python + `rustypot`

> **English**: Deployment tooling and a portable Python driver for the
> [AmazingHand](https://github.com/pollen-robotics/AmazingHand) dexterous hand
> running on an AidLux Rhino Pi A1 (Ubuntu) single-board computer.
> The upstream demo hard-codes `COM11`; this repo adds serial auto-detection,
> servo scanning, left/right hand support and a dry-run mode.

---

## 这个仓库解决什么问题

上游 AmazingHand 提供了完整的 CAD、Arduino 示例和一份 Python demo，但那份 demo：

- 串口写死 `COM11`，换机器就跑不起来
- 不做舵机在线检查，接错线时直接抛 `Timeout error`，报错信息毫无指向性
- 左右手要手改源码里的 `Side = 1`
- 没有无硬件调试手段

本仓库是它在 **AidLux Rhino Pi A1** 上的适配版本，改动集中在可移植性与可调试性：

| 改动 | 说明 |
|---|---|
| 串口自动探测 | 不再写死端口。Linux 扫 `/dev/ttyUSB*`、`/dev/ttyACM*`、`/dev/ttyCH343USB*` 等；Windows 用 `pyserial` 枚举真实 COM 口 |
| 舵机在线扫描 | 启动时扫描 ID 1~8，逐个上报，缺失 ID 会明确列出 |
| 左右手切换 | `--side right \| left`，无需改源码 |
| DRY-RUN 模式 | `--dry-run` 不接硬件，只打印每帧目标角度，用于验证逻辑与调试手势 |
| 中间位可传入 | `--middle-pos` 直接传 8 个标定值，配合标定流程使用 |
| 状态诊断工具 | `probe_servos.py` 读取每个舵机的位置 / 电压 / 温度 |
| Windows 兼容 | 同一套代码在 Windows + CH340 转接板上亦可运行 |

---

## 硬件

| 部件 | 说明 |
|---|---|
| 主控 | AidLux Rhino Pi A1（Ubuntu） |
| 手 | AmazingHand，8 × Feetech **SCS0009** 总线舵机 |
| 转接板 | Feetech TTL 总线转接板（USB-Serial，CH340 / CP2102 等） |
| 总线参数 | **1 000 000 bps**，舵机 ID 1~8 |

舵机 ID 与手指的对应关系（每指 2 个舵机，`MiddlePos` 索引为 `id-1`）：

| 手指 | 舵机 ID | MiddlePos 索引 |
|---|---|---|
| index 食指 | 1, 2 | 0, 1 |
| middle 中指 | 3, 4 | 2, 3 |
| ring 无名指 | 5, 6 | 4, 5 |
| thumb 拇指 | 7, 8 | 6, 7 |

> 机械结构、打印件、装配图纸见上游仓库：<https://github.com/pollen-robotics/AmazingHand>（本仓库不重复分发）

---

## 快速开始（AidLux Rhino Pi A1 / Ubuntu）

### 1. 克隆与安装依赖

```bash
git clone https://github.com/XIZAINB/AmazingHand_A1.git
cd AmazingHand_A1

python3 -m venv .venv
source .venv/bin/activate
pip install -r tools/requirements.txt
```

### 2. 串口权限

Ubuntu 下访问 `/dev/ttyUSB*` 需要串口权限，否则会报 `Permission denied`：

```bash
sudo usermod -aG dialout $USER
```

**重新登录**后生效。不想改权限也可以临时用 `sudo` 运行。

### 3. 确认转接板已识别

```bash
ls -l /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
dmesg | tail -20          # 插拔转接板，看内核是否枚举成功
```

### 4. 扫描舵机

```bash
python tools/probe_servos.py          # 自动扫描所有串口
python tools/probe_servos.py /dev/ttyUSB0
```

正常输出：

```
== /dev/ttyUSB0 ==
  [发现] 舵机 ID: [1, 2, 3, 4, 5, 6, 7, 8]
    ID  1: 位置=+1.2579 rad  电压=51.0 V  温度=21 °C
    ...
✅ /dev/ttyUSB0  -> 舵机 [1, 2, 3, 4, 5, 6, 7, 8]
```

> 电压字段是舵机原始单位 **0.1 V**，`51.0` 表示 **5.1 V**。

### 5. 跑手势

```bash
python tools/hand_demo.py                 # 自动找串口，跑一遍全部手势
python tools/hand_demo.py --side left     # 左手
python tools/hand_demo.py --loops 0       # 无限循环，Ctrl+C 停止
python tools/hand_demo.py --dry-run       # 不接硬件，只打印目标角度
```

完整参数与 Windows 下的用法见 **[tools/README.md](tools/README.md)**。

---

## 目录结构

```
AmazingHand_A1/
├── LICENSE                  Apache 2.0 全文
├── NOTICE                   上游归属声明（Apache 2.0 §4d 要求）
├── README.md                本文件
└── tools/
    ├── README.md            部署与排障手册（参数表、COM 口识别、常见问题）
    ├── hand_demo.py         手势 Demo（衍生自上游 PythonExample）
    ├── probe_servos.py      舵机总线扫描与状态读取
    ├── requirements.txt     依赖清单
    └── watch_usb.sh         USB 热插拔监听（Linux）
```

---

## 标定

所有手势都是「中间位 + 偏移量」算出来的，**中间位不准，手势全歪**。
`hand_demo.py` 中的 `DEFAULT_MIDDLE_POS = [3, 0, -5, -8, -2, 5, -12, 0]` 是上游作者的标定值，**不能直接用于你的手**。

标定流程参见上游 `ArduinoExample/Hand_FingerMiddlePos`。标好后用

```bash
python tools/hand_demo.py --middle-pos 3 0 -5 -8 -2 5 -12 0
```

逐组验证，确认满意再写入代码默认值。

---

## 故障排查

| 现象 | 原因 / 处理 |
|---|---|
| `Permission denied: '/dev/ttyUSB0'` | 未加入 `dialout` 组，见「串口权限」 |
| 扫不到任何舵机 | 90% 是串口选错（Windows 上尤其容易选到蓝牙虚拟串口）；其次检查转接板接线（信号 / 电源 / GND）与舵机供电 |
| `RuntimeError: Timeout error` | 端口上没有舵机应答。先跑 `probe_servos.py` 确认链路 |
| 转接板指示灯亮但仍扫不到 | 指示灯只代表转接板通电，不代表数据线通。确认接的是**总线口**而非 PWM 口 |
| 部分 ID 未响应 | 该舵机未接、ID 非 1~8，或总线接触不良 |
| 手势方向相反 / 撞限位 | `--side` 用反了，或中间位未标定 |
| 插拔后串口名变化 | `/dev/ttyUSB0` ↔ `/dev/ttyUSB1` 会变，建议用 `probe_servos.py` 自动扫描 |

---

## 许可协议

本仓库代码以 **Apache License 2.0** 发布，全文见 [LICENSE](LICENSE)。

选择 Apache 2.0 而非 MIT 的原因：本仓库的 `tools/hand_demo.py` 衍生自
[AmazingHand](https://github.com/pollen-robotics/AmazingHand) 的 `PythonExample/AmazingHand_Demo.py`，
而该上游项目以 Apache 2.0 发布。Apache 2.0 要求衍生作品保留原许可证并附带归属声明，
因此本仓库沿用同一协议，归属信息见 [NOTICE](NOTICE)。

上游机械设计（CAD / STL / 装配图纸）另以 **CC BY 4.0** 发布，本仓库不分发这些文件。

---

## 致谢

- [Pollen Robotics](https://www.pollen-robotics.com/) — AmazingHand 原始设计与开源
- [rustypot](https://github.com/pollen-robotics/rustypot) — Feetech SCS 系列总线舵机协议实现

## 免责声明

本项目为第三方适配，非 Pollen Robotics 官方发布。舵机运动涉及机械结构，调试时请
先使用 `--dry-run` 验证角度范围，避免撞限位损坏打印件。使用风险自负。
