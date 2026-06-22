# Theos 云端构建完整教程

通用 Theos 云端构建方案：在 `theos-build` 仓库维护一套可复用 workflow，所有 Theos 项目共享，新项目接入只需两个极简调用文件。

---

## 一、架构总览

```
┌─────────────────────────────────────────────────────────┐
│  theos-build 仓库（langshiyunyi/theos-build）            │
│                                                          │
│  .github/workflows/                                      │
│    build.yml    ← 可复用构建 workflow（workflow_call）    │
│    release.yml  ← 可复用发布 workflow（workflow_call）   │
│  README.md      ← 参数说明                                │
│  USAGE.md       ← 本教程                                  │
│                                                          │
│  tag: v1（锁定版本，调用方用 @v1 引用）                   │
└─────────────────────────────────────────────────────────┘
                          ▲
                          │ uses: ...@v1
          ┌───────────────┼───────────────┐
          │               │               │
   DynamicIslandTweak  新项目A        新项目B
   .github/workflows/  .github/...    .github/...
     build.yml          build.yml      build.yml
     release.yml        release.yml    release.yml
   （每个仅 10 行调用）
```

### 核心设计

- **可复用 workflow**：构建逻辑集中在 `theos-build` 仓库，改一处所有项目受益
- **版本锁定**：调用方用 `@v1` tag 引用，theos-build 改动不影响已锁定项目
- **macOS runner**：用 `macos-latest`，`brew install ldid` 一键装签名工具
- **roothide/theos**：用 roothide 官方维护的 Theos fork，同时支持 rootless + roothide
- **双 scheme 矩阵**：rootless + roothide 并行构建，产出两个 deb

---

## 二、theos-build 仓库内容

### 仓库地址

```
git@github.com:langshiyunyi/theos-build.git
https://github.com/langshiyunyi/theos-build
```

### 本地工作副本

```
/private/preboot/.../Documents/theos-build/
```

### 文件结构

```
theos-build/
├── .github/
│   └── workflows/
│       ├── build.yml      # 可复用构建 workflow
│       └── release.yml    # 可复用发布 workflow
├── README.md              # 参数说明
└── USAGE.md               # 本教程
```

### build.yml 核心逻辑

1. **macOS runner**：`runs-on: macos-latest`
2. **缓存 Theos**：`~/theos` 目录跨 run 复用
3. **安装 Theos**：clone `roothide/theos`（兼容官方，支持 roothide）
4. **安装 ldid**：`brew install ldid`
5. **安装 iOS SDK**：从 `theos/sdks` releases 下载 16.5（fallback 15.2/14.5）
6. **构建**：`make package THEOS=~/theos THEOS_PACKAGE_SCHEME={rootless|roothide}`
7. **产物**：从 `control` 文件读包名，artifact 命名 `{包名}-{scheme}`

### release.yml 核心逻辑

1. 调用同仓库 `build.yml` 构建
2. 下载所有 artifact
3. 用 `softprops/action-gh-release@v2` 创建 GitHub Release，附带 deb

---

## 三、可用输入参数

### build.yml 参数

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `scheme` | string | `both` | `rootless` / `roothide` / `both` |
| `sdk-version` | string | 空 | 指定 SDK 版本如 `16.5`，空则自动选 |
| `make-target` | string | `package` | make 目标 |
| `extra-make-flags` | string | 空 | 额外 make 参数（如 `FINALPACKAGE=1`） |
| `artifact-retention` | number | `30` | artifact 保留天数 |
| `theos-ref` | string | `master` | Theos 仓库引用（分支/tag/commit） |

### release.yml 参数

继承 build.yml 所有参数，额外：

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `tag-name` | string | 空 | Release tag，空则用触发 tag |
| `prerelease` | boolean | `false` | 是否标记预发布 |

---

## 四、新项目接入步骤

### 前提条件

- 项目根目录有 `Makefile`（标准 Theos 结构）
- 项目根目录有 `control` 文件（用于读取包名）
- `Makefile` 用 `THEOS ?= ...`（允许命令行覆盖路径，不要用 `THEOS = ...`）
- 项目已推送到 GitHub 仓库

### 步骤 1：创建 workflow 目录

在新项目仓库根目录下创建 `.github/workflows/` 目录。

### 步骤 2：创建 build.yml（日常构建）

文件路径：`.github/workflows/build.yml`

```yaml
name: Build
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    uses: langshiyunyi/theos-build/.github/workflows/build.yml@v1
    with:
      scheme: both
```

### 步骤 3：创建 release.yml（打 tag 发布）

文件路径：`.github/workflows/release.yml`

```yaml
name: Release
on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write

jobs:
  release:
    uses: langshiyunyi/theos-build/.github/workflows/release.yml@v1
    secrets: inherit
    with:
      scheme: both
```

### 步骤 4：提交推送

```bash
cd /path/to/your-project
git add .github/workflows/
git commit -m "ci: add theos cloud build workflows"
git push
```

推送后，每次 push 到 main 自动触发构建，可在 Actions 页面下载 deb artifact。

---

