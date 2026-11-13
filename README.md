# HC-SR04 → APM 超声波改件 使用说明

本仓库收录 2014 年 panxu 的「超声波改件」源码：HC-SR04 + Arduino 伪装成 Maxbotix I2CXL 声呐（I2C `0x70`），供 APM / Pixhawk 飞控读取距离。

| 仓库路径 | 说明 |
|---|---|
| `ApmSonar/ApmSonar.ino` | 主程序，烧到 Arduino |
| `SR04/SR04.h`、`SR04/SR04.cpp` | HC-SR04 驱动库 |
| `SR04/examples/SR04_Example/SR04_Example.ino` | 传感器单独测试示例 |
| `SR04/keywords.txt` | Arduino IDE 语法高亮 |
| `SR04/Readme.txt` | 原安装说明 |
| `README.md` | 本文（完整使用教程） |

## 0. 这套东西做什么

用一块 Arduino 读取 HC-SR04 超声波模块，把 Arduino 伪装成 **Maxbotix I2CXL 声呐**（I2C 从机地址 `0x70`），让 APM / Pixhawk 飞控通过现成的 `MaxsonarI2CXL` 驱动拿到距离，用于低空定高、自动降落辅助、避障。

**不是 ArduPilot 飞控固件**，是烧在 Arduino 上的外挂转接板程序 + 配套 SR04 库。作者 panxu，2014 年。

> 先判断有没有必要用：ArduPilot 4.x 已原生支持 HC-SR04（`RNGFND1_TYPE = 30`）。若飞控有空闲引脚且用新版固件，直接接 SR04 更省事。本方案的价值在于走 I2C 总线、兼容老版 APM2 固件、或传感器离飞控较远时减少长线脉冲干扰。

## 1. 准备清单

| 类别 | 需要 |
|---|---|
| 主控板 | Arduino Pro Mini / Nano（ATmega168/328），建议 **3.3 V / 8 MHz 版本**，便于与飞控 I2C 电平匹配 |
| 传感器 | HC-SR04（或 HC-SR05 / US-015 等 Trig-Echo 型） |
| 飞控 | APM2.x 或 Pixhawk 系列，跑 ArduPilot Copter / Plane / Rover |
| 软件 | Arduino IDE 1.0.x（Readme 指定）或 1.6/1.8；Mission Planner |
| 其他 | 杜邦线、5 V 供电、必要时双向电平转换模块、I2C 上拉确认 |

> HC-SR04 需要 5 V。若 Arduino 用 3.3 V 版，传感器需单独 5 V 供电，且 **Echo 是 5 V 输出，必须分压（如 1 kΩ / 2 kΩ）后再接 Arduino D3**，否则可能损坏引脚。

## 2. 文件怎么放（整理结论）

**保持原有结构，不要移动或合并文件** —— 它已经是标准 Arduino 工程/库布局，改动反而会破坏编译。

```
ArduPilot-Sonar/
├── README.md                        ← 本文档（使用教程）
├── ApmSonar/
│   └── ApmSonar.ino                 ← 主程序，烧到 Arduino
└── SR04/                            ← 整个文件夹作为一个 Arduino 库
    ├── SR04.h                       ← 必须
    ├── SR04.cpp                     ← 必须
    ├── keywords.txt                 ← 仅 IDE 语法高亮
    ├── Readme.txt                   ← 原安装说明（GBK 编码）
    └── examples/
        └── SR04_Example/
            └── SR04_Example.ino     ← 单独测试传感器用
```

两种放置方式，选其一：

- **方式 A（推荐，最省事）**：把 `SR04.h`、`SR04.cpp` 复制到 `ApmSonar\` 目录内，与 `ApmSonar.ino` 同级。因为主程序写的是 `#include "SR04.h"`（双引号本地包含），Arduino IDE 会自动编译同目录的 `.cpp`。
- **方式 B（按原 Readme）**：把整个 `SR04` 文件夹复制到 Arduino IDE 的 `libraries\` 目录（1.0.x 为 `arduino-1.0\libraries\`，新版为 `文档\Arduino\libraries\`），**重启 IDE**，之后 `文件 → 示例 → SR04` 能看到示例即安装成功。

## 3. 接线

| HC-SR04 | Arduino | 备注 |
|---|---|---|
| VCC | 5 V | 3.3 V 板需外接 5 V，并与 Arduino 共地 |
| GND | GND | 必须共地 |
| Trig | **D2** | `#define TRIG_PIN 2` |
| Echo | **D3** | `#define ECHO_PIN 3`，5 V 电平注意分压 |

