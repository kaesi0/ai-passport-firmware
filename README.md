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

## 版本号规则（硬要求）

**`PROJECT_VER` 必须等于 `9.9.<固件编号>`，固件编号就是 Release tag 里的 `fw-N` 的 N**（= GitHub Actions 的
`run_number`）。例如 tag `fw-25` 的固件，设备串口必须打印 `Ota: Current version: 9.9.25`。

- **为什么不能固定成 9.9.9**：那样无法从设备上确认当前跑的是哪一版——排查时必须先知道固件版本，
  否则「设备上到底是哪一版」只能靠读 flash 指纹反查。
- **怎么落地**：补丁 `0005` 只设**基线** `9.9.0`（保证任何本地/手动构建都不会被官方 OTA 覆盖）；
  发布构建由 workflow 的 `Stamp firmware version` 步骤 `sed` 注入 `9.9.${{ github.run_number }}`，
  并在 `Sanity checks` 里断言它确实等于 `9.9.<run_number>`（不等就直接失败，不产出固件）。
- **约束**：版本号必须 **≥ 云端版本**（本板云端是 `2.5.1`）且**单调不减**，否则会被官方 OTA 覆盖回 `2.5.1`。
  `9.9.N` 满足；N 增大时语义上仍是升序。
- **验证**：设备串口 `Ota: Current version:` 一行；`firmware/tools/preflight.sh` 会直接核对
  「文件名里的 N」与「设备上报的版本」是否一致。

## 补丁

按序号打，顺序不能乱。`0002` 依赖 `0001`，`0004` 依赖 `0002`，`0007` 依赖 `0002` 与 `0006`；`0003`/`0005`/`0008`/`0009`/`0010` 独立。

| 补丁 | 层级 | 内容 |
|---|---|---|
| `0001-audio-play-url-board-agnostic.patch` | 板无关 | `Application::PlayAudioUrl()`：进官方 `Notifying` 态播放网络音频，云端收尾打不断它。**本仓用的是 fw-19 修订**（见下） |
| `0002-ai-passport-audio-play-url.patch` | 板级 | 注册 `self.audio.play_url` |
| `0003-heap-stats-largest-free-block.patch` | 板无关 | 诊断：`SystemInfo` 多打一个 `largest block`（无 PSRAM 的板只看 free sram 会误判） |
| `0004-ai-passport-playback-screen-off.patch` | 板级 | 长音频播放期间熄屏省电；按键点亮 |
| `0005-ai-passport-project-ver.patch` | 版本 | 把上游 `PROJECT_VER`（`2.5.1`）抬成**基线 `9.9.0`**，防官方 OTA 覆盖；**发布构建的版本号由 workflow 注入**（见「版本号规则」） |
| `0006-fullscreen-image-layer-board-agnostic.patch` | **板无关** | 常驻全屏图片层 `ShowFullscreenImage/HideFullscreenImage`，`scale=0` 自动按面板尺寸等比铺满 |
| `0007-ai-passport-image-tools.patch` | 板级 | 图片工具 `show_test_pattern` / `show_image` / `hide_image` |
| `0008-notify-player-range-resume.patch` | 板无关 | `NotifyPlayer` 抗链路抖动：读超时 5s→10s；读失败/服务端提前关闭时用 `Range` 从**已消费字节**续传，喂同一个 `OggDemuxer`（不丢 OpusHead/解析状态） |
| `0010-audio-keep-radio-awake.patch` | 板无关 | 播放故事期间**不把射频降到 modem sleep**：`OnAudioChannelClosed` 里那句降档移到 `Notifying` 守卫内。真机实测降档后 109 秒出现 10 秒 stall → 丢段 |
| `0009-notify-player-resume-retry.patch` | 板无关 | 续传**带退避重试**：链路整体断掉时第一次重连必然失败，改为时间窗 60 s / 最多 20 次 / 每次失败等 1.5 s；失败与放弃都打日志 |

## 相对官方固件新增的模型可见工具

| 工具 | 作用 |
|---|---|
| `self.audio.play_url` | 播放 URL 上的单声道 Ogg Opus，采样率 24000 Hz |
| `self.screen.show_test_pattern` | 本机生成 120×160 彩色测试图案（不联网），用于自检屏幕链路 |
| `self.screen.show_image` | 全屏显示 `http://` 直链上的 **RGB565 原始像素**文件 |
| `self.screen.hide_image` | 关闭全屏图片；按机身任意键也可以退出 |

### 音频

必须是**单声道 Ogg Opus、24000 Hz**；立体声会被直接中止（不降混），本板无 MP3 解码器。
格式不符时设备**不报错、只是没有声音**。地址一律用明文 `http`：本板无 PSRAM，TLS 收包要
16,749 字节连续堆，播放中必然中途断流。

