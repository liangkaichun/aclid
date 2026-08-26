# aclid — 插电合盖不休眠，拔电合盖正常休眠

macOS LaunchDaemon：监听电源事件，按电源来源切换 `pmset disablesleep`。

- 插电（AC Power）→ `pmset disablesleep 1`：合上盖子系统保持运行（前提：本机实测该选项在插电时对合盖有效）
- 拔电（Battery Power）→ `pmset disablesleep 0`：恢复默认，合盖正常休眠

## 文件清单

| 文件 | 用途 |
|---|---|
| `aclid` | 守护脚本（安装到 `/usr/local/bin/aclid`） |
| `com.aclid.daemon.plist` | launchd 配置（安装到 `/Library/LaunchDaemons/`） |

## 安装

```bash
cd /Users/kaichun/works/aclid
sudo cp aclid /usr/local/bin/aclid
sudo chmod 755 /usr/local/bin/aclid
sudo cp com.aclid.daemon.plist /Library/LaunchDaemons/
sudo launchctl bootstrap system /Library/LaunchDaemons/com.aclid.daemon.plist
```

若提示 `Bootstrap failed: 5: Input/output error`（说明已加载过），先卸载再装：

```bash
sudo launchctl bootout system/com.aclid.daemon
sudo launchctl bootstrap system /Library/LaunchDaemons/com.aclid.daemon.plist
```

开机自启动由 plist 的 `RunAtLoad` + `KeepAlive` 保证；即使系统重启后 `disablesleep` 值被重置，守护进程每次启动时也会按当前电源重新设置。

## 验证

```bash
pgrep -fl aclid                            # 应看到主脚本 + 管道子 shell 两个 bash（正常）
sudo pmset -g | grep -i SleepDisabled      # 插电应为 1
tail -f /var/log/aclid.log                 # 事件日志（Ctrl+C 退出）
```

## 实测（建议装完后做一次）

1. 拔掉电源线 → 日志出现 `on battery -> SleepDisabled=0`
2. 插回电源 → 日志出现 `on AC -> SleepDisabled=1`
3. 插电合盖 → 系统不休眠；拔电合盖 → 正常休眠

## 卸载

```bash
sudo launchctl bootout system/com.aclid.daemon
sudo rm /Library/LaunchDaemons/com.aclid.daemon.plist
sudo rm /usr/local/bin/aclid
sudo rm /var/log/aclid.log     # 可选，清日志
```

## 原理

脚本核心是 `pmset -g pslog` 的事件流（电源变化时输出一行，免 sudo、非轮询），每一行触发一次 `apply_state`：解析 `pmset -g ps` 的当前电源来源，插电写 `disablesleep 1`、电池写 `disablesleep 0`。进程结构：launchd → 主脚本 bash →（管道）`pmset` + `while read` 子 shell，所以 `pgrep -fl aclid` 会看到两个 bash，属正常。

## 注意事项

- 低电量保护：`disablesleep 1` 时电池 <2% 系统仍可能强制睡眠（插电无此问题）。
- 若日志在插电状态下反复刷 `on AC`，是充电器反复协商（弱充电器常见），无害。
- 本工具只影响系统睡眠；显示器休眠、屏保、锁屏不受影响。
- 脚本修改后需重新执行安装步骤（`cp` 覆盖 + `bootout`/`bootstrap` 重载）。