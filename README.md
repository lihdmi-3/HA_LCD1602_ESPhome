# LCD1602 Display - ESPHome 项目说明

这是一个基于 ESPHome 的 LCD1602 显示器项目，使用 Wemos D1 Mini (ESP8266) 开发板，通过 PCF8574 I2C 模块驱动 16x2 字符型 LCD，并连接到 Home Assistant 实现自定义数据显示和远程控制。

## 硬件连接

| 组件       | 引脚       | 说明                     |
|------------|------------|--------------------------|
| D1 Mini    | GPIO4 (D2) | I2C SDA（接 PCF8574 SDA）|
| D1 Mini    | GPIO5 (D1) | I2C SCL（接 PCF8574 SCL）|
| D1 Mini    | GPIO2 (D4) | 背光控制（高电平点亮）   |
| PCF8574    | 默认 I2C 地址 0x27        |

**注意**：部分 PCF8574 模块地址可能为 0x3F，需根据实际情况修改配置中的 `address`。

## 功能特性

- **5 个可自定义显示页面**（Page0 ~ Page4），每个页面显示两行数据。
- **每行可独立选择显示内容**，共支持 24 种数据源（包括本地传感器和 Home Assistant 实体）。
- **自动轮播页面**（间隔可调，1~30 秒）或 **固定页面**。
- **背光控制**：可通过按钮或开关在 Home Assistant 中开启/关闭。
- **重启计数器**：记录设备重启次数（掉电保存）。
- **累计运行时间**：记录设备总运行时间（秒/小时），断电重启后继续累加。
- **Web 管理界面**：设备自带 Web 服务器（端口 80），可在浏览器直接配置。

## 支持的数据源

| 索引 | 显示内容               | 对应实体/传感器         |
|------|------------------------|-------------------------|
| 0    | 今日电量 (kWh)         | `sensor.tasmota_energy_today` |
| 1    | 今日燃气 (m³)          | `sensor.jin_ri_ran_qi_liang` |
| 2    | 当前功率 (W)           | `sensor.tasmota_energy_power` |
| 3    | WiFi质量 (%)           | 本地计算                |
| 4    | WiFi信号 (dBm)         | 本地 `wifi_signal`      |
| 5    | 剩余内存 (KB)          | 本地 `free_memory`      |
| 6    | 内存占用 (%)           | 本地 `memory_usage`     |
| 7    | 设备运行时间 (h)       | 本地 `uptime`           |
| 8    | 重启次数 (次)          | 本地 `restart_count`    |
| 9    | Pi运行时间 (d)         | `sensor.raspberrypi_uptime` |
| 10   | DNS查询数              | `sensor.pi_hole_dns_queries_today` |
| 11   | DNS拦截数              | `sensor.pi_hole_ads_blocked_today` |
| 12   | 系统时间               | Home Assistant 时间     |
| 13   | 拓竹剩余时间           | `sensor.a1_..._remaining_time` |
| 14   | 拓竹总使用时间         | `sensor.a1_..._total_usage` |
| 15   | 拓竹打印进度           | `sensor.a1_..._print_progress` |
| 16   | 温度 (°C)              | `sensor.xiaomi_..._temperature` |
| 17   | 湿度 (%)               | `sensor.xiaomi_..._humidity` |
| 18   | 甲醛浓度 (mg/m³)       | `sensor.xiaomi_..._hcho_density` |
| 19   | ES6剩余电量 (%)        | `sensor.soc`            |
| 20   | ES6实估续航 (km)       | `sensor.remaining_actual_range` |
| 21   | ES6累计里程 (km)       | `sensor.mileage`        |
| 22   | 路由器下载 (kbit/s)    | `sensor.xiao_ai_..._download_speed` |
| 23   | 路由器上传 (kbit/s)    | `sensor.xiao_ai_..._upload_speed` |

**说明**：以上实体名称需根据你的 Home Assistant 实际配置修改。本地传感器已由 ESPHome 自动生成。

## 累计运行时间原理

