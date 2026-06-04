---
name: electron-packaging
description: Package Electron apps for macOS DMG distribution — build scripts, asarUnpack pitfalls, Mac mirror sources, version drift, binary installation, splash screens, race conditions, and post-build verification. Use when packaging, building, or distributing Electron desktop apps.
category: software-development
---

# Electron macOS 打包（DMG 分发）

从真实项目（章台 ChangeTale）提炼的通用 Electron 打包经验。覆盖构建脚本、asar 坑、Mac 国内镜像、版本漂移、二进制安装、启动画面、竞态修复、构建后验证。

## 触发条件

- 打包 Electron 应用为 DMG
- 配置 asarUnpack / build 脚本
- Mac 上 Electron 启动失败/二进制缺失
- 需要构建后验证清单
- BB 说"打包""构建""DMG"

## 核心架构：源码 ↔ 构建目录分离

```
源码目录（改代码改这里）          构建目录（产物，不要手动改）
my-app/                        my-app-dist/
├── electron/                  ├── electron/
│   ├── main.js                │   ├── main.js (terser压缩)
│   ├── preload.js             │   ├── preload.js
│   └── package.json           │   └── package.json
├── app.js                     ├── app/
├── server.js                  │   ├── app.js
├── index.html                 │   ├── server.js
└── ...                        │   ├── index.html
                               │   └── ...
                               ├── 一键启动测试.command
                               └── 一键打包_Mac.command
```

**铁律：先改源码，再同步到构建目录。绝对不要反过来。**

## asar 打包后文件位置

| 源文件 | 打包后位置 | 是否 asarUnpack |
|--------|-----------|----------------|
| `electron/main.js` | `app.asar/` 内 | ❌ 不需要 |
| `electron/preload.js` | `app.asar/` 内 | ❌ 不需要 |
| `app.js` / `server.js` / `index.html` | `app.asar.unpacked/app/` | ✅ 必须 |
| `assets/**` | `app.asar.unpacked/app/assets/` | ✅ 必须 |

**规则：需要从文件系统读取的文件必须进 asarUnpack，否则 server.js 的 `fs.readFileSync` 找不到。**

## ⚠️ 已知坑（按严重度排序）

### P0：打包后全部瘫痪

#### 1. asarUnpack 漏文件 → 静默 404

**症状**：dev 模式正常，打包 DMG 后页面白屏或功能全挂。

**根因**：`electron/package.json` 的 `asarUnpack` 数组漏了文件。server.js 从 `asar.unpacked/app/` 读文件，漏掉的留在 asar 包内找不到。

**三道防线**：
1. `electron/package.json` asarUnpack 白名单 — 必须包含所有 `index.html` 中 `<script src="/...">` 引用的文件
2. 构建脚本 step 3 必须 `cp` 这些文件到构建目录
3. 构建末尾自动校验（检查文件存在性）

**排查**：Mac Console 报 `Failed to load resource: 404` + `xxx is not defined`

**容易漏的文件**：`mobile.js`、`mobile.css`、`assets/**`、`i18n.js`、`templates.json`

#### 2. 新增文件必须三处同步

任何新静态文件被 `index.html` 引用或 `server.js` serve 时，必须同时更新：
1. `electron/package.json` asarUnpack
2. 构建脚本复制步骤
3. 构建目录手动同步

**漏任何一处 → 打包后 404 → 全功能瘫痪**

### P1：Mac 启动失败

#### 3. Electron 版本漂移

**症状**：Mac 打包日志显示旧版 Electron（如 28.0.0 而非 33）。

**根因**：`node_modules` 缓存旧版 Electron，`package-lock.json` 锁定旧版本号。

**修复**：启动脚本检测版本号 < 目标版本 → 自动 `rm -rf node_modules package-lock.json` 后重装。

```bash
# 版本检测示例
ELECTRON_VER=$(node -e "console.log(require('./node_modules/electron/package.json').version)")
TARGET_VER="33.0.0"
if [ "$(printf '%s\n' "$TARGET_VER" "$ELECTRON_VER" | sort -V | head -1)" != "$TARGET_VER" ]; then
    echo "Electron version $ELECTRON_VER < $TARGET_VER, reinstalling..."
    rm -rf node_modules package-lock.json
    npm install
fi
```

#### 4. Electron 二进制安装失败

**症状**：`npm install` 完成但 `npm start` 报 `Electron failed to install correctly`。

**根因**：postinstall 脚本下载平台二进制时静默失败（网络/代理/权限）。

**检测**：检查 `node_modules/electron/path.txt` 是否存在。

**修复**：
```bash
if [ ! -f node_modules/electron/path.txt ]; then
    echo "Electron binary missing, installing..."
    cd node_modules/electron && node install.js
fi
```

#### 5. Mac 国内镜像源（必须）

GitHub 的 Electron 二进制下载在国内超时。

```bash
export ELECTRON_MIRROR="https://npmmirror.com/mirrors/electron/"
export ELECTRON_CUSTOM_DIR="{{ version }}"
```

**所有 Mac 相关操作必须设置这两个环境变量。** 启动脚本、打包脚本都要带。

