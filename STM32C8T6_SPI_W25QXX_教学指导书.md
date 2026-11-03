# STM32C8T6_SPI_W25QXX 教学指导书

> 适用工程：`STM32C8T6_SPI`
> 目标芯片：STM32F103C8T6（64KB Flash / 20KB RAM）
> 存储芯片：W25QXX 系列 SPI NOR Flash
> 通信方式：USB CDC 虚拟串口 / USART1
> 文档版本：V1.0

---

## 目录

1. [项目概述](#1-项目概述)
2. [硬件准备与接线](#2-硬件准备与接线)
3. [软件环境](#3-软件环境)
4. [工程结构](#4-工程结构)
5. [系统时钟与 USB 48MHz](#5-系统时钟与-usb-48mhz)
6. [SPI 与 W25QXX 驱动](#6-spi-与-w25qxx-驱动)
7. [命令协议](#7-命令协议)
8. [USB CDC 虚拟串口](#8-usb-cdc-虚拟串口)
9. [PC 工具 Scom 使用](#9-pc-工具-scom-使用)
10. [编译、烧录与测试](#10-编译烧录与测试)
11. [常见问题与排查](#11-常见问题与排查)
12. [调试方法](#12-调试方法)
13. [实验与扩展](#13-实验与扩展)
14. [附录](#14-附录)

---

## 1. 项目概述

### 1.1 项目目标

本工程实现一个基于 STM32F103C8T6 的 W25QXX SPI Flash 烧录器：

- 通过 PC 的 USB 接口连接 STM32 原生 USB（PA11/PA12），识别为虚拟串口（CDC）。
- 通过命令协议完成 Flash 的识别、读取、擦除、写入和校验。
- 保留 USART1 命令接口作为备用通道。
- 支持 W25Q 系列以及兼容的 GD25Q、P25Q、XM25Q 等 SPI NOR Flash。

### 1.2 主要功能

| 功能 | 说明 |
|---|---|
| Flash 识别 | 读取 JEDEC ID，计算容量和型号 |
| 数据读取 | 任意地址、任意长度读取 |
| 扇区/块擦除 | 4KB 扇区、32KB 块、64KB 块自动选择 |
| 整片擦除 | Chip Erase |
| 页写入 | 自动处理 256 字节页边界 |
| 数据校验 | PC 端读回比较 |
| USB 虚拟串口 | CDC ACM，Windows 免驱（usbser） |
| UART 备用通道 | USART1，115200 8N1 |
| 流控保护 | USB 接收环形缓冲不足时自动 NAK，避免大文件丢包 |

### 1.3 系统框图

```text
┌──────────────┐    USB CDC (PA11/PA12)     ┌──────────────────┐
│   PC / Scom  │ ─────────────────────────► │                  │
│              │                            │  STM32F103C8T6   │
│              │    USART1 (PA9/PA10)       │                  │
│              │ ─────────────────────────► │                  │
└──────────────┘                            └────────┬─────────┘
                                                     │ SPI1
                                                     │ PA4 CS
                                                     │ PA5 SCK
                                                     │ PA6 MISO
                                                     │ PA7 MOSI
                                                     ▼
                                            ┌──────────────────┐
                                            │   W25QXX Flash   │
                                            └──────────────────┘
```

### 1.4 主要参数

| 项目 | 参数 |
|---|---|
| MCU | STM32F103C8T6，Cortex-M3，72MHz |
| Flash | 64KB |
| RAM | 20KB |
| SPI | SPI1，主机，模式 0，18MHz |
| USB | USB 2.0 Full Speed，12Mbps，CDC ACM |
| UART | USART1，115200 8N1 |
| 命令缓冲 | 2048 字节环形缓冲 |
| USB 收发缓冲 | 接收 2048 字节，发送分块阻塞 |
| 堆 | 0x800（2048 字节） |
| 栈 | 0x400（1024 字节） |

---

## 2. 硬件准备与接线

### 2.1 物料清单

| 物料 | 数量 | 说明 |
|---|---|---|
| STM32F103C8T6 最小系统板 | 1 | Blue Pill 或兼容板 |
| W25QXX 模块/芯片 | 1 | 3.3V 供电 |
| USB 数据线 | 1 | 必须是数据线，不能是纯充电线 |
| ST-Link V2 | 1 | 下载和调试 |
| 杜邦线 | 若干 | 尽量短，减少 SPI 干扰 |
| 3.3V 电源 | 1 | 若板子由 USB 供电可省略 |

### 2.2 接线表

#### SPI Flash

| STM32 引脚 | W25QXX 引脚 | 说明 |
|---|---|---|
| PA4 | CS / nSS | 片选，低有效 |
| PA5 | CLK / SCK | SPI 时钟 |
| PA6 | DO / MISO | 主入从出 |
| PA7 | DI / MOSI | 主出从入 |
| 3.3V | VCC | 3.3V |
| GND | GND | 共地 |

> 注意：W25QXX 是 3.3V 器件，不要接 5V。

#### USB

| STM32 引脚 | USB 接口 | 说明 |
|---|---|---|
| PA11 | D- | USB 数据负 |
| PA12 | D+ | USB 数据正 |
| 5V | VBUS | 可给板子供电 |
| GND | GND | 共地 |

> 关键：PA12（D+）需要 1.5kΩ 上拉到 3.3V。Blue Pill 板载已有；裸芯片/自制板必须外加，否则 PC 完全看不到设备。

#### USART1（备用）

| STM32 引脚 | USB-TTL | 说明 |
|---|---|---|
| PA9 | RXD | STM32 发送 |
| PA10 | TXD | STM32 接收 |
| GND | GND | 共地 |

#### SWD 下载

| STM32 引脚 | ST-Link | 说明 |
|---|---|---|
| PA13 | SWDIO | 数据 |
| PA14 | SWCLK | 时钟 |
| 3.3V | 3.3V | 参考电平 |
| GND | GND | 共地 |

### 2.3 硬件检查清单

- [ ] 3.3V 供电正常，W25QXX 的 VCC 对 GND 无短路。
- [ ] PA4 默认高电平，Flash 未被误选中。
- [ ] PA12 有 1.5kΩ 上拉到 3.3V。
- [ ] USB 线是数据线。
- [ ] 8MHz 晶振和匹配电容正常（本工程默认使用外部 8MHz 晶振）。
- [ ] SPI 线尽量短，必要时降低 SPI 时钟。

---

## 3. 软件环境

### 3.1 开发工具

| 工具 | 版本 | 用途 |
|---|---|---|
| Keil MDK / µVision | 5.33 | 编译、下载、调试 |
| ARM Compiler | V5.06 update 7 | C 编译器 |
| STM32CubeMX | 可选 | 外设初始化参考 |
| STM32CubeProgrammer | 可选 | 烧录、读寄存器 |
| Scom.exe | 随工程提供 | PC 端烧录工具 |
| 串口终端 | 任意 | 手动发送命令调试 |

### 3.2 工程路径

```text
STM32C8T6_SPI/
├─ Core/
│  ├─ Inc/
│  └─ Src/
├─ Drivers/
│  ├─ CMSIS/
│  └─ STM32F1xx_HAL_Driver/
├─ Middlewares/
│  └─ ST/STM32_USB_Device_Library/
├─ MDK-ARM/
│  ├─ STM32C8T6_SPI.uvprojx
│  └─ STM32C8T6_SPI/
│     └─ STM32C8T6_SPI.hex
└─ STM32C8T6_SPI.ioc
```

### 3.3 编译配置

- Device：`STM32F103C8`
- 宏定义：`USE_HAL_DRIVER, STM32F103xB`
- 输出：`MDK-ARM\STM32C8T6_SPI\STM32C8T6_SPI.hex`
- 参考占用：Code ≈ 19.3KB，RO ≈ 0.37KB，RW ≈ 0.4KB，ZI ≈ 10.5KB。

---

## 4. 工程结构

### 4.1 源文件职责

| 文件 | 职责 |
|---|---|
| `main.c` | 初始化、系统时钟、主循环 |
| `gpio.c` | GPIO 初始化（PA4 片选） |
| `spi.c` | SPI1 初始化 |
| `usart.c` | USART1 初始化、环形缓冲、USB/UART 发送桥接 |
| `w25qxx.c` | W25QXX 底层驱动 |
| `w25qxx_cmd.c` | 命令解析与执行 |
| `stm32f1xx_it.c` | 中断服务函数（SysTick、USART1、USB） |
| `usb_device.c` | USB Device 初始化 |
| `usbd_conf.c` | USB 底层 PCD 回调、PMA 配置、MSP |
| `usbd_desc.c` | USB 设备/字符串描述符 |
| `usbd_cdc_if.c` | CDC 类接口、USB 收发、流控 |
| `startup_stm32f103xb.s` | 启动文件、堆栈大小 |

### 4.2 启动流程

```text
上电/复位
  ↓
SystemInit()
  ↓
main()
  ↓
HAL_Init()                  // SysTick、NVIC 分组
  ↓
SystemClock_Config()        // HSE 8MHz → PLL 72MHz → USB 48MHz
  ↓
MX_GPIO_Init()
  ↓
MX_SPI1_Init()
  ↓
MX_USART1_UART_Init()
  ↓
MX_USB_DEVICE_Init()        // USBD_Init → CDC → USBD_Start
  ↓
W25QXX_CMD_Init()           // Flash 唤醒、读 ID
  ↓
while (1)
  W25QXX_CMD_Poll();        // 解析 UART/USB 命令
```

---

## 5. 系统时钟与 USB 48MHz

### 5.1 时钟树

```text
HSE 8MHz 晶振
  │
  ├── PLL × 9 ──► 72MHz ──► SYSCLK
  │                          ├── AHB  /1  = 72MHz
  │                          ├── APB1 /2  = 36MHz
  │                          └── APB2 /1  = 72MHz
  │
  └── PLL 72MHz / 1.5 ──► USB 48MHz
```

USB Full Speed 要求时钟 **48MHz ±0.25%**，所以不能随意使用 HSI。

### 5.2 关键代码

`Core/Src/main.c` 的 `SystemClock_Config()`：

1. 优先使用 **8MHz 晶振**：
   ```c
   RCC_OscInitStruct.HSEState = RCC_HSE_ON;
   RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;
   ```
2. 如果晶振配置失败，自动尝试 **外部时钟输入（BYPASS）**。
3. 如果仍然失败，回退到 **HSI/2 × 12 = 48MHz**，并把 USB 时钟源改为 PLL 直连。
4. USB 时钟：
   ```c
   PeriphClkInit.PeriphClockSelection = RCC_PERIPHCLK_USB;
   PeriphClkInit.UsbClockSelection = RCC_USBCLKSOURCE_PLL_DIV1_5;
   ```

> 重要经验：如果板子是 8MHz 晶振，却把 HSE 配成 `RCC_HSE_BYPASS`，`HAL_RCC_OscConfig()` 会等待 HSE 就绪直到超时，程序进入 `Error_Handler()` 死循环。此时 USB 外设根本没有初始化，PC 端会提示“设备描述符请求失败”或“无法识别 USB 设备”。

### 5.3 Flash 等待周期

72MHz 下需要 2 个 Flash 等待周期：

```c
HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2);
```

### 5.4 时钟自检

如果怀疑时钟不对，可以：

- 用 ST-Link 读取 `RCC->CR`，确认 `HSERDY=1`、`PLLRDY=1`。
- 读取 `RCC->CFGR`，确认 `SW=PLL`、`USBPRE=0`（1.5 分频）。
- 用 MCO 输出时钟到 PA8 观察（需要额外配置）。
- 检查 UART 波特率是否正常，间接判断系统时钟。

---

## 6. SPI 与 W25QXX 驱动

### 6.1 SPI1 配置

`Core/Src/spi.c`：

| 参数 | 值 | 说明 |
|---|---|---|
| Mode | Master | 主机 |
| Direction | 2 Lines | 全双工 |
| DataSize | 8 bit | 字节传输 |
| CPOL | Low | 空闲低电平 |
| CPHA | 1 Edge | 第一个边沿采样 |
| NSS | Soft | 软件片选 |
| Prescaler | 4 | 72MHz / 4 = 18MHz |
| FirstBit | MSB | 高位先发 |
| CRC | Disable | 不使用 |

SPI 模式 0（CPOL=0，CPHA=0）与 W25QXX 兼容。

### 6.2 W25QXX 常用命令

| 命令 | 操作码 | 说明 |
|---|---|---|
| Write Enable | `0x06` | 写使能，置 WEL |
| Read Status 1 | `0x05` | 读状态寄存器 |
| Read Data | `0x03` | 普通读 |
| Page Program | `0x02` | 页编程，最多 256 字节 |
| Sector Erase | `0x20` | 4KB 扇区擦除 |
| Block Erase 32KB | `0x52` | 32KB 块擦除 |
| Block Erase 64KB | `0xD8` | 64KB 块擦除 |
| Chip Erase | `0xC7` | 整片擦除 |
| JEDEC ID | `0x9F` | 读厂商/类型/容量 |
| Release Power Down | `0xAB` | 唤醒 |

### 6.3 状态寄存器

| 位 | 名称 | 说明 |
|---|---|---|
| bit0 | BUSY | 1 表示正在擦除/编程 |
| bit1 | WEL | 1 表示写使能已置位 |

每次页编程和擦除前都必须先发 `Write Enable`，并检查 WEL。

### 6.4 容量计算

JEDEC ID 的低字节是容量代码 `capacity_code`：

```c
g_w25qxx_capacity = 1UL << capacity_code;
```

例如：

| JEDEC ID | 容量代码 | 容量 |
|---|---|---|
| `EF 40 14` | 0x14 = 20 | 1MB（W25Q80） |
| `EF 40 15` | 0x15 = 21 | 2MB（W25Q16） |
| `EF 40 16` | 0x16 = 22 | 4MB（W25Q32） |
| `EF 40 17` | 0x17 = 23 | 8MB（W25Q64） |
| `EF 40 18` | 0x18 = 24 | 16MB（W25Q128） |

### 6.5 读数据流程

```text
CS 拉低
  → 发送 0x03
  → 发送 24 位地址
  → 连续接收数据
CS 拉高
```

`W25QXX_ReadData()` 内部按 4096 字节分块，避免单次 HAL 调用过长。

### 6.6 页写入流程

```text
写使能 0x06
  → CS 拉低
  → 发送 0x02
  → 发送 24 位地址
  → 发送数据（不超过本页剩余空间）
  → CS 拉高
  → 等待 BUSY 清零
```

`W25QXX_WriteData()` 会自动按 256 字节页边界拆分，跨页时重新发写使能和页编程命令。

### 6.7 擦除策略

`W25QXX_EraseRange()` 会优先选择大块擦除：

1. 地址 64KB 对齐且剩余 ≥64KB：`0xD8` 64KB 块擦除。
2. 地址 32KB 对齐且剩余 ≥32KB：`0x52` 32KB 块擦除。
3. 其他情况：按 4KB 扇区擦除。

擦除超时时间：

| 操作 | 代码超时 |
|---|---|
| 扇区擦除 | 3000ms |
| 32KB 块擦除 | 5000ms |
| 64KB 块擦除 | 10000ms |
| 整片擦除 | 300000ms |

实际芯片的整片擦除可能需要 25~200 秒，PC 端等待时间应足够。

### 6.8 写保护

如果写入后读回全是 `0xFF`，检查：

- 状态寄存器 BP0~BP2 是否设置了块保护。
- `WP` 引脚是否被拉低。
- 是否真的执行了擦除。
- 是否每次写入前都成功置位 WEL。

---

## 7. 命令协议

### 7.1 物理层

| 项目 | 参数 |
|---|---|
| USART1 | 115200，8 数据位，无校验，1 停止位 |
| USB CDC | 虚拟串口，参数同上（实际为 USB 批量传输） |
| 行结束符 | `\r\n`（固件只处理 `\n`，忽略 `\r`） |
| 数据字节 | 二进制原样传输，不做转义 |

> 固件解析规则：收到 `\n` 时认为一行命令结束；收到 `\r` 直接丢弃。这样可以避免 `\r\n` 中的 `\n` 被误当作写入数据的第一个字节。

### 7.2 命令列表

| 命令 | 格式 | 功能 |
|---|---|---|
| 识别 | `I` | 读取 JEDEC ID、容量、型号 |
| 擦除范围 | `S <addr> <len>` | 擦除指定范围 |
| 整片擦除 | `E` | 整片擦除 |
| 写入 | `W <addr> <len>` | 进入写入模式，随后发送 `len` 字节数据 |
| 读取 | `R <addr> <len>` | 读取 `len` 字节并返回 |

地址和长度支持十进制，也支持 `0x` 前缀的十六进制。

### 7.3 应答格式

#### 识别 `I`

```text
OK,<JEDEC_ID>,<CAPACITY>,<MODEL>\r\n
```

示例：

```text
OK,EF4017,8388608,W25Q64
```

失败：

```text
ERR,NOFLASH\r\n
```

#### 擦除范围 `S`

```text
OK,ERASED\r\n
```

错误：

```text
ERR,PARAM\r\n      参数错误
ERR,RANGE\r\n      地址/长度越界
ERR,ERASE\r\n      擦除失败
```

#### 整片擦除 `E`

```text
OK,ERASED\r\n
ERR,ERASE\r\n
```

#### 写入 `W`

PC 先发送命令行，固件回复：

```text
OK,READY\r\n
```

随后 PC 发送 `len` 字节二进制数据。固件写完后回复：

```text
OK,WROTE <len>\r\n
```

错误：

```text
ERR,PARAM\r\n
ERR,RANGE\r\n
ERR,RX\r\n         接收数据超时
ERR,WRITE\r\n      写入失败
```

#### 读取 `R`

固件先返回 `len` 字节二进制数据，再返回：

```text
OK,READ <len>\r\n
```

错误：

```text
ERR,PARAM\r\n
ERR,RANGE\r\n
ERR,READ\r\n
```

#### 未知命令

```text
ERR,CMD\r\n
```

### 7.4 协议时序示例

写入 256 字节到地址 0：

```text
PC  →  "W 0 256\r\n"
STM →  "OK,READY\r\n"
PC  →  <256 字节二进制数据>
STM →  "OK,WROTE 256\r\n"
```

读取 256 字节：

```text
PC  →  "R 0 256\r\n"
STM →  <256 字节二进制数据>
STM →  "OK,READ 256\r\n"
```

### 7.5 命令解析注意事项

- 命令行最大长度 `CMD_LINE_SIZE = 96`，超长会被丢弃。
- 写入数据长度只由命令行中的 `len` 决定，数据中的 `0x0D`、`0x0A` 都是普通数据。
- 写入过程中固件会阻塞在 `CMD_ReadExact()`，直到收满 `len` 字节或超时（每字节 2000ms）。
- 校验失败时，PC 会报告第一个不匹配的地址、期望值和实际值。

---

## 8. USB CDC 虚拟串口

### 8.1 为什么用 CDC

STM32F103C8T6 的原生 USB 是 Full Speed 设备接口，不支持 USB Host。接入 PC 时，最方便的方式是实现 **CDC ACM 虚拟串口**：

- Windows 10/11 自带 `usbser.sys`，免驱。
- PC 端按普通串口打开，无需写 USB 驱动。
- 带宽远高于 115200 波特率，适合大文件烧录。

### 8.2 USB 描述符

`Core/Src/usbd_desc.c` 中：

| 项目 | 值 |
|---|---|
| VID | `0x0483` |
| PID | `0x025B` |
| bDeviceClass | `0x02`（CDC） |
| bDeviceSubClass | `0x02`（ACM） |
| bDeviceProtocol | `0x00` |
| bMaxPacketSize0 | 64 |
| 厂商字符串 | `STMicroelectronics` |
| 产品字符串 | `STM32 Virtual ComPort` |
| 配置字符串 | `CDC Config` |
| 接口字符串 | `CDC Interface` |

> 本工程的 VID/PID 与可正常工作的参考固件一致，Windows 会直接绑定 `usbser`。

### 8.3 端点与 PMA

| 端点 | 地址 | 类型 | 包长 | PMA 地址 |
|---|---|---|---|---|
| EP0 OUT | `0x00` | Control | 64 | `0x40` |
| EP0 IN | `0x80` | Control | 64 | `0x80` |
| EP1 IN | `0x81` | Bulk | 64 | `0xC0` |
| EP1 OUT | `0x01` | Bulk | 64 | `0x110` |
| EP2 IN | `0x82` | Interrupt | 8 | `0x100` |

PMA 总大小 512 字节，上述分配互不重叠。BTABLE 位于 PMA 偏移 0，8 个端点占 `0x00~0x3F`，因此 EP0 缓冲从 `0x40` 开始是安全的。

### 8.4 初始化流程

`Core/Src/usb_device.c`：

```c
USBD_Init(&hUsbDeviceFS, &FS_Desc, 0U);
USBD_RegisterClass(&hUsbDeviceFS, &USBD_CDC);
USBD_CDC_RegisterInterface(&hUsbDeviceFS, &USBD_Interface_fops_FS);
USBD_Start(&hUsbDeviceFS);
```

`Core/Src/usbd_conf.c`：

1. 打开 GPIOA 和 USB 时钟。
2. PA11/PA12 配置为 `GPIO_MODE_AF_INPUT` + 上拉。
3. 配置 USB 中断 `USB_LP_CAN1_RX0_IRQn`。
4. `HAL_PCD_Init()`。
5. `HAL_PCDEx_PMAConfig()` 配置 5 个端点缓冲区。
6. 实现 `USBD_LL_*` 系列函数，把 PCD 回调转发给 USB 协议栈。

### 8.5 USB 中断

`Core/Src/stm32f1xx_it.c`：

```c
void USB_LP_CAN1_RX0_IRQHandler(void)
{
  HAL_PCD_IRQHandler(&hpcd_USB_FS);
}
```

### 8.6 堆空间

USB CDC 类句柄 `USBD_CDC_HandleTypeDef` 内部包含 `data[512]`，大约 544 字节，由 `USBD_malloc()`（即 `malloc`）分配。

因此启动文件中的堆必须足够：

```asm
Heap_Size      EQU     0x800
```

`.ioc` 中也应保持：

```text
ProjectManager.HeapSize=0x800
```

如果堆太小，`USBD_CDC_Init()` 中的 `malloc` 返回 `NULL`，`SET_CONFIGURATION` 会失败，设备可能能识别但无法打开串口。

### 8.7 收发数据流

#### USB 接收

```text
PC → USB OUT 端点
  → HAL_PCD_DataOutStageCallback
  → USBD_CDC_DataOut
  → CDC_Receive_FS
  → UART_RxPush
  → 命令环形缓冲
```

`CDC_Receive_FS` 会检查环形缓冲剩余空间：

- 空间足够：立即写入，并重新挂起 OUT 端点。
- 空间不足：暂不重新挂起，USB 主机会收到 NAK，从而自动等待，避免丢包。

#### USB 发送

```text
UART_Send()
  → CDC_IsReady()
  → CDC_Transmit_FS_Blocking()
  → USBD_CDC_TransmitPacket()
  → HAL_PCD_EP_Transmit()
  → PC
```

发送采用带超时的阻塞方式，完成后才返回。若 USB 未枚举成功或发送超时，`UART_Send()` 会自动回退到 USART1。

### 8.8 USB 与 UART 的选择

```c
if (CDC_IsReady() != 0U)
{
    if (CDC_Transmit_FS_Blocking(data, len, 200U) == USBD_OK)
    {
        return;
    }
}
/* 回退到 USART1 */
```

- USB 已连接并枚举成功：优先走 USB。
- USB 未连接或发送失败：走 USART1。
- 接收方向：USB 和 USART1 的数据都写入同一个环形缓冲，由同一个命令解析器处理。

### 8.9 USB 硬件要求

- PA11/PA12 直接连 USB D-/D+。
- PA12 需要 1.5kΩ 上拉到 3.3V。
- USB 线必须是数据线。
- 若使用自制板，注意 USB 差分线尽量等长、远离 SPI 和电源干扰。

---

## 9. PC 工具 Scom 使用

### 9.1 打开软件

运行：

```text
C:\Users\Administrator\Desktop\临时文件\scom\Scom.exe
```

### 9.2 连接设备

1. 用 USB 线连接 STM32 原生 USB 口。
2. 在设备管理器中确认出现 `USB 串行设备 (COMx)`。
3. 在 Scom 中选择对应 COM 口。
4. 点击连接/识别。
5. 日志中应显示 Flash 型号、JEDEC ID 和容量。

### 9.3 烧录流程

Scom 的完整流程：

```text
读取 Flash ID
  → 擦除目标范围
  → 写入文件数据
  → 读回校验
```

对应固件命令：

```text
I
S <addr> <len>
W <addr> <len> + 二进制数据
R <addr> <len>
```

### 9.4 日志示例

```text
检测到 Flash：Winbond W25Q64，ID=0xEF4017，容量=8.00 MB
开始擦除 0x0 ~ 0x100000（256 个扇区）
擦除完成。
开始写入：地址=0x0，长度=1048576 字节
写入完成。
开始读回：地址=0x0，长度=1048576 字节
读回完成。
===== 校验通过 =====
===== 烧写成功 =====
```

### 9.5 大文件传输说明

- PC 端每 16KB 为一个发送块。
- 固件端 USB 接收有流控，不会因为 PC 发送过快而丢数据。
- 写入速度主要受 Flash 页编程时间限制，W25Q64 典型值约 0.7~3ms/页。
- 擦除时间远大于写入时间，整片擦除请耐心等待。

---

## 10. 编译、烧录与测试

### 10.1 编译

1. 打开 `MDK-ARM\STM32C8T6_SPI.uvprojx`。
2. 选择目标 `STM32C8T6_SPI`。
3. 点击 `Rebuild`。
4. 确认输出：

```text
0 Error(s), 0 Warning(s)
```

### 10.2 烧录

使用 ST-Link：

1. 连接 SWDIO、SWCLK、3.3V、GND。
2. Keil 中选择 ST-Link Debugger。
3. 点击 Download。
4. 或使用 STM32CubeProgrammer 烧录：

```text
MDK-ARM\STM32C8T6_SPI\STM32C8T6_SPI.hex
```

### 10.3 上电测试

1. 拔掉 USB，重新上电。
2. 插入 USB 数据线。
3. 设备管理器出现 `USB 串行设备 (COMx)`。
4. 打开 Scom，选择 COM 口。
5. 发送 `I` 命令，确认返回 Flash 信息。

### 10.4 完整功能测试

建议按以下顺序测试：

1. **识别**：`I`，确认 ID 和容量。
2. **读取**：读取 256 字节，确认不超时。
3. **擦除**：擦除 4KB 扇区，读回应为全 `0xFF`。
4. **写入**：写入 256 字节测试图案。
5. **校验**：读回比较，确认一致。
6. **整片烧录**：选择实际固件文件，执行完整流程。

---

## 11. 常见问题与排查

### 11.1 PC 完全看不到 USB 设备

| 可能原因 | 排查方法 |
|---|---|
| USB 线是充电线 | 换一根确认能传数据的线 |
| PA12 没有 1.5k 上拉 | 检查 Blue Pill R10 或外加 1.5k 到 3.3V |
| USB 时钟不是 48MHz | 检查 HSE/PLL 配置，读 `RCC->CFGR` |
| MCU 卡在 `Error_Handler()` | 检查 HSE 模式是否为晶振 |
| USB 外设未使能 | 检查 `HAL_PCD_MODULE_ENABLED`、`MX_USB_DEVICE_Init()` |
| USB 中断未响应 | 检查 `USB_LP_CAN1_RX0_IRQHandler()` 是否链接 |

### 11.2 设备管理器提示“设备描述符请求失败”

这是最典型的 USB 枚举失败。优先检查：

1. **HSE 时钟模式**：8MHz 晶振板必须用 `RCC_HSE_ON`，不能用 `RCC_HSE_BYPASS`。
2. 程序是否卡在 `Error_Handler()`。
3. USB 48MHz 是否正常。
4. PMA 配置是否与端点匹配。
5. 堆是否足够（`Heap_Size >= 0x800`）。

> 本工程实际遇到过：工程配置为 HSE BYPASS，但板子是 8MHz 晶振，导致 MCU 在时钟初始化时死循环，USB 从未初始化，PC 报设备描述符请求失败。改为优先使用晶振后解决。

### 11.3 设备能识别，但打开串口失败

| 可能原因 | 处理 |
|---|---|
| 堆太小，CDC 句柄分配失败 | 把 `Heap_Size` 改为 `0x800` 或更大 |
| USB 描述符配置错误 | 用 USB 描述符查看工具检查 |
| 驱动异常 | 卸载设备后重新插拔，让 Windows 重新安装 usbser |
| 串口被其他软件占用 | 关闭其他串口工具 |

### 11.4 校验失败，地址 0 期望 0x00，实际 0x0A

这是命令解析把 `\r\n` 中的 `\n` 当成数据首字节导致的。

- 旧逻辑：收到 `\r` 就执行命令，环形缓冲里剩余的 `\n`（0x0A）被当作第一个数据字节。
- 结果：整包数据整体错位 1 字节。
- 修复：命令只在收到 `\n` 时执行，`\r` 直接忽略。

本工程已修复 `Core/Src/w25qxx_cmd.c` 中的 `W25QXX_CMD_Poll()`。

### 11.5 读回全是 0xFF

| 可能原因 | 处理 |
|---|---|
| 没有擦除 | 先执行擦除 |
| 写使能失败 | 检查 `Write Enable` 和 WEL |
| 写保护 | 检查状态寄存器 BP 位和 WP 引脚 |
| 页地址越界 | 检查写入地址和长度 |
| SPI 写时序问题 | 用逻辑分析仪抓 CS/CLK/MOSI |

### 11.6 读回全是 0x00

| 可能原因 | 处理 |
|---|---|
| MISO 被拉低 | 检查 PA6 是否短路到 GND |
| Flash 未供电 | 测量 VCC |
| CS 未拉低 | 检查 PA4 和片选逻辑 |
| SPI 模式错误 | 确认 CPOL=0、CPHA=0 |

### 11.7 读 JEDEC ID 返回 0xFFFFFF 或 0x000000

| 可能原因 | 处理 |
|---|---|
| MISO 悬空/未接 | 检查 PA6 到 Flash DO |
| CS 未连接 | 检查 PA4 到 Flash CS |
| Flash 未供电 | 测量 3.3V |
| SPI 时钟太快 | 降低分频 |
| Flash 处于掉电模式 | 发送 `0xAB` 唤醒 |

### 11.8 擦除超时

| 可能原因 | 处理 |
|---|---|
| 整片擦除时间不够 | 增大 PC 端和固件端超时 |
| Flash 进入保护状态 | 检查状态寄存器 |
| SPI 通信异常 | 抓波形确认命令和地址 |
| 电源不稳 | 检查 3.3V 纹波和电流能力 |

### 11.9 UART 乱码

| 可能原因 | 处理 |
|---|---|
| 系统时钟不是 72MHz | 检查 HSE/PLL |
| 波特率计算错误 | 检查 `USART1->BRR` |
| 共地不良 | 确保 STM32 和 USB-TTL 共地 |
| 使用了 5V TTL | 确认电平兼容 |

### 11.10 重新用 CubeMX 生成后 USB 又失效

CubeMX 可能覆盖以下内容：

- `SystemClock_Config()` 中的 USB 时钟和 HSE 模式
- `main.c` 中的 `MX_USB_DEVICE_Init()`
- 堆大小
- USB 中间件文件

重新生成后必须检查：

1. RCC HSE 是否为晶振模式。
2. USB Device / CDC 是否启用。
3. `Heap_Size` 是否 ≥ `0x800`。
4. `HAL_PCD_MODULE_ENABLED` 是否打开。
5. USB 中断是否加入。

---

## 12. 调试方法

### 12.1 用 ST-Link 读寄存器

重点寄存器：

| 寄存器 | 地址 | 检查内容 |
|---|---|---|
| `RCC->CR` | `0x40021000` | HSERDY、PLLRDY |
| `RCC->CFGR` | `0x40021004` | SW、USBPRE、PPRE1/2 |
| `USB->CNTR` | `0x40005C40` | PDWN、FRES、中断使能 |
| `USB->ISTR` | `0x40005C44` | 中断状态 |
| `USB->BTABLE` | `0x40005C50` | BTABLE 偏移 |

### 12.2 逻辑分析仪抓 SPI

建议同时抓：

- CS（PA4）
- CLK（PA5）
- MOSI（PA7）
- MISO（PA6）

检查：

1. CS 是否在命令期间保持低电平。
2. 命令码是否正确。
3. 24 位地址是否高位在前。
4. 读数据时 MISO 是否有响应。
5. 时钟频率是否过高。

### 12.3 串口打印调试

可以在关键位置通过 USART1 打印状态，例如：

```c
UART_SendString("CLOCK OK\r\n");
UART_SendString("USB INIT\r\n");
```

注意：USB 枚举成功后，`UART_SendString` 会优先走 USB，调试时可以先不插 USB。

### 12.4 USB 描述符查看

Windows 可使用 UsbTreeView、USB Device Tree Viewer 等工具查看：

- VID/PID
- 设备描述符
- 配置描述符
- 端点
- 枚举失败原因

### 12.5 对比法

如果 USB 不工作，可以烧录一个已知正常的固件（例如 `stm32f103c8t6wq25xxdownload.bin`）验证硬件：

- 正常固件能识别：硬件、USB 线、上拉、晶振都没问题，问题在软件。
- 正常固件也不能识别：优先检查硬件。

本工程就是通过对比正常固件的反汇编，发现其使用 `RCC_HSE_ON` 而不是 `RCC_HSE_BYPASS`，从而定位时钟问题。

---

## 13. 实验与扩展

### 实验 1：读取 Flash ID

目标：理解 JEDEC ID 和容量计算。

步骤：

1. 连接 Flash。
2. 发送 `I`。
3. 记录 ID、容量、型号。
4. 对照芯片丝印和数据手册。

### 实验 2：扇区擦除与读回

目标：理解擦除后数据为 `0xFF`。

步骤：

1. 读取 4KB 数据保存。
2. 执行 `S 0 4096`。
3. 读回，确认全 `0xFF`。

### 实验 3：页写入与跨页

目标：理解 256 字节页边界。

步骤：

1. 写入 300 字节到地址 0。
2. 读回比较。
3. 观察驱动如何拆分为 256 + 44 两次页编程。

### 实验 4：测速与优化

可优化方向：

- SPI 时钟提高到 36MHz（需确认 Flash 支持）。
- 使用 DMA 传输，减少 CPU 占用。
- 增大 USB 接收环形缓冲。
- 使用 `0x0B` Fast Read 提高读取速度。
- PC 端增大每次读写块大小。

### 实验 5：固件升级

把 W25QXX 作为外部固件存储：

1. PC 通过 USB 把固件写入 W25QXX。
2. STM32 从 W25QXX 读取固件。
3. 校验后跳转执行或搬运到内部 Flash。

### 实验 6：文件系统

在 W25QXX 上实现简单文件系统：

- 扇区管理
- 磨损均衡
- 坏块标记
- 掉电保护

---

## 14. 附录

### 附录 A 引脚分配

| 引脚 | 功能 | 说明 |
|---|---|---|
| PA4 | GPIO 输出 | W25QXX CS |
| PA5 | SPI1_SCK | 时钟 |
| PA6 | SPI1_MISO | 数据输入 |
| PA7 | SPI1_MOSI | 数据输出 |
| PA9 | USART1_TX | 串口发送 |
| PA10 | USART1_RX | 串口接收 |
| PA11 | USB_DM | USB D- |
| PA12 | USB_DP | USB D+ |
| PA13 | SWDIO | 调试数据 |
| PA14 | SWCLK | 调试时钟 |
| PD0 | OSC_IN | 8MHz 晶振输入 |
| PD1 | OSC_OUT | 8MHz 晶振输出 |

### 附录 B 命令速查

```text
I
S <addr> <len>
E
W <addr> <len>
R <addr> <len>
```

应答：

```text
OK,<id>,<capacity>,<model>
OK,ERASED
OK,READY
OK,WROTE <len>
OK,READ <len>
ERR,PARAM
ERR,RANGE
ERR,ERASE
ERR,RX
ERR,WRITE
ERR,READ
ERR,NOFLASH
ERR,CMD
```

### 附录 C 关键函数

| 文件 | 函数 | 说明 |
|---|---|---|
| `main.c` | `SystemClock_Config()` | 系统时钟和 USB 时钟 |
| `main.c` | `main()` | 初始化与主循环 |
| `usart.c` | `MX_USART1_UART_Init()` | USART1 初始化 |
| `usart.c` | `UART_Send()` | USB/UART 发送 |
| `usart.c` | `UART_RxPush()` | 接收数据入环形缓冲 |
| `usart.c` | `UART_RxPop()` | 从环形缓冲取数据 |
| `w25qxx.c` | `W25QXX_Init()` | Flash 唤醒和识别 |
| `w25qxx.c` | `W25QXX_ReadData()` | 读数据 |
| `w25qxx.c` | `W25QXX_WriteData()` | 页写入 |
| `w25qxx.c` | `W25QXX_EraseRange()` | 范围擦除 |
| `w25qxx.c` | `W25QXX_EraseChip()` | 整片擦除 |
| `w25qxx_cmd.c` | `W25QXX_CMD_Poll()` | 命令解析 |
| `usbd_conf.c` | `USBD_LL_Init()` | USB PCD 和 PMA 初始化 |
| `usbd_cdc_if.c` | `CDC_Receive_FS()` | USB 接收回调 |
| `usbd_cdc_if.c` | `CDC_Transmit_FS_Blocking()` | USB 阻塞发送 |
| `usbd_cdc_if.c` | `CDC_RxPoll()` | USB 接收流控 |

### 附录 D 参考资料

- STM32F103x8/xB 数据手册
- STM32F10xxx 参考手册（RM0008）
- W25QXX 数据手册
- USB 2.0 规范
- USB CDC ACM 类规范
- STM32CubeF1 固件库
  `C:\Users\Administrator\STM32Cube\Repository\STM32Cube_FW_F1_V1.8.7`

---

## 结语

本工程的核心难点不在 SPI 读写本身，而在：

1. **USB 枚举依赖精确的 48MHz 时钟**，HSE 模式必须与硬件一致。
2. **USB CDC 需要协议栈、描述符、PMA、堆和中断协同工作**。
3. **命令协议中行结束符和二进制数据必须严格区分**，否则会出现“数据整体错位 1 字节”的校验失败。

掌握这三点后，W25QXX 的读写、擦除和校验只是标准的 SPI 时序问题，调试起来会非常直接。