## 五、日常使用场景

### 场景 1：改代码后自动构建

```bash
git add -A
git commit -m "fix: 修复某问题"
git push
```

构建完成后到 Actions 页面下载 deb：
```
https://github.com/<用户名>/<项目名>/actions
```
每个 run 产出两个 artifact：
- `<包名>-rootless`（arm64，适配 Dopamine/palera1n rootless）
- `<包名>-roothide`（arm64e，适配 palera1n roothide）

### 场景 2：发布版本

```bash
git tag v1.0.0
git push origin v1.0.0
```

GitHub 自动创建 Release，附带两个 deb：
```
https://github.com/<用户名>/<项目名>/releases
```

### 场景 3：手动触发构建

1. 打开 GitHub 仓库页面
2. 点击 **Actions** 标签
3. 左侧选 **Build**
4. 右侧点 **Run workflow**
5. 可选 scheme：`rootless` / `roothide` / `both`
6. 点绿色 **Run workflow** 按钮

### 场景 4：自定义构建参数

调用方 workflow 传参：

```yaml
jobs:
  build:
    uses: langshiyunyi/theos-build/.github/workflows/build.yml@v1
    with:
      scheme: rootless                    # 只构建 rootless
      sdk-version: '16.5'                 # 指定 SDK 版本
      extra-make-flags: 'FINALPACKAGE=1'  # 额外 make 参数
      artifact-retention: 90              # artifact 保留 90 天
      theos-ref: master                   # Theos 仓库分支
```

### 场景 5：只构建单一 scheme

如果项目只支持 rootless（不需要 roothide）：

```yaml
jobs:
  build:
    uses: langshiyunyi/theos-build/.github/workflows/build.yml@v1
    with:
      scheme: rootless
```

只产出 1 个 deb artifact，构建更快。

### 场景 6：本地验证（不依赖云端）

```bash
cd /path/to/your-project

# rootless 构建
make clean
make package THEOS=/var/jb/var/mobile/theos THEOS_PACKAGE_SCHEME=rootless

# roothide 构建
make clean
make package THEOS=/var/jb/var/mobile/theos THEOS_PACKAGE_SCHEME=roothide

# deb 产出在 packages/ 目录
ls packages/*.deb
```

本地 Theos 需要是 roothide/theos 分支（支持 roothide scheme）。检查方法：
```bash
ls /var/jb/var/mobile/theos/lib/iphone/roothide/ 2>/dev/null && echo "支持 roothide" || echo "不支持 roothide，需安装 roothide/theos"
```

---

## 六、升级 theos-build（改构建逻辑）

当需要修改构建逻辑（如换 SDK 版本、加新依赖、改构建参数）时：

```bash
cd /private/preboot/60EB5887E734CB8D57541DED891DA2651D2D4DC4EB287427F6175CA7A63D97251CB731DAB4FAD4B121BC68FEE4699C30/dopamine-BYouLN/procursus/var/mobile/Documents/theos-build

# 改 build.yml 或 release.yml
vim .github/workflows/build.yml

# 提交推送 main
git add -A
git commit -m "改进构建逻辑"
git push origin main

# 移动 v1 tag 到新 commit（所有用 @v1 的项目自动用新逻辑）
git tag -f v1
git push --force origin v1
```

### 版本管理建议

- **v1**：稳定版，调用方用 `@v1` 引用
- **main**：最新开发版，可通过 `@main` 引用测试
- 重大改动时打 `v2` tag，调用方按需升级

---

## 七、产物差异说明

| scheme | 适用越狱方案 | deb 架构 | 安装路径前缀 |
|--------|------------|---------|-------------|
| rootless | Dopamine / palera1n rootless | `iphoneos-arm64` | `/var/jb/` |
| roothide | palera1n roothide | `iphoneos-arm64e` | roothide 动态路径 |

两个 deb 功能相同，只是架构和安装路径前缀不同，适配不同越狱方案。用户根据自己设备的越狱方式选择对应 deb 安装。

---

## 八、Makefile 要求

### 正确写法（允许命令行覆盖）

```makefile
THEOS ?= /var/jb/var/mobile/theos
export ARCHS = arm64 arm64e
export TARGET = iphone:clang:latest:15.0
THEOS_PACKAGE_SCHEME = rootless

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = MyTweak
MyTweak_FILES = Tweak.x
MyTweak_CFLAGS = -fobjc-arc

include $(THEOS_MAKE_PATH)/tweak.mk
```

### 关键点

1. **`THEOS ?=` 而非 `THEOS =`**：`?=` 允许命令行/环境变量覆盖，云端构建传 `THEOS=/path/to/ci/theos` 才能生效
2. **`THEOS_PACKAGE_SCHEME = rootless` 可被命令行覆盖**：`make package THEOS_PACKAGE_SCHEME=roothide` 会覆盖 Makefile 里的赋值
3. **`control` 文件必须有 `Package:` 字段**：workflow 从这里读包名命名 artifact

### control 文件示例