累计运行时间（`LCD Total Uptime`）通过两个全局变量实现断电记忆：

- `total_uptime_base`：已保存的累计秒数（整数，`restore_value: true`，掉电保存）。
- `last_merged_uptime`：上次合并时的 `uptime_sec` 值（不保存，启动时重置）。

工作流程：
1. 设备启动时，`last_merged_uptime` 被设置为当前 `uptime_sec`（整数秒）。
2. 每隔 10 分钟，计算 `delta = current_uptime - last_merged_uptime`，将 `delta` 累加到 `total_uptime_base`，并更新 `last_merged_uptime`。
3. 传感器 `total_uptime_sec` 实时返回 `total_uptime_base + (current_uptime - last_merged_uptime)`，确保显示值连续。
4. 断电重启后，`total_uptime_base` 恢复，`last_merged_uptime` 重置为当前 uptime，继续累加。

这种方式既保证了断电不丢失数据，又减少了频繁写入闪存。

## 在 Home Assistant 中的控制实体

项目会向 Home Assistant 注册以下实体：

### 传感器（Sensors）
- `sensor.lcd_device_uptime`：本次运行时间（小时）
- `sensor.lcd_wifi_quality`：WiFi 信号质量（%）
- `sensor.lcd_wifi_rssi`：WiFi 信号强度（dBm）
- `sensor.lcd_free_memory`：剩余内存（KB）
- `sensor.lcd_memory_usage`：内存使用率（%）
- `sensor.lcd_restart_count`：重启次数
- `sensor.lcd_total_uptime_seconds`：累计运行时间（秒）
- `sensor.lcd_total_uptime`：累计运行时间（小时）

### 选择器（Select）
- `select.lcd_refresh_interval`：自动翻页间隔（1~30 秒）
- `select.p0_line_0` ~ `select.p4_line_1`：每个页面的每行内容选择（共 10 个）

### 按钮（Button）
- `button.lcd_auto_switch_page`：切换为自动轮播模式
- `button.lcd_fixed_page0` ~ `button.lcd_fixed_page4`：固定到指定页面
- `button.lcd_backlight_on` / `button.lcd_backlight_off`：手动开关背光
- `button.lcd_restart`：重启设备
- `button.lcd_refresh_display`：强制刷新屏幕

### 开关（Switch）
- `switch.lcd_backlight`：背光总开关（可恢复状态）

## 配置修改指南

1. **修改实体 ID**：在 `sensor:` 部分，将所有 `entity_id` 替换为你 Home Assistant 中对应的实体。
2. **调整 I2C 地址**：如果 LCD 不显示，尝试将 `address: 0x27` 改为 `0x3F`。
3. **背光极性**：若背光逻辑相反，将 `inverted: false` 改为 `true`。
4. **WiFi 凭据**：在 `secrets.yaml` 中定义 `wifi_ssid` 和 `wifi_password`。

## 使用示例

- 在 Home Assistant 的“开发者工具”→“状态”中，可查看所有传感器值。
- 在“概览”中添加实体卡片，即可实时监控数据。
- 通过选择器更改每行显示内容，屏幕会立即更新（需等待刷新间隔或手动点击刷新按钮）。

## 注意事项

- 确保 ESPHome 版本 >= 2023.12，以支持 `restore_mode` 等新特性。
- 首次烧录后，需在 Home Assistant 中集成该设备（通过 API）。
- 累计运行时间首次启动时基数为 0，之后会正常累加。
- 若需更频繁的累计保存，可修改 `interval: 10min` 为更短时间，但会增加闪存写入次数。

## 许可证

MIT License

---

如有问题或建议，欢迎提交 Issue 或 Pull Request。

![7AE92448F11D81630A105696CDEE3FF1](https://github.com/user-attachments/assets/cc67d5eb-7e99-4131-a3e0-8c934d46cfb0)
![8E6E91331081EA48E380D72EFD015C5C](https://github.com/user-attachments/assets/a1146929-4fbb-4faa-91ad-dd7424edda8b)
