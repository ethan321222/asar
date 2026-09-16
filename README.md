# asar

基于 [@electron/asar](https://github.com/electron/asar) 的 fork，修复了官方 `asar extract` 命令在处理平台相关文件时的 bug。

## 原始问题

使用官方 `asar extract` 命令解包时会报错：

```
asar extract ./app.asar ./app

Error: ENOENT: no such file or directory, open 'D:\app\resources\app.asar.unpacked\node_modules\@napi-rs\canvas-darwin-arm64\package.json'
    at Object.openSync (node:fs:560:18)
    at Object.readFileSync (node:fs:444:35)
    at module.exports.readFileSync (...\asar\lib\disk.js:110:17)
    at module.exports.extractAll (...\asar\lib\asar.js:204:28)
```

之前已经对应issue

https://github.com/electron/asar/issues/37

#37:fails with an error "No such file or directory"


## 问题分析

基于上述issue的讨论记录

`asar extract` 在处理 `unpacked` 文件时存在设计缺陷。

Electron 应用可以将部分文件标记为 `unpacked`，这些文件不打包在 `.asar` 内部，而是存放在同级的 `app.asar.unpacked` 目录中。通常以下文件会被标记为 unpacked：

- 原生 Node.js 模块（`.node` 文件）
- 包含平台相关二进制的包（如 `@napi-rs/canvas-darwin-arm64`）
- 需要在文件系统上直接可访问的资源

**问题出在哪里：**

1. asar 头文件中记录了所有文件的元信息，包括标记为 `unpacked` 的文件路径
2. 官方 `asar extract` 遇到 `unpacked` 标记时，会直接用 `fs.readFileSync` 去读取 `app.asar.unpacked` 中的对应文件
3. 但在**跨平台场景**下（例如在 Windows 上解包一个包含 macOS 原生模块的 app），`app.asar.unpacked` 中可能并不存在所有平台的原生模块
4. 官方工具**没有做文件存在性检查**，直接读取导致 `ENOENT` 崩溃

**具体到本例：** `@napi-rs/canvas-darwin-arm64` 是 macOS ARM64 的原生模块，在 Windows 上的 `app.asar.unpacked` 中自然不存在这个文件，官方工具在尝试读取其 `package.json` 时就崩溃了。

## 解决方案

本工具在提取 unpacked 文件时会**检查文件是否存在**：
- 存在 → 正常复制
- 不存在 → 创建空文件占位，继续处理后续文件

这样即使缺少特定平台的原生模块，解包过程也能正常完成。

## 使用方法

### 环境要求

- Node.js >= 22.12.0

### 安装

```bash
# 克隆仓库
git clone https://github.com/ethan321222/asar.git
cd asar

# 安装依赖并编译
yarn install
yarn build
```

**方式 1：全局安装（推荐）**

```bash
npm install -g .
```

**方式 2：本地链接（开发调试）**

```bash
npm link
```

> `npm link` 创建符号链接，修改代码后重新 `yarn build` 即可生效，无需重新安装。

**方式 3：直接运行（无需安装）**

```bash
node bin/asar.mjs extract ./app.asar ./output
```

**方式 4：从 CI 产物安装（无需 clone 和 build）**

从 [GitHub Actions](https://github.com/ethan321222/asar/actions) 下载最新的 `asar-package` artifact，解压后安装：

```bash
npm install -g ./electron-asar-0.0.0-development.tgz
```

> 每次 push 到 `main` 都会自动构建并上传产物，下载即可使用。

**安装方式对比：**

| 方式 | 本质 | 需要 clone | 需要 build | 适用场景 |
|------|------|------------|------------|----------|
| `npm install -g .` | 复制文件 | ✅ | ✅ | 正式使用 |
| `npm link` | 符号链接 | ✅ | ✅ | 开发调试 |
| `node bin/asar.mjs` | 直接运行 | ✅ | ✅ | 临时测试 |
| CI 产物 `.tgz` | 预编译包 | ❌ | ❌ | 快速使用 |

### 使用命令

```bash
# 解包 asar 文件
asarx extract ./app.asar ./output

# 解包到默认目录
asarx extract ./app.asar

# 查看帮助
asarx --help
```

### 其他命令

```bash
# 打包
asarx pack <dir> <output>

# 列出文件
asarx list <archive>

# 提取单个文件
asarx extract-file <archive> <filename>
```

### 卸载

```bash
npm uninstall -g .
```

## 与官方工具的区别

| 特性 | `asar extract` | `asarx extract` |
|------|---------------|-----------------|
| 处理缺失的外部文件 | ❌ 报错退出 | ✅ 创建空文件继续 |
| 处理平台相关模块 | ❌ 需要所有文件存在 | ✅ 自动跳过缺失文件 |