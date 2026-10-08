# AI Passport custom firmware

FoloToy `ai-passport`（ESP32-C3 / 8MB / 无 PSRAM / 240×320）的小智固件编译配方。
本仓只存补丁和 GitHub Actions workflow；编译时拉取上游源码再打补丁，不 vendored 官方仓。

## 基线

| 项 | 值 |
|---|---|
| 上游 | `https://github.com/78/xiaozhi-esp32.git` |
| 基线 | `main` @ `0d576d3d4c049c6f55eaf879725dc23e516511b4` |
| 板型 | `folotoy/ai-passport` |
| 版本 | `PROJECT_VER=9.9.9`（挡住官方 OTA 覆盖） |
| 镜像 | `espressif/idf:v6.1` |

## 相对官方固件新增的模型可见工具

| 工具 | 作用 |
|---|---|
| `self.audio.play_url` | 播放 URL 上的单声道 Ogg Opus，采样率 24000 Hz |
| `self.screen.show_image` | 全屏显示 URL 图片（PNG 走 LVGL，JPEG 走固件解码器） |
| `self.screen.hide_image` | 关闭全屏图片；按机身任意键也可以退出 |

音频必须是单声道 Ogg Opus、24000 Hz，否则设备静默不出声。图片下载不超过 128KB，解码后 RGB565 不超过 120KB。

## 产物

Actions 成功后，Release（`fw-<run_number>`）和 Artifacts 里都有 `merged-binary.bin`。

## 烧录

合并固件从 `0x0` 起烧。本板是 USB Serial/JTAG，插上 USB 即可。**NVS 会被清空，烧录后必须重新配网。**

```bash
python -m esptool --chip esp32c3 -p /dev/cu.usbmodem* -b 921600 \
  write_flash 0x0 merged-binary.bin
```

连接失败时把波特率降到 `-b 115200`。
