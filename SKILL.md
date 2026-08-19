---
name: c-drive-cleanup
description: Windows C盘空间清理与维护。当用户说"清理C盘/C盘满了/释放C盘空间/C盘空间不足/磁盘清理/清理垃圾"时使用。自动诊断C盘占用、执行系统垃圾清理（临时文件/更新缓存/日志/回收站等19项）、迁移大文件到D盘、清理AI模型缓存、修改浏览器下载路径、DISM组件深度清理，全程保留系统文件与个人资料，输出清理报告。
---

# C盘清理 (c-drive-cleanup)

Windows 10/11 下全套 C 盘空间释放流程。2026-07-27 实战验证：单次操作将 C 盘可用空间从 0.15 GB 释放到 14.46 GB（净释放 ~14.3 GB）。

## 核心原则（必须遵守）

1. **只清理缓存/临时文件**，绝不触碰 `C:\Windows\System32`、`C:\Program Files` 软件目录
2. **迁移而非删除**个人文件（Downloads、桌面项目 → D 盘）
3. **先扫描 → 列清单 → 用户确认 → 分批执行**（每批 ≤10 项，逐批验证）
4. 删除前先确认文件确实无用（安装包、旧日志、不再使用的 AI 模型）
5. 操作前先检查 D 盘空间是否充足

## 工作流（按序执行）

### 第一步：诊断

```powershell
# C盘概况
$c = Get-PSDrive C
Write-Host "已用: $([math]::Round($c.Used/1GB,1)) GB / 可用: $([math]::Round($c.Free/1GB,1)) GB"
# D盘空间
Get-PSDrive D -ErrorAction SilentlyContinue
# 根目录大文件夹
Get-ChildItem "C:\" -Directory -ErrorAction SilentlyContinue | ForEach-Object {
    $size = (Get-ChildItem $_.FullName -Recurse -ErrorAction SilentlyContinue -File | Measure-Object Length -Sum).Sum
    [PSCustomObject]@{ Name = $_.Name; SizeGB = [math]::Round($size/1GB, 2) }
} | Sort-Object SizeGB -Descending
# 大文件 >200MB
Get-ChildItem "C:\" -Recurse -ErrorAction SilentlyContinue -File | Where-Object { $_.Length -gt 200MB } | Sort-Object Length -Descending | Select-Object -First 15
```

**诊断结论：**
- `Users` 目录最大 → 个人数据/AppData 占大头（执行第二步迁移、第四步应用清理）
- `Windows` 目录最大 → 系统组件/更新缓存（执行第二步脚本、第五步 DISM）
- 发现 `hiberfil.sys` → 可 `powercfg /h off`（失去休眠功能，需用户确认）

### 第二步：系统垃圾清理（CleanC_Garbage.bat）

脚本位置：`C:\Users\Administrator\WorkBuddy\2026-07-27-09-29-37\CleanC_Garbage.bat`
（若已迁移，可在桌面 zip「C盘清理工具包.zip」中找到）

```powershell
Start-Process -FilePath "C:\Users\Administrator\WorkBuddy\2026-07-27-09-29-37\CleanC_Garbage.bat" -Verb RunAs -Wait
```

脚本自动：提权 → 检测系统版本 → 19项清理（系统/用户临时文件、更新缓存、缩略图、回收站、日志、Prefetch、.NET临时、错误报告、INetCache、字体缓存、商店缓存、搜索索引、DISM备份、cleanmgr、Windows.old、还原点、组件残留）→ 汇总报告。

### 第三步：个人文件迁移到 D 盘

**Downloads 整体迁移：**

```powershell
$src = "$env:USERPROFILE\Downloads"; $dst = "D:\Downloads"
New-Item -Path $dst -ItemType Directory -Force | Out-Null
# 列出清单给用户确认后，分批迁移（每批≤10项）
Get-ChildItem $src | Select-Object -First 10 | ForEach-Object {
    Move-Item -Path $_.FullName -Destination $dst -Force
}
```

**桌面项目迁移：**（先列清单确认）

```powershell
$dst = "D:\Projects"; New-Item -Path $dst -ItemType Directory -Force | Out-Null
foreach ($proj in @("工作文件","workbuddy","dolibarr-develop")) {
    Move-Item "$env:USERPROFILE\Desktop\$proj" "$dst\$proj" -Force
}
```

⚠️ 迁移后桌面快捷方式失效，需告知用户从 D 盘新位置访问；被进程占用的目录重启后释放。

### 第四步：应用数据清理

**WorkBuddy 日志（隐藏大户，通常 3~9 GB，保留最近3天）：**

```powershell
$logDir = "$env:USERPROFILE\.workbuddy\logs"
Get-ChildItem $logDir -Directory | Where-Object { $_.Name -lt (Get-Date).AddDays(-3).ToString("yyyy-MM-dd") } |
    ForEach-Object { Remove-Item $_.FullName -Recurse -Force }
```

**AI 模型（不用则删，先确认）：**

```powershell
Remove-Item "$env:USERPROFILE\.cache\whisper\*.pt" -Force -ErrorAction SilentlyContinue   # Whisper语音模型 1~2GB
Remove-Item "$env:USERPROFILE\.cache\modelscope" -Recurse -Force -ErrorAction SilentlyContinue  # ModelScope 1~2GB
```

**浏览器缓存：**

```powershell
Remove-Item "$env:LOCALAPPDATA\Microsoft\Edge\User Data\Default\Cache\*" -Recurse -Force -ErrorAction SilentlyContinue
```

### 第五步：系统组件深度清理（DISM）

```powershell
dism /online /Cleanup-Image /StartComponentCleanup /ResetBase /Quiet
```

⚠️ `/ResetBase` 删除旧组件版本后无法回滚到旧版 Windows，执行前告知用户。耗时长（3-10分钟），建议后台运行，重启后完全生效。

### 第六步：浏览器下载路径改 D 盘（防止再次吃C盘）

Edge 默认下载到桌面，每次下载都在吃C盘：

```powershell
$prefFile = "$env:LOCALAPPDATA\Microsoft\Edge\User Data\Default\Preferences"
$content = Get-Content $prefFile -Raw
$content = $content -replace '"savefile":\{[^}]*"default_directory":"[^"]*"', '"savefile":{"default_directory":"D:\\Downloads","type":1}'
$content | Set-Content $prefFile -NoNewline -Encoding UTF8
```

### 第七步：验证与报告

```powershell
$cFree = (Get-PSDrive C).Free
Write-Host "C盘可用: $([math]::Round($cFree/1GB,2)) GB"
```

向用户输出：释放明细表、迁移对照表、当前可用空间、维护建议（每月跑脚本/每2-4周清日志/每周看空间）。

## 维护频率建议

| 项目 | 频率 |
|------|------|
| 运行 CleanC_Garbage.bat | 每月 1 次 |
| 清理 WorkBuddy 日志 | 每 2~4 周 1 次 |
| 迁移 Downloads 大文件 | 每月 1 次 |
| DISM 组件清理 | 每季度 1 次 |
| 空间预警：<2GB 紧急 / <5GB 关注 / <10GB 观察 | 每周检查 |

## 参考资料

- 完整方案文档：`C:\Users\Administrator\WorkBuddy\2026-07-27-09-29-37\C盘清理方案_完整版.md`
- 桌面工具包：`C:\Users\Administrator\Desktop\C盘清理工具包.zip`（含脚本+文档+清理记录）
