# oxp3-steamos-fixes — ONEXPLAYER 3 (Intel Panther Lake) 在 SteamOS 上的修复包

版本 v1.0（2026-09-06） · MIT 许可 · 作者：<YOUR NAME HERE>

## 这是什么 / 修了什么

| # | 症状 | 根因 | 修复方式 |
|---|---|---|---|
| 1 | **游戏模式睡眠唤醒后“死机/黑屏”**（前灯亮、按键无声、Wi‑Fi/声音/手柄全失、只能硬关机） | NVMe **Predator GM7**（Biwin/Maxio `1dee:1602`）经 ACPI StorageD3 “simple suspend” 路径在 s2idle 恢复时 `nvme nvme0: Disabling device after reset failure: -19` → 根文件系统下线 | 内核参数 `nvme.noacpi=1`（`/etc/default/grub.d/oxp3-nvme.cfg` + `update-grub`）。仍可进入最深 S0ix，实测 2/2 成功 |
| 2 | 开机/切换会话/唤醒后**面板不亮**（Steam 音效正常） | gamescope 一开 HDR，xe 以 BT.2020/PQ 10bpc 输出，Samsung SDC **AMS881KB01-0** OLED 就黑屏 | 用户级 gamescope lua（`~/.config/gamescope/scripts/98-oxp3-oled-hdr.lua`）：像 Steam Deck OLED 一样保持面板 gamma2.2 模式、由 gamescope 内部做 HDR 色调映射 → **HDR 可用**且亮度滑块恢复 |
| 3 | 内核每帧刷 `xe … DSB 0 poll error` | xe Display State Buffer 在此面板上失败 | `/etc/modprobe.d/xe-oxp3.conf`：`options xe enable_dsb=0`（仅消除报错，可选） |
| 4 | **音量键快速连按会“卡住”**一直调节 | EC 经 i8042 发音量键但**丢失释放码**，内核认为一直按住 | `oxp3-volkey-fix.service`：开机在图形会话前独占 `AT Translated Set 2 keyboard`，创建无自动重复的虚拟键盘，把每次按键变为瞬时按下+释放 |
| 5 | 机身 **Home / Console / Keyboard** 键无功能；Steam 里手柄为普通 Xbox 360 | 这些键走 MCU 的键盘接口/厂商接口 | InputPlumber 复合设备（`/etc/inputplumber/devices.d/50-onexplayer_3.yaml` + 能力映射 `capability_maps.d/onexplayer_type3.yaml`）：Xbox→Steam 菜单、Console→快速访问、Keyboard→屏幕键盘、Home→Steam 菜单；Steam 看到的是一个内置 “Steam Deck 控制器” |

## 未修复（需上游）
- **背键 M1/M2**：xinput 模式下 MCU 不上报任何事件；“按键拦截”模式会让全部手柄键改走厂商接口并使手柄短暂失灵——请勿自行尝试。见 `issues/02-inputplumber.md`。
- **RGB 灯环**：内核 `hid-oxp` 的 LED 接口与 HHD/HueSync 的原始协议对 MCU 固件 1.55 均无效；**切勿解绑/重绑 hid-oxp（会触发内核 Oops）**。见 `issues/01-hid-oxp.md`。
- **陀螺仪**：BMI160 探测失败（`Error reading chip id`，-121）。见 `issues/03-bmi160.md`。
- 震动手感：两颗小 ERM + X360 模拟属硬件/协议固有；可调 `/sys/bus/hid/devices/0003:1A86:FE00.0003/rumble_intensity`（0–5）。

## 适用配置（已测试）
SteamOS 3.10 main build 20260827.1000，内核 7.2.0-valve1-1-neptune-72，ONEXPLAYER 3 BIOS 5.09（2026-08-10），面板 SDC AMS881KB01-0，SSD Predator GM7 1TB（fw BM345CVN）。脚本会检查机型（DMI）与 SteamOS，并在版本不同时提示；`nvme.noacpi=1` 只在检测到 GM7/1dee:1602 时应用（其它 SSD 用 `--force-nvme` 强制）。

## 安装
```bash
mkdir -p ~/oxp3-fix
cp oxp3-apply-fixes.sh oxp3-volkey-fix.py ~/oxp3-fix/   # oxp3-volkey-fix.py 也内嵌于脚本，可省略
chmod +x ~/oxp3-fix/oxp3-apply-fixes.sh
cp OXP3-*.desktop ~/Desktop/ && chmod +x ~/Desktop/OXP3-*.desktop   # 桌面双击图标（可选）
~/oxp3-fix/oxp3-apply-fixes.sh          # 需要输入 sudo 密码；确认后写入；提示需要重启时重启
```
- 先做 `--check`（不需 sudo、不改动）：`~/oxp3-fix/oxp3-apply-fixes.sh --check`
- 撤销全部：`~/oxp3-fix/oxp3-apply-fixes.sh --revert`（然后重启）
- 参数：`--yes` 免确认；`--force` 跳过机型/系统检查；`--force-nvme` 强制 NVMe 参数。
- 若 SteamOS 提示只读，脚本会执行 `steamos-readonly disable`（系统更新会自动恢复只读并可能覆盖 /etc 改动 → **更新后重新运行本脚本**）。