### P2：构建产物问题

#### 6. terser 压缩纯注释文件 → 0 字节

**症状**：preload.js 只有注释（无实际代码），terser 压缩后变成 0 字节空文件。

**修复**：构建脚本中压缩后验证非空才覆盖：
```bash
for f in main.js preload.js; do
    npx terser "$f" -o "${f}.min" -c -m 2>/dev/null
    if [ -s "${f}.min" ]; then
        mv "${f}.min" "$f"
    else
        echo "WARNING: $f compressed to empty, keeping original"
        rm -f "${f}.min"
    fi
done
```

#### 7. .bak 文件混入 git

**修复**：`.gitignore` 必须包含：
```
*.bak.*
*.bak
```

### P3：Mac 窗口问题

#### 8. 双窗口竞态

**症状**：Dock 出现两个相同窗口，或窗口不自动显示。

**根因**：`createWindow()` 是 async，macOS `whenReady` 和 `activate` 同时触发。

**修复**：
```js
let _creatingWindow = false;

async function createWindow() {
    if (_creatingWindow || mainWindow) return;
    _creatingWindow = true;
    // ... await initialization ...
    mainWindow = new BrowserWindow({ ... });
    _creatingWindow = false;
}

app.on('activate', () => {
    if (mainWindow) {
        if (mainWindow.isMinimized()) mainWindow.restore();
        mainWindow.show();
        mainWindow.focus();
        return;
    }
    if (_creatingWindow) return;
    createWindow();
});
```

#### 9. 窗口无法拖动（hiddenInset）

**症状**：`titleBarStyle: 'hiddenInset'` 后窗口只能点红绿灯，拖不动。

**修复**：
```css
.app-header { -webkit-app-region: drag; }
.app-header button, .app-header input, .app-header select {
    -webkit-app-region: no-drag;
}
```

#### 10. 启动画面（Splash Screen）

**问题**：Electron 启动慢，用户看到空窗口以为卡死。

**方案**：创建窗口 → 立刻 show → 加载 data URL 启动画面 → 服务就绪后 loadURL 真实页面。

```js
mainWindow = new BrowserWindow({
    width: 800, height: 600,
    show: true,
    backgroundColor: '#041c1c'  // 防白闪
});
mainWindow.loadURL(`data:text/html;charset=utf-8,${encodeURIComponent(splashHTML)}`);
await waitForServerReady(port);
mainWindow.loadURL(`http://localhost:${port}`);
```

### P4：路径问题

#### 11. main.js 路径解析（多布局兼容）

打包后路径跟开发时不同，必须兼容：

| 模式 | main.js | server.js | serverPath |
|------|---------|-----------|------------|
| 开发 | `electron/` | 项目根 | `path.join(__dirname, '..', 'server.js')` |
| 构建 | `dist/electron/` | `dist/electron/app/` | `path.join(__dirname, 'app', 'server.js')` |
| 打包 | `app.asar/electron/` | `app.asar.unpacked/app/` | `path.join(unpackedPath, 'app', 'server.js')` |

```js
const fs = require('fs');
const path = require('path');

function resolveServerPath() {
    const candidates = [
        path.join(__dirname, 'app', 'server.js'),      // 构建/打包布局
        path.join(__dirname, '..', 'server.js'),         // 开发布局
    ];
    for (const p of candidates) {
        if (fs.existsSync(p)) return p;
    }
    throw new Error('server.js not found');
}
```

#### 12. require.main.filename 打包后失效

打包后 `require.main.filename` 指向 asar 路径，`path.dirname` 后可能解析错误。

**规则**：Electron 主进程代码路径拼接优先用 `__dirname`，别碰 `require.main`。

#### 13. Mac 红绿灯与自定义标题栏重合

**修复**：页面加载后检测平台 → CSS 避让：
```js
// main.js
mainWindow.webContents.on('did-finish-load', () => {
    if (process.platform === 'darwin') {
        mainWindow.webContents.executeJavaScript(
            'document.body.classList.add("platform-darwin")'
        );
    }
});
```
```css
body.platform-darwin .app-header { padding-left: 80px; }
```

## 构建脚本模板

### build.sh（Linux/VM 端）

```bash
#!/bin/bash
set -e

SRC_DIR="$1"          # 源码目录
DIST_DIR="$2"         # 构建输出目录
ELECTRON_DIR="$DIST_DIR/electron"
APP_DIR="$ELECTRON_DIR/app"

# 清理旧构建（保留根目录脚本）
rm -rf "$ELECTRON_DIR"
mkdir -p "$APP_DIR/electron" "$APP_DIR/assets"

# Step 1: 复制 electron 主进程文件
cp "$SRC_DIR/electron/main.js" "$ELECTRON_DIR/"
cp "$SRC_DIR/electron/preload.js" "$ELECTRON_DIR/"
cp "$SRC_DIR/electron/package.json" "$ELECTRON_DIR/"

# Step 2: 复制应用文件到 app/
for f in server.js app.js index.html style.css mobile.js mobile.css i18n.js templates.json favicon.svg; do
    [ -f "$SRC_DIR/$f" ] && cp "$SRC_DIR/$f" "$APP_DIR/"