| Arduino ↔ 飞控 | 接线 |
|---|---|
| SDA | A4（ATmega328；Mega 为 D20）→ 飞控 I2C SDA |
| SCL | A5（ATmega328；Mega 为 D21）→ 飞控 I2C SCL |
| GND | 飞控 GND，**必须共地** |

注意事项：

- I2C 总线由飞控上拉，多数为 3.3 V。5 V Arduino 的 SDA/SCL 若带内部上拉可能把总线抬高，**建议用 3.3 V 板或加双向电平转换**。
- `0x70` 是 Maxbotix 默认地址，确认总线上没有其他设备占用（GPS、外置罗盘、OLED 常见冲突）。
- 传感器探头朝下、避开起落架和螺旋桨气流；超声对软布、斜面、吸音材料不敏感。

## 4. 步骤一：单独测 SR04

1. 打开 Arduino IDE，装好 SR04 库（见第 2 节方式 B），打开 `文件 → 示例 → SR04 → SR04_Example`。
2. 选择板型与端口，烧录。
3. 打开串口监视器，波特率 **9600**。
4. 正常输出：每秒一行 `xx cm`；用手在探头前移动，数值应随之变化。
5. 若一直是 `0` 或固定大数：检查 Trig/Echo 是否接反、供电是否为 5 V、探头是否被遮挡。

## 5. 步骤二：烧主程序

1. 打开 `ApmSonar\ApmSonar.ino`（工程目录名须与 `.ino` 同名，已是 `ApmSonar`）。
2. 确认 `SR04.h` / `SR04.cpp` 可被找到（方式 A 或 B）。
3. 烧录到 Arduino。烧完后串口**不再打印**，属正常。
4. 此时 Arduino 已成为 I2C 从机 `0x70`：飞控来读时返回 2 字节 `[高字节, 低字节]`，单位 cm。

主程序逻辑：`DistanceAvg(25,1)` 测 3 次取中值 → `tempread < 500` 才更新读数（否则保持上次值）→ `onRequest` 中断回调返回最新值。

## 6. 步骤三：飞控参数

**改完参数必须重启飞控**，否则测距仪不生效。

### ArduPilot 3.x / APM2（当年原配）

| 参数 | 值 | 说明 |
|---|---|---|
| `RNGFND_TYPE` | `2` | APM2-MaxbotixI2C（旧 wiki 写 4 已过时） |
| `RNGFND_ADDR` | `-1` | 使用驱动默认地址 0x70 |
| `RNGFND_PIN` | `-1` | — |
| `RNGFND_MIN_CM` | `20` | 小于此值视为不可靠 |
| `RNGFND_MAX_CM` | `400` | **不得超过 SR04 实际量程**，超了飞控会误判 |
| `RNGFND_GNDCLEAR` | 实测值 | 机体落地时探头到地面的距离（cm） |
| `RNGFND_ORIENT` | 朝下 | 按地面站选项选 Down |

### ArduPilot 4.x（参数名加了 `1`）

| 参数 | 值 | 说明 |
|---|---|---|
| `RNGFND1_TYPE` | `2` | MaxbotixI2C |
| `RNGFND1_ADDR` | 默认 `0` | 走驱动内建 0x70；需显式指定填 `112`（0x70 十进制，7-bit） |
| `RNGFND1_MIN_CM` | `20` | — |
| `RNGFND1_MAX_CM` | `400` | 官方 Maxbotix 示例为 700，SR04 请按实测填 |
| `RNGFND1_GNDCLEAR` | 实测值 | — |
| `RNGFND1_ORIENT` | 默认 25 (Down) | 保持朝下即可 |

> 4.0 起 `RNGFND_xxx` 全面改为 `RNGFND1_xxx`，旧参数名不再存在。以地面站实际列出的参数表为准。