## 注意事项
- 先移除自己做过的同类 hack（如开机 `chvt` 切 VT 脚本、被 mask 的 `powerbuttond`），否则可能干扰。
- **不要**解绑/重绑 `hid-oxp`，**不要**向 `1a86:fe00` 的 hidraw 写“拦截”命令。
- 首次会话中启用 InputPlumber 后，Steam 的控制器页面可能要重启 inputplumber（`sudo systemctl restart inputplumber`）或重启 Steam 才显示控制器。
- 音量键修复会独占 i8042 键盘设备并以虚拟设备转发全部按键（含电源/其它键）；如接了外接 PS/2 类键盘请知悉。
- 一切改动位于 `/etc`、`/home`；`--revert` 可完全撤销。

## 发现过程（简述）
在游戏模式用 `rtcwake` 自动唤醒复现 7 次，均失败；KDE 对照成功。通过 SD 卡上的 `dmesg -w` 跟踪拿到恢复期日志，定位到 NVMe reset failure；用 `nvme.noacpi=1` 修复并两次验证 S0ix。开机黑屏由 gamescope HDR 引起，用已知面板 lua 关闭 PQ 输出解决。音量键用 `evtest` 抓到 EC 丢释放码；用 python-evdev 代理修复。手柄用 hidraw/UHID 报文抓取定位到各键的来源与 InputPlumber 的解码。

---

# oxp3-steamos-fixes — ONEXPLAYER 3 (Intel Panther Lake) fix pack for SteamOS

Version v1.0 (2026-09-06) · MIT · Author: <YOUR NAME HERE>

## What it fixes
1. **Game-mode suspend/resume "freeze"** (LED on, black/frozen screen, no sound, Wi‑Fi/controller dead, hard power-off needed): the Predator GM7 NVMe (Biwin/Maxio `1dee:1602`) fails to come back from s2idle via the ACPI StorageD3 path (`nvme nvme0: Disabling device after reset failure: -19`, root FS goes away). Fix: kernel parameter `nvme.noacpi=1` via `/etc/default/grub.d/oxp3-nvme.cfg` + `update-grub`. Deep S0ix is still reached.
2. **Panel stays black in game mode** (boot / session switch / resume): gamescope enables HDR and xe drives the Samsung AMS881KB01-0 OLED with BT.2020/PQ 10bpc, which the panel does not display. Fix: a gamescope known-display lua that keeps the panel in native gamma-2.2 mode and lets gamescope tone-map internally (Steam Deck OLED style) — HDR works and the brightness slider starts working.
3. Per-frame `xe … DSB 0 poll error` flood: `options xe enable_dsb=0` (cosmetic, optional).
4. **Volume keys "stick"** on quick presses: the EC drops key-release scancodes on the i8042 keyboard. Fix: a boot-time evdev forwarder (`oxp3-volkey-fix.service`) that grabs the raw device and re-emits each key as a press+release pulse on a virtual keyboard without autorepeat.
5. Chassis **Home / Console / Keyboard** keys and a single virtual controller in Steam: InputPlumber composite device + capability map (Xbox→Steam menu, Console→Quick Access, Keyboard→on-screen keyboard, Home→Steam menu). Steam sees a built-in "Steam Deck Controller".

## Not fixed (upstream needed) — see `issues/`
Back paddles M1/M2 (MCU silent in xinput mode; intercept mode breaks the controller), RGB rings (hid-oxp LED and HHD/HueSync raw protocols are ignored by MCU fw 1.55; **never unbind/rebind hid-oxp — kernel Oops**), gyro (BMI160 probe fails with -121). Rumble coarseness is inherent to two small ERM motors behind X360 emulation; only `rumble_intensity` (0–5) is tunable.

## Tested configuration
SteamOS 3.10 main 20260827.1000, kernel 7.2.0-valve1-1-neptune-72, OXP3 BIOS 5.09, panel SDC AMS881KB01-0, SSD Predator GM7 1TB. The script refuses to run on other DMI/OS (`--force` overrides) and only applies `nvme.noacpi=1` when a GM7 / 1dee:1602 SSD is present (`--force-nvme` overrides).

## Install / usage
```bash
mkdir -p ~/oxp3-fix && cp oxp3-apply-fixes.sh oxp3-volkey-fix.py ~/oxp3-fix/ && chmod +x ~/oxp3-fix/oxp3-apply-fixes.sh
~/oxp3-fix/oxp3-apply-fixes.sh --check     # dry check, no sudo
~/oxp3-fix/oxp3-apply-fixes.sh             # apply (asks sudo password, confirms, tells you if a reboot is needed)
~/oxp3-fix/oxp3-apply-fixes.sh --revert    # undo everything, then reboot
```
Flags: `--yes`, `--force`, `--force-nvme`. SteamOS updates may re-enable the read-only rootfs and reset `/etc` — re-run the script afterwards (it is idempotent). Remove earlier hacks (chvt boot scripts, masked powerbuttond) first. Do not unbind hid-oxp or write intercept commands to the 1a86:fe00 hidraw device.

## How it was found (short)
Reproduced 7/7 with `rtcwake`-timed suspends in game mode (KDE control succeeded); resume-time kernel logs captured with `dmesg -w` onto an SD card exposed the NVMe reset failure; `nvme.noacpi=1` fixed it (2/2, S0i2.x reached). Boot black screen traced to gamescope HDR; volume keys traced with `evtest` to EC dropping releases; controller keys traced with hidraw/UHID report captures and InputPlumber debug logs.