done
cp -r "$SRC_DIR/assets/"* "$APP_DIR/assets/" 2>/dev/null || true

# Step 3: Terser 压缩（跳过纯注释文件）
for f in main.js preload.js; do
    npx terser "$ELECTRON_DIR/$f" -o "$ELECTRON_DIR/${f}.min" -c -m 2>/dev/null || true
    if [ -s "$ELECTRON_DIR/${f}.min" ]; then
        mv "$ELECTRON_DIR/${f}.min" "$ELECTRON_DIR/$f"
    else
        rm -f "$ELECTRON_DIR/${f}.min"
        echo "⚠️ $f compressed to empty, keeping original"
    fi
done

# Step 4: asarUnpack 完整性校验
UNPACK_FILES=(
    "app/server.js" "app/index.html" "app/style.css"
    "app/app.js" "app/mobile.js" "app/mobile.css"
    "app/favicon.svg" "app/templates.json"
)
MISSING=0
for f in "${UNPACK_FILES[@]}"; do
    if [ ! -f "$ELECTRON_DIR/$f" ]; then
        echo "❌ MISSING: $f"
        MISSING=1
    fi
done
[ $MISSING -eq 1 ] && { echo "❌ Build failed: missing asarUnpack files"; exit 1; }
echo "✅ asarUnpack integrity check passed"
```

### Mac 启动脚本模板（.command）

```bash
#!/bin/bash
cd "$(dirname "$0")"

# 国内镜像源
export ELECTRON_MIRROR="https://npmmirror.com/mirrors/electron/"
export ELECTRON_CUSTOM_DIR="{{ version }}"

# 安装依赖
if [ ! -d "node_modules" ]; then
    npm install
fi

# Electron 版本检测
ELECTRON_VER=$(node -e "console.log(require('./node_modules/electron/package.json').version)" 2>/dev/null)
TARGET_VER="33.0.0"
if [ "$(printf '%s\n' "$TARGET_VER" "$ELECTRON_VER" | sort -V | head -1)" != "$TARGET_VER" ]; then
    echo "Electron $ELECTRON_VER → $TARGET_VER, reinstalling..."
    rm -rf node_modules package-lock.json
    npm install
fi

# 二进制完整性检测
if [ ! -f "node_modules/electron/path.txt" ]; then
    echo "Electron binary missing, downloading..."
    cd node_modules/electron && node install.js && cd ../..
fi

# 清理旧进程
lsof -ti:8000 | xargs kill -9 2>/dev/null
pkill -f "node.*server" 2>/dev/null
sleep 1

# 启动
npm start
```

### Mac 打包脚本模板（.command）

```bash
#!/bin/bash
cd "$(dirname "$0")"

export ELECTRON_MIRROR="https://npmmirror.com/mirrors/electron/"
export ELECTRON_CUSTOM_DIR="{{ version }}"

# 只清旧构建产物，不清 node_modules
rm -rf dist

# 打包（只 DMG，不要 mas 需要证书）
npx electron-builder --mac dmg

echo "✅ DMG built in dist/"
```

## 构建后验证清单

```bash
# 1. asarUnpack 文件完整性
ls -la electron/app/server.js electron/app/index.html electron/app/app.js

# 2. preload.js 非空（terser 坑）
wc -c electron/preload.js
# 必须 > 0

# 3. 文件时间戳（构建比源码新）
ls -lt electron/app/*.js | head -3

# 4. 版本号一致性
grep -r "version" electron/package.json

# 5. git 工作区干净
git status --short
```

## Mac Electron 调试

用户在 Mac 测试时无法直接调试：

1. **Cmd+Option+I** 打开 Electron DevTools
2. 切到 **Console** 标签
3. 复现问题
4. 截图红色报错发过来

**Console 的报错（404/ReferenceError）能直接定位根因，比盲猜高效 10 倍。**

## DMG vs MAS 选择

| 维度 | DMG | MAS (Mac App Store) |
|------|-----|---------------------|
| 代码保护 | asar（可解包） | asar + 沙盒（同级别） |
| 分发 | 自己分发 | App Store 托管 |
| 支付 | 自己接 | StoreKit 内置 |
| 信任 | 用户自行信任 | Apple 审核背书 |
| 成本 | $0 | $99/年 + 30% 抽成 |
| 审核 | 无 | 需要通过 Apple 审核 |

**asar 对 DMG 和 MAS 的代码保护无本质区别。** 选 MAS 主要是为了信任背书+发现性+内置支付。

## 安全/加密方案讨论铁律

讨论加密/分发方案时，**第一句话就要说清楚适用范围**：
- 试用版用什么机制
- App Store 版用什么机制
- 两者是否共用

不要把两个渠道的功能混在一起讨论。

## 参考链接

- electron-builder 文档：https://www.electron.build/
- asarUnpack 配置：https://www.electron.build/configuration#asarunpack
- npmmirror Electron 镜像：https://npmmirror.com/mirrors/electron/
