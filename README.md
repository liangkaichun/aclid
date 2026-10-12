# aclid — 插电合盖不休眠、屏幕立即熄，拔电合盖正常休眠

macOS LaunchDaemon，监听电源事件 + 合盖事件，按需切换 `pmset` 行为：

- 插电（AC Power）→ `pmset disablesleep 1`：合上盖子系统保持运行（本机实测该选项在插电时对合盖有效）
- 插电 + 合盖 → `pmset displaysleepnow` + 显示器睡眠后要求密码：屏幕立即熄灭并锁定，系统不睡
- 拔电（Battery Power）→ `pmset disablesleep 0`：恢复默认，合盖正常休眠

合盖锁屏依赖 macOS 的"显示器关闭后要求密码"开关（一次性设置，见下文"合盖锁屏设置"）。

## 文件清单

| 文件 | 用途 |
|---|---|
| `aclid` | 守护脚本（安装到 `/usr/local/bin/aclid`） |
| `com.aclid.daemon.plist` | launchd 配置（安装到 `/Library/LaunchDaemons/`） |

## 安装

### 安装工具：aclid-manager.app.zip，macos原生应用，下载后修改去掉.zip扩展名，直接运行。

以下命令同时适用于**全新安装**和**重装**（`bootout 2>/dev/null` 在未加载过时静默跳过，已加载时先卸载）：

```bash
cd /Users/kaichun/works/aclid
sudo cp aclid /usr/local/bin/aclid
sudo chmod 755 /usr/local/bin/aclid
sudo cp com.aclid.daemon.plist /Library/LaunchDaemons/
sudo launchctl bootout system/com.aclid.daemon 2>/dev/null
sudo launchctl bootstrap system /Library/LaunchDaemons/com.aclid.daemon.plist
```

合盖锁屏需另做一次性设置（见下节"合盖锁屏设置"）。

开机自启动由 plist 的 `RunAtLoad` + `KeepAlive` 保证；即使系统重启后 `disablesleep` 值被重置，守护进程每次启动时也会按当前电源重新设置。

## 验证

```bash
pgrep -fl aclid                            # 可看到多个 bash（主脚本 + 电源/日志子 shell），属正常
sudo pmset -g | grep -i SleepDisabled      # 插电应为 1
tail -f /var/log/aclid.log                 # 事件日志（Ctrl+C 退出）
```

## 合盖锁屏设置（一次性）

```bash
defaults -currentHost write com.apple.screensaver askForPassword -int 1
defaults -currentHost write com.apple.screensaver askForPasswordDelay -int 0
```

开盖后即停在锁屏界面。注意：此开关作用于**所有**"显示器睡眠后唤醒"场景（含闲置灭屏后回来），属 macOS 标准安全行为。撤销：

```bash
defaults -currentHost delete com.apple.screensaver askForPassword
defaults -currentHost delete com.apple.screensaver askForPasswordDelay
```

（或在 系统设置 → 锁定屏幕 里关闭"屏幕保护程序开始或显示器关闭后需要密码"。）

## 实测（建议装完后做一次）

1. 插上电源 → 日志出现 `on AC -> SleepDisabled=1`，此时合盖 → 日志出现 `lid closed, system awake -> displaysleepnow`，屏幕立即熄灭；开盖 → 停在锁屏界面，输入密码进入桌面（系统全程未睡眠）
2. 拔掉电源 → 日志出现 `on battery -> SleepDisabled=0`，合盖 → 正常休眠

## 卸载

```bash
sudo launchctl bootout system/com.aclid.daemon
sudo rm /Library/LaunchDaemons/com.aclid.daemon.plist
sudo rm /usr/local/bin/aclid
sudo rm /var/log/aclid.log     # 可选，清日志
```

## 原理

两条事件流，均非轮询：

1. **电源**：`pmset -g pslog` 在电源变化时输出一行，触发 `apply_state`：解析 `pmset -g ps` 当前电源，插电写 `disablesleep 1`、电池写 `disablesleep 0`。
2. **合盖**：`log stream --predicate 'process == "ContinuityCaptureAgent" AND eventMessage CONTAINS[c] "Clamshell"'` 订阅系统日志中的合盖事件（`Clamshell closed status changed from 0 to 1`，实测在 disablesleep=1 生效时也照常发出）。检测到合盖且当前 `SleepDisabled=1` 时执行 `pmset displaysleepnow`（只熄屏、不睡系统）。

进程结构：launchd → 主脚本 bash → `pmset pslog` + `while read` 子 shell（电源）+ `log stream` 子 shell（合盖），所以 `pgrep -fl aclid` 会看到多个 bash，属正常。

## 注意事项

- 合盖事件依赖系统日志里 `ContinuityCaptureAgent` 的事件行（本机实测可靠；若未来 macOS 版本移除该日志，合盖熄屏功能会失效，电源部分不受影响）。
- 低电量保护：`disablesleep 1` 时电池 <2% 系统仍可能强制睡眠（插电无此问题）。
- 若日志在插电状态下反复刷 `on AC`，是充电器反复协商（弱充电器常见），无害。
- 本工具只影响系统睡眠与熄屏；锁屏由"显示器关闭后要求密码"开关实现（见上），非本工具行为。
- 脚本修改后需重新执行安装步骤（`cp` 覆盖 + `bootout`/`bootstrap` 重载）。
