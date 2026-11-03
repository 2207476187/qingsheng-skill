# /qingsheng-upgrade — 升级情圣技能

升级 qingsheng-skill 到最新版本。

## 触发方式

用户说 `/qingsheng-upgrade` 或"升级情圣技能"时执行本流程。

---

## 执行步骤

### Step 1：检测安装位置和类型

> **Windows / DSH 适配版**：安装目录固定为 `C:\Users\Administrator\.dsh\skills\qingsheng`，无需探测；上游原版在 `C:\Users\Administrator\.dsh\skills\_bak-upstream\qingsheng\`。

安装目录（唯一）：

```
C:\Users\Administrator\.dsh\skills\qingsheng\    ← DSH skills 目录（本机实际位置）
```

用 PowerShell 工具：
```powershell
$QS_HOME = "C:\Users\Administrator\.dsh\skills\qingsheng"
Test-Path "$QS_HOME\VERSION"   # 存在即已安装
Test-Path "$QS_HOME\.git"      # 本机是 vendored（无 .git），走 git clone 路径
```

### Step 2：读取当前版本

```powershell
$QS_HOME = "C:\Users\Administrator\.dsh\skills\qingsheng"
if (Test-Path "$QS_HOME\VERSION") { (Get-Content "$QS_HOME\VERSION" -Raw).Trim() } else { "unknown" }
```

### Step 3：检查远端最新版本

```powershell
try { (Invoke-RestMethod -Uri "https://raw.githubusercontent.com/tomwong001/qingsheng-skill/main/VERSION" -TimeoutSec 8).Trim() } catch { "FETCH_FAILED" }
```
（若 PowerShell 报证书错误，改用 `git ls-remote` 或 `git clone --depth 1` 到临时目录读 VERSION。）

如果当前版本 == 远端版本：告知用户已是最新，结束。

### Step 4：执行升级

**Git 管理的安装（有 .git 目录）**：
```powershell
git -C "C:\Users\Administrator\.dsh\skills\qingsheng" pull --ff-only
```

**Vendored 安装（无 .git，本机就是这种）**：
```powershell
$QS_HOME = "C:\Users\Administrator\.dsh\skills\qingsheng"
$TMP = Join-Path $env:TEMP ("qs-upgrade-" + [guid]::NewGuid().ToString('N').Substring(0,8))
git clone --depth 1 https://github.com/tomwong001/qingsheng-skill.git $TMP
# 只覆盖内容文件；覆盖后会冲掉 Windows 适配，需重新对照 _bak-upstream 套用
Copy-Item "$TMP\skill\*" $QS_HOME -Recurse -Force
Copy-Item "$TMP\VERSION" $QS_HOME -Force
if (Test-Path "$TMP\CHANGELOG.md") { Copy-Item "$TMP\CHANGELOG.md" $QS_HOME -Force }
Remove-Item -Recurse -Force $TMP
```

### Step 5：读取 CHANGELOG，汇报改动

```powershell
Get-Content "C:\Users\Administrator\.dsh\skills\qingsheng\CHANGELOG.md" -TotalCount 60
```

从 CHANGELOG 中提取本次升级的变更，用中文 3-5 条 bullet 告知用户。

### Step 6：更新版本记录

```powershell
$QS_HOME  = "C:\Users\Administrator\.dsh\skills\qingsheng"
$QS_STATE = "C:\Users\Administrator\AppData\Roaming\.qingsheng"
New-Item -ItemType Directory -Force -Path $QS_STATE | Out-Null
Copy-Item "$QS_HOME\VERSION" "$QS_STATE\installed-version" -Force
Set-Content "$QS_STATE\last-update-check" ([DateTimeOffset]::UtcNow.ToUnixTimeSeconds())
```

---

## 输出格式

```
✅ 情圣技能已升级：v{旧版本} → v{新版本}

本次更新：
• [变更1]
• [变更2]
• [变更3]
```

---

## 平台兼容性

- **本机（Windows / DSH）**：安装目录 `C:\Users\Administrator\.dsh\skills\qingsheng`，状态目录 `C:\Users\Administrator\AppData\Roaming\.qingsheng`，全部命令用 PowerShell
- 其他平台参考：macOS / Linux `~/.claude/skills/qingsheng-skill/`；Windows 原生（Claude Code）`$APPDATA/.claude/skills/qingsheng-skill/`

---

## 异常处理

| 情况 | 处理 |
|------|------|
| 找不到安装目录 | 从 `C:\Users\Administrator\.dsh\skills\_bak-upstream\qingsheng\` 恢复，或重新 `git clone` 到 `C:\Users\Administrator\.dsh\skills\qingsheng` |
| 拉取失败 / 无网络 | 告知网络问题，跳过（PowerShell 证书错误时改用 `git clone`） |
| git pull 有冲突 | `git stash` 后 `git pull --ff-only` |
| 版本文件损坏 | 用上游 `VERSION` 强制覆盖 |