```
Package: com.example.mytweak
Name: My Tweak
Version: 1.0.0
Architecture: iphoneos-arm64
Description: A sample tweak
Maintainer: YourName
Author: YourName
Section: Tweaks
Depends: mobilesubstrate, preferenceloader, firmware (>= 15.0)
```

`Architecture` 字段会被 Theos 根据 scheme 自动覆盖（rootless → arm64，roothide → arm64e），这里写占位值即可。

---

## 九、调试与问题排查

### 查看 Actions 构建状态

**方式 1：GitHub 网页**

打开 `https://github.com/<用户名>/<项目名>/actions`

**方式 2：API 查询（无需认证，公开仓库）**

```bash
# 查看最近的 runs
curl -sS "https://api.github.com/repos/<用户名>/<项目名>/actions/runs?per_page=5"

# 查看某次 run 的 jobs 和步骤状态
curl -sS "https://api.github.com/repos/<用户名>/<项目名>/actions/runs/<run_id>/jobs"
```

### 常见问题

#### 问题 1：`startup_failure`

**原因**：调用方 workflow 语法错误，或 `with` 里用了不支持的表达式

**解决**：`with` 参数用字面量或简单表达式，避免 `github.event.inputs.X || 'default'` 这类复杂表达式在非 workflow_dispatch 触发时导致解析失败

#### 问题 2：`reusable workflow not found`

**原因**：`theos-build` 仓库的 `v1` tag 不存在或未推送

**解决**：
```bash
cd /path/to/theos-build
git tag v1
git push origin v1
```

#### 问题 3：`ldid: command not found`

**原因**：Ubuntu runner 上源码编译 ldid 失败

**解决**：已改用 macOS runner + `brew install ldid`，确保 theos-build 用的是最新 v1

#### 问题 4：roothide 构建失败

**原因**：用了官方 `theos/theos`，不支持 roothide scheme

**解决**：已改用 `roothide/theos`（roothide 官方 fork），确保 theos-build 的 build.yml 里 clone 源是 `https://github.com/roothide/theos.git`

#### 问题 5：SDK 下载失败

**原因**：`theos/sdks` 仓库 URL 结构变化

**解决**：workflow 有 fallback 机制（16.5 → 15.2 → 14.5 → 13.0），都失败才报错。可手动指定 `sdk-version` 参数

#### 问题 6：artifact 里没有 deb

**原因**：项目构建失败（源码错误、依赖缺失等）

**解决**：查看 Actions 页面对应 job 的日志，或本地 `make package` 验证

### 清理缓存

如果 Theos 缓存导致问题（如换了 theos 源后缓存还指向旧的）：

1. GitHub 仓库 → Actions → 左侧选 **Build**
2. 点 **Run workflow** 时缓存 key 会自动判断
3. 或在 theos-build 改缓存 key 版本号（如 `v2` → `v3`）

---

## 十、仓库清单

### theos-build 仓库

- **地址**：`git@github.com:langshiyunyi/theos-build.git`
- **本地路径**：`/private/preboot/.../Documents/theos-build/`
- **作用**：集中维护构建逻辑
- **引用方式**：`langshiyunyi/theos-build/.github/workflows/build.yml@v1`

### DynamicIslandTweak 仓库（示例项目）

- **地址**：`git@github.com:langshiyunyi/DynamicIslandTweak.git`
- **本地路径**：`/private/preboot/.../Documents/DynamicIslandTweak/`
- **作用**：已接入 theos-build 的示例项目，可参考其 workflow 文件

### 官方依赖仓库

- **roothide/theos**：`https://github.com/roothide/theos`（Theos fork，支持 roothide）
- **theos/sdks**：`https://github.com/theos/sdks`（iOS SDK 下载）
- **theos/ldid**：`https://github.com/theos/ldid`（代码签名工具，macOS 用 brew 装）

---

## 十一、快速参考

### 新项目接入清单

- [ ] 项目根目录有 `Makefile`（`THEOS ?=` 写法）
- [ ] 项目根目录有 `control` 文件（含 `Package:` 字段）
- [ ] 项目已推送到 GitHub 仓库
- [ ] 创建 `.github/workflows/build.yml`（复制模板）
- [ ] 创建 `.github/workflows/release.yml`（复制模板）
- [ ] 推送到 GitHub，Actions 自动触发

### 调用方模板（build.yml）

```yaml
name: Build
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    uses: langshiyunyi/theos-build/.github/workflows/build.yml@v1
    with:
      scheme: both
```

### 调用方模板（release.yml）

```yaml
name: Release
on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write

jobs:
  release:
    uses: langshiyunyi/theos-build/.github/workflows/release.yml@v1
    secrets: inherit
    with:
      scheme: both
```

### 常用命令

```bash
# 推送代码触发构建
git push

# 打 tag 发布
git tag v1.0.0 && git push origin v1.0.0

# 升级 theos-build 后移动 v1 tag
cd /path/to/theos-build
git tag -f v1 && git push --force origin v1

# 本地验证构建
make clean && make package THEOS=/var/jb/var/mobile/theos THEOS_PACKAGE_SCHEME=rootless
```