### 图片

图片走「原始像素直传」，设备**不做任何解码**，因此**不是** PNG/JPEG。文件格式（v1）：

```
偏移           内容
0      ..  N-11   RGB565 像素，w*h*2 字节，小端（LVGL 原生顺序）
N-10   ..  N-5    ASCII "R5G6B5"
N-4    ..  N-3    宽 w，小端 16 位
N-2    ..  N-1    高 h，小端 16 位
```

头放在**尾部**是刻意的：固件的 `LvglAllocatedImage` 析构会对数据指针做 `heap_caps_free`，
指针必须是 `malloc` 的基地址；头放尾部才能「下载块直接当像素块」，零拷贝、零第二缓冲。

- 地址必须是明文 `http://`（理由同音频；固件会明确拒绝 `https`）。
- 默认 120×160（38,400 字节），LVGL 自动放大 2 倍铺满 240×320。
- 显示前会用 `largest block`（最大连续空闲块）预检，不够时**返回可读错误**而不是黑屏。
- 显示期间屏幕保持常亮，最长 5 分钟后自动关闭；有 PSRAM 的板只要调大板级
  `kMaxImageBytes` 并用更大的源图档即可，显示层（`0006`）不需要改。

## 产物

Actions 成功后，Release（`fw-<run_number>`）和 Artifacts 里都有 `merged-binary.bin`。
Artifact 是 zip，**不是可烧录文件**；烧 Release 附件里的那份。

## 烧录

合并固件从 `0x0` 起烧，覆盖到 `0x743fff`，会涂掉 `0x9000` 起的 NVS。推荐用配方仓同级的
刷写向导/脚本（先备份 NVS，烧完写回，免重新配网）：

```bash
# firmware/bin/刷写固件.command（双击）或：
firmware/tools/flash_ai_passport.sh <merged-binary.bin>
```

手动等价命令（**会清空 NVS，烧完必须重新配网**）：

```bash
python -m esptool --chip esp32c3 -p /dev/cu.usbmodem* -b 921600 \
  write_flash --flash_mode dio --flash_freq 80m --flash_size 8MB 0x0 merged-binary.bin
# 最后那次 write_flash 若用了 --before no_reset，必须再跑一条 default-reset 的命令
# （如 read-mac），否则芯片会留在 flasher stub 里，串口一行不打印，很像刷坏了。
```

连接失败时把波特率降到 `-b 115200`。

## 复验

改补丁后必须在**全新干净副本**上从基线起逐份 `git am` 全部通过再推本仓：

```bash
BASE=0d576d3d4c049c6f55eaf879725dc23e516511b4
git clone https://github.com/78/xiaozhi-esp32.git fresh && cd fresh
git checkout "$BASE"
for p in ../patches/*.patch; do git am "$p" || exit 1; done
git diff --shortstat "$BASE"   # 0001(fw-19 修订)~0010 累计应为 10 files changed, 1021 insertions(+), 17 deletions(-)
```

不要手工编辑 `.patch`：在已打好依赖补丁的副本里改源码，再 `git format-patch` 导出。

## 关于 `0001` 的两个修订（fw-19 / fw-20）

`0001` 有两份修订，差别只在 `main/application.cc`（55 增 40 删）：

- **fw-19 修订（本仓当前使用）**：进 `Notifying` 态播放 + 播放期间云端句子不上屏。
- **fw-20 修订**：在 fw-19 之上再加三条 code review 修正——① `OnAudioChannelClosed`
  不再在故事播放期间把射频降到 `LOW_POWER`（降档改由 `StopNotification()` 在故事真结束时做）；
  ② `PlayAudioUrl` 遇到「旧下载只是在收尾」时直接失败、保留现状，不再「先停旧的再起不来」；
  ③ 收到 `https` 音频地址时在串口明确告警（TLS 要 16,749 字节连续堆，本板无 PSRAM 必断流）。

**这三条在真机上从未被验证过**（`固件版本说明.md` 里 fw-19、fw-20 都标着「待验证」），
而 fw-20 首次上机的实测结果是两次都没放完整集。虽然失败签名（HTTP 读/建连超时、
`mqtt_client: No PING_RESP`、同时在 Mac 上下载同一 OSS 对象只要 0.136 s / 5.7 MB/s）
指向**环境侧的设备网络退化**而非这三条改动，但按本项目的规矩「一次只改一处」，
图片这条线不携带任何未验证的音频改动：**本仓把 `0001` 固定在 fw-19 修订**。

fw-20 那三条要合回来，应该单开一轮：`0001` 换回 fw-20 修订 + 一次完整集回归，
不与图片同时上线。git 历史里仍在（tag `fw-20` / `fw-21`）。
