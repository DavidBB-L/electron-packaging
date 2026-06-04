# 📦 Electron macOS 打包指南 / Electron macOS Packaging Guide

> 基于真实项目（章台 ChangeTale）提炼的通用 Electron macOS 打包经验。
> Universal Electron macOS packaging knowledge distilled from a production project.

---

## ✨ 功能 / Features

| 功能 / Feature | 说明 / Description |
|---|---|
| 📋 **构建脚本模板** / Build Script Templates | `build.sh` + Mac `.command` 一键启动/打包 |
| 🔧 **asarUnpack 三道防线** / asarUnpack Guards | 白名单 + 构建拷贝 + 完整性校验 |
| 🪞 **国内镜像源** / China Mirror | npmmirror Electron 镜像配置 |
| 🔄 **版本漂移检测** / Version Drift Detection | 自动检测并重装过期 Electron |
| 🖥️ **Mac 窗口问题修复** / Mac Window Fixes | 双窗口竞态、拖动、红绿灯避让、启动画面 |
| ✅ **构建后验证清单** / Post-Build Checklist | 6 项检查确保打包成功 |

---

## 🚀 快速开始 / Quick Start

### 安装 / Install

**作为 Hermes Skill / As a Hermes Skill：**

```bash
git clone https://github.com/249695811/electron-packaging.git ~/.hermes/skills/electron-packaging
```

然后对 Hermes Agent 说 "打包 Electron" / Tell Hermes Agent: "package Electron app"

**手动参考 / Manual Reference：**

直接查看 [SKILL.md](SKILL.md) 获取完整的打包指南和踩坑记录。

---

## 📋 核心经验 / Core Knowledge

### 1. asarUnpack 漏文件 / Missing asarUnpack Files

打包后 `server.js` 从 `asar.unpacked/app/` 读文件。漏掉的文件留在 asar 包内 → **静默 404，dev 模式正常但打包后全挂。**

After packaging, `server.js` reads from `asar.unpacked/app/`. Missing files stay in the asar → **silent 404, works in dev but breaks in packaged app.**

**三道防线 / Three guards：**
1. `electron/package.json` 的 `asarUnpack` 数组包含所有静态文件
2. 构建脚本 step 3 显式 `cp` 这些文件
3. 构建末尾自动校验文件存在性

### 2. Mac 国内镜像源 / China Mirror (Required)

```bash
export ELECTRON_MIRROR="https://npmmirror.com/mirrors/electron/"
export ELECTRON_CUSTOM_DIR="{{ version }}"
```

**所有 Mac 相关操作必须设置。** 不设 = 二进制下载超时 = `Electron failed to install correctly`

Must be set for all Mac operations. Without it = binary download timeout = `Electron failed to install correctly`

### 3. Electron 版本漂移 / Version Drift

Mac `node_modules` 可能缓存旧版 Electron（如 28 vs 33），`package-lock.json` 锁定旧版本号。

Mac `node_modules` may cache old Electron versions (e.g. 28 vs 33), locked by `package-lock.json`.

**修复 / Fix：** 启动脚本检测版本 → 低于目标版本自动清掉重装。

### 4. terser 压缩 0 字节 / terser Empty Output

纯注释文件（如 preload.js）terser 压缩后变 0 字节 → `mv` 覆盖 → 文件丢失。

Pure comment files (e.g. preload.js) compress to 0 bytes → `mv` overwrites → file lost.

**修复 / Fix：** 压缩后 `[ -s ]` 验证非空才覆盖。

### 5. Mac 窗口问题 / Mac Window Issues

| 问题 / Issue | 症状 / Symptom | 修复 / Fix |
|---|---|---|
| 双窗口竞态 / Double window | Dock 两个窗口 | `_creatingWindow` 锁 + `activate` 三道判断 |
| 窗口拖不动 / Can't drag | 只能点红绿灯 | `-webkit-app-region: drag` on header |
| 红绿灯重合 / Traffic lights overlap | 按钮遮住 logo | `padding-left: 80px` on darwin |
| 启动白屏 / White flash on start | 窗口白一下 | `backgroundColor` + data URL splash |

---

## 📂 文件结构 / File Structure

```
electron-packaging/
├── SKILL.md           # Hermes Agent 技能文件 / Hermes skill file
├── README.md          # 本说明 / This file
├── .gitignore
└── references/
    └── (未来补充 / future additions)
```

---

## 📏 构建后验证 / Post-Build Verification

```bash
# 1. asarUnpack 完整性 / asarUnpack integrity
ls -la electron/app/server.js electron/app/index.html

# 2. preload.js 非空 / preload.js not empty (terser pitfall)
wc -c electron/preload.js  # must be > 0

# 3. 文件时间戳 / File timestamps
ls -lt electron/app/*.js | head -3

# 4. Mac Console 零报错 / Mac Console zero errors
# Cmd+Option+I → Console → check for 404/ReferenceError
```

---

## ⚠️ 常见坑速查 / Pitfall Quick Reference

| # | 坑 / Pitfall | 严重度 / Severity |
|---|---|---|
| 1 | asarUnpack 漏文件 → 打包后全挂 | 🔴 P0 |
| 2 | 新增文件未三处同步 | 🔴 P0 |
| 3 | Electron 版本漂移（旧 node_modules） | 🟡 P1 |
| 4 | 二进制安装失败（postinstall 静默失败） | 🟡 P1 |
| 5 | Mac 国内未设镜像源 → 超时 | 🟡 P1 |
| 6 | terser 压缩注释文件 → 0 字节 | 🟠 P2 |
| 7 | .bak 文件混入 git | 🟠 P2 |
| 8 | Mac 双窗口竞态 | 🔵 P3 |
| 9 | 窗口无法拖动（hiddenInset） | 🔵 P3 |
| 10 | main.js 路径解析（多布局不兼容） | 🔵 P4 |
| 11 | `require.main.filename` 打包后失效 | 🔵 P4 |

---

## 🔗 相关资源 / Related Resources

- [electron-builder 文档](https://www.electron.build/)
- [asarUnpack 配置](https://www.electron.build/configuration#asarunpack)
- [npmmirror Electron 镜像](https://npmmirror.com/mirrors/electron/)
- [Electron 官方打包指南](https://www.electronjs.org/docs/latest/tutorial/tutorial-packaging)

## 📜 许可证 / License

MIT
