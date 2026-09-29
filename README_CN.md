# NFC Run

[中文](README_CN.md) | [English](README.md)

这是一个面向 **ESP32-S3** 的 NFC/按键输入设备固件。它通过 PN532 读取 NFC 令牌，或从 6 个 GPIO 按键生成令牌，再将令牌上报至 Home Assistant 事件端点。项目还包括 WS2812 状态灯、UDP 日志转发和浏览器 Web OTA 升级。

## 功能

- 通过固定的 APDU SELECT AID 读取 **Android HCE** 手机或卡片，并上报返回的文本令牌。
- 对 **MIFARE Classic** 卡执行认证，读取第 1 扇区第 0 块，并上报其中的 16 字节文本载荷。
- 轮询 6 个低电平有效按键，以 30 ms 消抖上报 `K1`–`K6` 令牌。
- 异步将 JSON 发送到 Home Assistant 的 `/api/events/nfc_scanned` 事件端点。
- 在 `http://<设备IP>/` 提供 Web OTA 页面，并通过 `POST /update` 接收固件二进制文件。
- 同时输出串口日志，并转发至可配置的 UDP 日志接收端。
- 用板载或外接 WS2812 LED 指示启动、Wi-Fi、OTA 与错误状态。
- 使用双 OTA 分区，每个 OTA 应用分区大小为 3 MiB。

## 硬件与接线

当前提交的配置目标为 **ESP32-S3**。上电前请根据所用开发板确认所有 GPIO 分配。

| 功能 | GPIO / 设置 | 说明 |
| --- | --- | --- |
| PN532 SDA | GPIO 8 | I²C，启用内部上拉 |
| PN532 SCL | GPIO 9 | I²C，频率 100 kHz |
| PN532 地址 | `0x24` | 固件使用的 7 位 I²C 地址 |
| 状态灯 | GPIO 48 | 一个 WS2812/NeoPixel，亮度限制为 20/255 |
| 按键 K3 | GPIO 4 | 低电平有效，内部上拉 |
| 按键 K2 | GPIO 13 | 低电平有效，内部上拉 |
| 按键 K1 | GPIO 14 | 低电平有效，内部上拉 |
| 按键 K6 | GPIO 16 | 低电平有效，内部上拉 |
| 按键 K5 | GPIO 17 | 低电平有效，内部上拉 |
| 按键 K4 | GPIO 18 | 低电平有效，内部上拉 |

每个按键应在按下时将对应 GPIO 接至 GND。并非所有 ESP32-S3 开发板都可使用 GPIO 48，请先核对板卡原理图。

## 环境要求

- ESP-IDF **v5.3.5**（由 `dependencies.lock` 锁定）
- ESP32-S3 开发板
- 已切换到 I²C 模式的 PN532
- 设备可连接的 2.4 GHz Wi-Fi 网络
- 可访问的 Home Assistant 实例及 Long-Lived Access Token
- 可选：用于收集日志的 UDP 接收端

## 烧录前配置

当前项目将环境相关参数直接写在源码中，构建前请替换为自己的值：

| 文件 | 需要设置的参数 |
| --- | --- |
| `main/main.c` | `WIFI_SSID`、`WIFI_PASS` |
| `components/ha_client/ha_client.c` | Home Assistant 事件 URL、Bearer Token |
| `components/udp_logger/udp_logger.c` | UDP 日志接收端 IP 和端口 |
| `components/pn532_reader/pn532_reader.c` | PN532 引脚/地址、Android HCE AID、与所用卡片匹配的 MIFARE 密钥 |
| `components/button_reader/button_reader.c` | 按键 GPIO 及令牌映射 |
| `components/status_led/status_led.c` | WS2812 GPIO 与亮度 |

设备上报的 JSON 结构如下：

```json
{
  "device": "esp32s3_n8r2",
  "token": "<NFC-or-button-token>"
}
```

请在 Home Assistant 中创建自动化，监听 `nfc_scanned` 事件，并根据 `trigger.event.data.token` 执行相应操作。

## 构建、烧录与监视日志

打开 ESP-IDF 终端后执行：

```bash
cd NFC_Run
idf.py set-target esp32s3
idf.py build
idf.py -p <串口> flash monitor
```

使用 `Ctrl+]` 退出串口监视器。Wi-Fi 连接成功后，串口日志会显示设备获取到的 IP 地址。

## Web OTA 升级

1. 先用 `idf.py build` 构建固件。
2. 确保设备和浏览器位于同一网络。
3. 在浏览器打开 `http://<设备IP>/`。
4. 选择 `build/NFC_Run.bin`，然后开始上传。
5. 设备重启前请勿断电。

分区表包含两个 OTA 应用分区；升级成功后，固件会写入非活动分区，并在下一次启动时切换使用。

> 注意：当前 OTA 服务使用未鉴权的 HTTP。仅应在可信网络内使用；若要扩大部署范围，请添加认证和 TLS。

## 状态灯说明

| 状态 | 颜色 |
| --- | --- |
| 启动中 / 令牌上报后的空闲状态 | 黄色 |
| 正在连接 Wi-Fi | 蓝色 |
| Wi-Fi 已连接 | 绿色 |
| 正在 OTA 升级 | 紫色 |
| 初次 Wi-Fi 连接超时 | 红色 |

Wi-Fi 断开后会自动重连。即使首次等待 Wi-Fi 的 10 秒超时，OTA 服务和 UDP 日志也会启动。

## 项目结构

```text
main/
  main.c                         Wi-Fi 初始化与应用启动入口
components/
  pn532_reader/                  PN532 轮询、HCE 与 MIFARE Classic 支持
  button_reader/                 六按键轮询与消抖
  ha_client/                     Home Assistant 事件客户端
  web_ota/                       HTTP 固件升级服务
  udp_logger/                    UDP 日志转发
  status_led/                    WS2812 系统状态指示
partitions.csv                   Factory 与双 OTA 分区布局
```

## 安全注意事项

- 不要提交 Wi-Fi 密码、Home Assistant Token、卡片密钥或私有 IP 地址。已被提交过的凭据应立即轮换。
- 请为此设备使用权限最小化的独立 Home Assistant Token。
- 当前实现中的 OTA 流量和 Home Assistant 通信均为 HTTP。请将设备限制在可信局域网内，或自行加入 HTTPS 与认证。
- NFC/卡片内容和按键令牌会被记录并转发，应将它们视为敏感标识符。
- 仅连接和使用你有权管理的硬件与卡片。
