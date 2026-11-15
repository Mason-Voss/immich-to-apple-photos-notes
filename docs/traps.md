# 陷阱详解

本文记录在 Photos AppleScript 通道上实测到的行为。每条都标注了**可复现的症状**，便于后续排查。

---

## 1. 模态弹窗 = 240 秒全局阻塞

### 症状
- `osascript -e 'tell application "Photos" to count of albums'` → 超时（-1712 AppleEvent timed out）
- 但同时 `tell application "Photos" to get name` → 0.1 秒返回
- 屏幕上有一个「无法导入 N 个项目」的对话框

### 机理
Photos 在导入失败时弹出模态 sheet，主线程等待用户确认。此时它仍能回答"应用级属性"查询，但**任何需要访问图库模型的事件都会阻塞**，直到 AppleScript 的默认超时（约 240 秒）。

### 判别方法
```bash
# 能回答 = 应用活着；不回答 = 主线程被占
osascript -e 'tell application "Photos" to get name'
```
注意：`get name` **不能**判断它是否健康，只能判断进程活着。真正的健康判据是"图库类指令能否在 30 秒内返回"。

### 三层防御
1. **预检**（成本最低，覆盖 90%）
   ```bash
   sips -g pixelWidth -g pixelHeight file.jpg   # 出现 <nil> = 坏文件
   ffprobe -v error -select_streams v:0 -show_entries stream=codec_name -of csv=p=0 file.mp4
   ```
2. **规范化转码**：`sips -s format jpeg` 或 `ffmpeg -c:v libx264 -c:a aac`
3. **AX 兜底**：用辅助功能 API 找到弹窗的「好」按钮并点击

### 代价记录
单次阻塞 240 秒。230 次事件 ≈ 10–15 小时。101 次被迫重启 Photos。

---

## 2. WAL 膨胀

### 症状
`Photos.sqlite-wal` 从 MB 级涨到 60 GB；Photos 响应逐渐变慢；最终表现为"假死"。

### 检查
```bash
ls -lh "Photos Library.photoslibrary/database/Photos.sqlite-wal"
sqlite3 "file:...Photos.sqlite?mode=ro" "PRAGMA wal_checkpoint(PASSIVE);"
# 返回 1|14588277|42 = busy，1458 万待合并帧，只合并了 42 帧
```

### 机理
SQLite 的 WAL 只有在所有读者释放快照后才能被回收。`photolibraryd` 长期持有读连接，导致检查点无法推进。

### 处置
| 方法 | 效果 |
|---|---|
| `PRAGMA wal_checkpoint(PASSIVE)` | 无效（被读快照阻塞） |
| 杀 `photoanalysisd` / `mediaanalysisd` | 无效（不是它们持锁） |
| 杀 `photolibraryd` | 无效（launchd 立即重启并重新持锁） |
| **重启 Photos** | **有效**：60GB → 13GB → 644KB |

### 预防
每导入 5000 个或 WAL > 10GB 时主动重启一次 Photos。

---

## 3. `--stop-on-error` 静默不产报告

### 症状
批次返回非 0，且 `--report` 指定的 CSV **根本不存在**。

### 机理
osxphotos 在达到 `--stop-on-error` 阈值后中止，跳过报告写入。而账本依赖报告，于是"没有报告"被判定为"整批失败"，整批退回队列 → 无限重试同一批。

### 对策
- 阈值设大（我们改成 50）
- 更重要：**报告缺失时不能推断任何业务结论**，只能退回队列重试

---

## 4. TCC 三层与 launchd 无身份

### 症状
```
sqlite3 /Volumes/.../state.db
Error: unable to open database: authorization denied
```
同一条命令在终端/IDE 里执行正常，在 launchd 任务里失败。

### 机理
TCC 把权限绑定到**责任应用的身份**上，而不是文件或脚本。
launchd 启动的进程没有继承任何用户授权身份 → 访问外置卷、照片库均被拒。

### 三种身份的实际表现
| 启动者 | 外置盘 | Photos | 说明 |
|---|---|---|---|
| 终端 / IDE | ✅ | ✅ | 已授权 |
| launchd | ❌ | ❌ | 无身份 |
| .app 包 | ✅ | ✅ | 自有身份，可一次性授权 |

### 推论
**任何依赖 launchd 的 macOS 自动化，必须先用 `launchctl kickstart` 验证一次并看日志。** 我们那个任务跑了 330 次，全部静默失败。

---

## 5. 指纹去重与视频缺口

### 症状
视频被重复判定为"新文件"，导致重复导入。

### 机理
osxphotos 的 `--skip-dups` 比对 Photos 库中存储的指纹。**视频的指纹不一定被存储**。

### 对策
- 图片：指纹
- 视频：`(文件名, 字节数, 时长)` 三元组兜底
- 更彻底：导入前自算 SHA-256 存库

### 反面教训
文件名、大小、日期**单独任何一个**都不可靠：
- Immich 磁盘上是 UUID 名
- iOS 备份导出是时间戳名
- 不同照片可能同名同大小

---

## 附：其他观察

| 现象 | 说明 |
|---|---|
| `osxphotos` 的 `photosLibraryWaitForPhotos` 预检 | 在 4.6 万条以上的库上必然超时；需跳过 |
| 屏幕锁定 | Photos 不响应脚本；导入会挂起 |
| Live Photo | Immich 把 HEIC/MOV 拆成两条，导入后变成"图 + 视频" |
| 相册命名 | `--album "Immich/{filepath.parent.name}" --split-folder "/"` 可建层级相册 |
| 文件名 | 必须用硬链接暂存成原始名，否则库里全是 UUID |
