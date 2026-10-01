# 案例记录与技术说明

记录提供与发布：**Arrow36** · 整理日期：2026-10-02

本文依据 Arrow36 提供的 `FoloToy_AI_Passport_Rescue_Record.md` 整理。成功刷写日志和万用表测量描述来自该记录；本次文档整理没有重新刷写已经恢复的设备，也未独立复现每一项测量。

## 已有证据支持的过程

| 阶段 | 案例中的观察 |
| --- | --- |
| 硬件 | FoloToy AI Passport，记录中的板号为 `AIPASS_0001 VER: 1.0.0`；ESP32-C3，8 MB Flash |
| 触发 | 将模拟器测试固件刷到真机；该构建把 GPIO18/19 用作 UART |
| 正常启动 | Windows 报 USB 设备描述符请求失败，代码 43 |
| Boot/G 短接上电 | USB 重新枚举为 `303A:1001`，出现 CDC 串口和 JTAG 接口 |
| CDC 下载 | 网页工具与 esptool 仍然发生 `Write timeout` |
| 首次尝试 JTAG | OpenOCD 报 `LIBUSB_ERROR_NOT_FOUND`；系统中仅看到 WinUSB 名称仍不足以建立连接 |
| 驱动修复 | Zadig 2.9 对 `USB JTAG/serial debug unit (Interface 2)` 安装/替换 WinUSB |
| 最终刷写 | 乐鑫 OpenOCD 的 `program_esp ... 0x0 verify reset exit` 完成写入并报告 `Verify OK` |
| 恢复结果 | Arrow36 报告设备恢复正常；原记录说明屏幕和原生 USB 已恢复 |

成功日志报告芯片修订版 `v1.1`、Flash `8192 KB`。原始完整工具版本、当时固件 SHA-256 和完整日志文件没有随记录提供，因此不补写这些信息，也不把当前下载文件当作当时镜像的逐字节证明。

## GPIO2 与 UART0_BOOT：保留日志，区分推断

原记录提供了这段 ROM 日志：

```text
ESP-ROM:esp32c3-eco7-20230720
Build:Jul 20 2023
rst:0x15 (USB_UART_CHIP_RESET), boot:0x2 (UART0_BOOT)
Saved PC:0x40047ea4
wait uart0 download
```

这段记录可以作为“当时报告了 UART0_BOOT”的线索，但不单独证明完整的电气原因。原稿进一步断言 GPIO2 被 ES8311 拉低、PCB 没有上拉电阻，并因此“必定不能 USB 下载”；这些结论需要原理图、复位瞬间的实测电平、确切芯片/ROM 修订信息及采集记录共同支持。

核对时使用的 [ESP32-C3 数据手册 v2.4，§3.1、表 3-3](https://www.espressif.com/sites/default/files/documentation/esp32-c3_datasheet_en.pdf) 明确说明：GPIO2 实际上不决定 SPI Boot 与 Joint Download Boot 的选择，但为避免毛刺建议上拉。表 3-1 中 GPIO2 和 GPIO8 的默认配置均为浮空，GPIO9 才是弱上拉。这与原稿关于 GPIO8 默认弱上拉、GPIO2 低电平必然选择纯 UART 的概括不一致。

因此，本文**不把 GPIO2 的单一解释作为普遍结论，也不要求读者给 GPIO2 飞线或加上拉电阻**。本案例可复现的关键，是 USB 重新枚举后改用可正常工作的 JTAG 接口，而不是先完成 GPIO2 硬件改造。

官方 [esptool 启动模式说明](https://docs.espressif.com/projects/esptool/en/latest/esp32c3/advanced-topics/boot-mode-selection.html) 可用于核对 GPIO9、GPIO8 和启动日志。不同芯片/ROM、eFuse 配置以及实际连线仍应分别检查，不能仅凭 `Write timeout` 反推某一根引脚的电平。

## JTAG 并非无条件可用

USB CDC 与 USB JTAG 是同一 USB Serial/JTAG 控制器提供的不同功能。JTAG 刷写不依赖 esptool 的 CDC 下载握手，但依然依赖可用的 USB 物理连接、供电、USB 引脚和 JTAG 权限。

问题固件如果已经占用了 GPIO18/19，原生 USB JTAG 也可能不可达；本流程先通过 Boot/G 在上电时阻止该固件正常运行，不能省略这个前提。JTAG 被安全配置/eFuse 禁用、设备硬件损坏等情形，也不能保证按本文恢复。本文不包含修改或烧写 eFuse 的步骤。

资料：[USB Serial/JTAG 限制](https://docs.espressif.com/projects/esp-idf/en/v5.5.3/esp32c3/api-guides/usb-serial-jtag-console.html)、[JTAG 接口与禁用说明](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/api-guides/jtag-debugging/configure-other-jtag.html)。

## 测量与供电经验的边界

- 原记录称多组方形金色焊盘与 GND 导通。本指南据此提醒不要把装饰铜区当成 GPIO；其他板卡仍要在断电状态下测量确认，不能仅按外观套用。
- 原记录称无电池、只接 USB 时仍需按 `SW4` 才能开机。实用结论是确认设备真正上电；本文不据此断言所有版本的电源芯片内部工作方式，也不要求拆下电池。
- Boot/G 之外的焊点、芯片细脚和电源连接不属于本指南的操作范围。照片中的位置用于辅助辨认，板上明确丝印才是直接依据。

## 固件与数据

原记录以 `niulai-full.bin` 作为恢复镜像文件名。2026-10-02 文档整理时仅核对了 [官方接口](https://ai-passport.folotoy.cn/api/download/official/niulai-interactive-player) 的 GET 响应头：HTTP 200，`application/octet-stream`，文件名 `folo-ai-passport-niulai-v0.0.2-full.bin`，内容长度 1,424,752 字节；没有在本次整理中下载或刷入该文件。

原记录日志中的擦写长度为 1,425,408 字节。文件长度和擦写长度不能混为一谈，也不能由它们推断两个下载文件的哈希相同。

完整镜像从 `0x0` 写入，会覆盖镜像涉及的区域及其擦除扇区，可能包含分区表和 NVS。它不是“保留所有旧数据”的保证。分段刷写只有在分区布局兼容、组件来源明确、写入和擦除范围避开需要保留的数据时才适用；不能只把完整镜像的地址改成 `0x10000`。

## 发布与验证范围

本仓库整理了恢复步骤、故障分流、工具命令和成功日志摘录，并核对了官方资料及下载入口。文档中的命令经过 PowerShell 语法检查，但文档整理阶段没有执行驱动替换、芯片读写或再次救砖。

不公开设备私有二维码、SN/KEY 链接、账户密钥、Flash/NVS 转储、个人电脑路径或登录凭据。本仓库没有附带原始固件、第三方工具安装包或未经核实的原稿全文。