## 7. 验证

1. Mission Planner → **飞行数据 → 状态**选项卡，找到 `sonarrange`。
2. 手在探头前从近到远移动，`sonarrange` 应在 20–400 cm 间连续变化。
3. 用 I2C 扫描确认总线上存在 `0x70` 设备（可用另一块 Arduino 跑 i2c_scanner）。
4. 读数正常后，再启用定高 / 自动降落相关功能，先地面测试再试飞。

## 8. 想改参数，改代码哪里

| 目的 | 位置 |
|---|---|
| 换 Trig / Echo 引脚 | `ApmSonar.ino` 的 `#define TRIG_PIN 2`、`#define ECHO_PIN 3` |
| 加强滤波 | `sr04.DistanceAvg(25,1)` 第二个参数 `count`，N 表示测 N+2 次去最大最小后取平均 |
| 改量程上限 | `ApmSonar.ino` 的 `if (tempread<500)` |
| 换 I2C 地址 | `Wire.begin(0x70)`，同时改飞控 `RNGFND1_ADDR` |
| 改测量超时 | `SR04.h` 的 `PULSE_TIMEOUT`（当前 150000 µs，源码注释误写 100 ms） |

注意：`DistanceAvg` 的第一个参数 `wait` 在源码中被 clamp 后**从未使用**，测量间隔实际固定由 `Distance()` 内的 `delay(25)` 决定，改它无效。

## 9. 故障排查

| 现象 | 常见原因 | 处理 |
|---|---|---|
| 串口一直 `0 cm` | 无回波、Trig/Echo 接反、供电不足 | 回第 4 节重测；确认 5 V |
| `sonarrange` 无数据 | 参数未重启、I2C 未接、未共地、地址不符 | 重启飞控；查 SDA/SCL/GND；确认 0x70 |
| 读数 0 或 `Bad Lidar Health` | 无回波时 `pulseIn` 返回 0，而 `tempread<500` **过滤不掉 0**，会把 0 cm 上报 | 见第 10 节建议 |
| 数值跳动大 | SR04 噪声、软表面、气流 | 增大 `DistanceAvg` 的 `count` |
| 距离卡住不更新 | 超过 500 cm 被丢弃，代码保持上次值 | 改大量程阈值或缩短目标距离 |
| 罗盘/OLED/GPS 同时异常 | I2C 地址冲突或总线电平/供电不足 | 换地址、加电平转换、外接 BEC 供电 |
| 偶尔出现明显错值 | `reading_cm` 为 16 位非 `volatile`，中断回调与主循环并发读写，8 位 AVR 上可能读到半更新值 | 见第 10 节建议 |

## 10. 已知限制（建议修改）

代码是 2014 年的简化实现，实机使用前建议改两处：

```cpp
// 1) 主循环：把 0 值也过滤掉
if (tempread > 0 && tempread < 500) {
    reading_cm = tempread;
}

// 2) 声明加 volatile，中断里做原子拷贝
volatile uint16_t reading_cm;
// requestEvent() 内用 noInterrupts()/interrupts() 包裹取值
```

其他限制：

- 未处理 Maxbotix 的 `0x51`（take range reading）写命令，靠「持续测量 + 读时给最新值」兼容，属简化实现。
- SR04 可靠量程通常 2–400 cm，室外强光/吸音表面表现差；`RNGFND_MAX_CM` 必须按实测填写，不能照抄 Maxbotix 的 700。
- 不支持热插拔；飞控开机前 Arduino 必须先上电。

## 附：目录内 6 个原始文件

| 文件 | 作用 |
|---|---|
| `ApmSonar/ApmSonar.ino` | 主程序，I2C 从机 0x70 |
| `SR04/SR04.h` | 类声明、引脚与超时常量 |
| `SR04/SR04.cpp` | 触发、脉宽测距、中值滤波实现 |
| `SR04/examples/SR04_Example/SR04_Example.ino` | 串口打印示例 |
| `SR04/keywords.txt` | IDE 语法高亮 |
| `SR04/Readme.txt` | 原安装说明（GBK 编码） |

源码注释与本文第 10 节包含主要实现说明与已知限制。
