# 运行层

一个能跑的批处理脚本，和一个能长期活着的系统，是两件事。

迁移跑完之后，更难的问题是：**怎么让它在没有人盯着的情况下活几个月？**

---

## 拓扑

```
ImmichSync.app        （Swift，LSUIElement —— 只有菜单栏，没有 Dock 图标）
  └── guard_loop.sh    （直接子进程，写 guard.pid）
        └── watchdog.sh            每 5 分钟
              ├── 锁屏检查
              ├── Photos 存活探测   （30 秒超时）
              ├── Immich 可达性
              ├── 磁盘剩余 / WAL 体积
              ├── 写 status.json
              ├── 停滞检测 → 重启
              ├── 空闲时跑修复队列
              └── 发现新资产
```

## `status.json` 契约

App 从不直接和 Python 对话，两边只通过一个文件沟通：

```json
{
  "ts": "2026-11-16 23:57:17",
  "state": "idle",
  "pending": 0, "failed": 0, "imported": 48126,
  "photos_ok": true, "immich_ok": true,
  "disk_free_gb": 2492, "wal_gb": 0.0,
  "busy": 0, "batch_age_min": 0
}
```

把这个契约做“笨”，界面就能随便重写而不用碰引擎。

---

## 只有进了运行层才会遇到的五个坑

### 1. 分离的子进程会丢掉父进程的 TCC 身份

守护必须是 App 的**直接子进程**：

```swift
let p = Process()
p.executableURL = URL(fileURLWithPath: "/bin/bash")
p.arguments = [projectDir + "/guard_loop.sh"]
try p.run()          // 必须持有引用
```

如果改成 `nohup ... &`，子进程会被 launchd 收养，**失去对外置卷的访问权**——
同样的二进制、同样的用户，只是换了个父进程，权限就不一样了。

### 2. App Nap 会冻结一切，包括线程

没有窗口的后台 App 会被系统挂起。`Timer`、`DispatchSourceTimer`、
甚至连 `Thread.sleep` 的后台线程都会一起停摆。

解法：

```swift
ProcessInfo.processInfo.beginActivity(
    options: [.userInitiated, .idleSystemSleepDisabled, .suddenTerminationDisabled],
    reason: "sync supervisor")
```

### 3. `pgrep -f` 会匹配到它自己的命令行

```bash
/bin/bash -c "pgrep -f guard_loop.sh"   # 永远返回 0——它匹配到了自己
```

看护于是永远认为“已经在跑”，从不启动任何东西。
**不要用进程名做监管，写 PID 文件。**

### 4. PID 复用会让“是否存活”判断失效

重启之后 PID 会被复用。只看 `kill(pid, 0)`，会把别人的进程当成自己的守护，
于是**永远不启动**。必须再核对命令行：

```swift
guard kill(pid, 0) == 0 else { return false }
return runCapture("ps -p \(pid) -o command=").contains("guard_loop.sh")
```

### 5. 重建 App 会让授权悄悄失效

TCC 把授权绑在代码签名上。每次重新编译，临时签名就变了，
用户之前的批准不再适用——**不报错、不弹窗，就是没了**。

**铁律：编译 → 签名 → 授权 → 之后再也不重建。**
要改逻辑，改它调用的 shell / Python，不要动 App 本身。

---

## 锁屏

屏幕锁定期间，Photos 可能无视 AppleScript。天真的看护会把它当成「Photos 卡死」，
然后反复重启——永远重启不完。

**但「锁屏」这个信号本身不可靠。** `CGSSessionScreenIsLocked` 会被一次短暂的
屏幕休眠置位。把它当成前置条件，就会把一套完全健康的系统停掉——我们正好踩了这个坑：
App 停在 `state: locked`，而 Photos 响应只要 0.2 秒。

正确的顺序是反过来的：**先探测 Photos，锁屏只用来解释失败。**

```bash
if probe_photos; then
    PHOTOS_OK=true
elif [ "$LOCKED" = "yes" ]; then
    pause            # 真锁屏 + 真不应答
else
    restart_photos   # 因为别的原因不应答
fi
```

供参考，检查本身的写法：

```python
import Quartz
d = Quartz.CGSessionCopyCurrentDictionary()
locked = bool(d and d.get('CGSSessionScreenIsLocked'))
```

锁屏就写 `state: locked`、什么都不做、退出。解锁后自动恢复。

---

## 重启之后还有一个坑

**Photos 自己不会启动。** 登录项把 App 拉起来、App 拉起守护，
然后整条链报告 `state: error`——因为照片 App 根本没在跑。

只“探测”不够，必须“拉起”：

```bash
if pgrep -x Photos >/dev/null; then
    ... 探测 ...
else
    open -g -a Photos      # -g = 后台启动，不抢焦点
    sleep 20
    ... 再探测 ...
fi
```

---

## 权限（一次性，绑在 App 上）

| 授权 | 用途 |
|---|---|
| 完全磁盘访问（或可移除宗卷） | 读写外置照片库 |
| 照片 | 向 Photos 发送 Apple 事件 |
| 登录项（SMAppService） | 开机自启 |

`SMAppService.mainApp.register()` 可以程序化注册登录项，用户只需在系统设置里确认一次。

---

## 这一层买到了什么，买不到什么

**买到了：** 重启后自启、守护死了自动重启、锁屏自动暂停、出错弹通知、状态可查。
**买不到：** 适配一个改了 AppleScript 的 macOS 大版本，或一个改了 schema 的 Immich 版本。

所谓长期可靠，指的是 *“坏了会说，而且修起来很小”*，不是 *“永远不坏”*。
