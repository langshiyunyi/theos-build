# theos-build

通用 Theos 云端构建工作流（reusable workflow）。所有 Theos 项目共享一套构建逻辑，新项目接入只需几行调用。

## 提供的工作流

| 文件 | 类型 | 用途 |
|------|------|------|
| `.github/workflows/build.yml` | `workflow_call` | 构建 `.deb` 并上传 artifact |
| `.github/workflows/release.yml` | `workflow_call` | 构建 + 创建 GitHub Release |

## 新项目接入（2 个文件）

在任意 Theos 项目仓库的 `.github/workflows/` 下放两个极简调用文件即可。

### 1. `.github/workflows/build.yml`（日常构建）

```yaml
name: Build
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

jobs:
  build:
    uses: langshiyunyi/theos-build/.github/workflows/build.yml@v1
    with:
      scheme: both              # rootless / roothide / both
```

### 2. `.github/workflows/release.yml`（打 tag 发布）

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

## 可用输入参数

### build.yml

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `scheme` | string | `both` | `rootless` / `roothide` / `both` |
| `sdk-version` | string | 空 | 指定 SDK 版本如 `16.5`，空则自动选 |
| `make-target` | string | `package` | make 目标 |
| `extra-make-flags` | string | 空 | 额外 make 参数 |
| `artifact-retention` | number | `30` | artifact 保留天数 |
| `theos-ref` | string | `master` | Theos 仓库引用 |

### release.yml

继承 build.yml 所有参数，额外：

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `tag-name` | string | 空 | Release tag，空则用触发 tag |
| `prerelease` | boolean | `false` | 是否标记预发布 |

## 项目要求

- 根目录有 `Makefile`（标准 Theos 结构）
- 根目录有 `control` 文件（用于读取包名命名 artifact）
- `Makefile` 用 `THEOS ?= ...` 而非 `THEOS = ...`（允许命令行覆盖）

## 本地测试此仓库自身

theos-build 仓库本身也能手动触发构建（用于测试工作流）：

```bash
git tag v0.0.1-test
git push origin v0.0.1-test
```

或在 GitHub Actions 页面手动 `Run workflow`。

## 构建逻辑

1. Ubuntu runner 安装 Theos 依赖 + ldid（源码编译）
2. 缓存 `/opt/theos` 跨 run 复用
3. 下载 iOS SDK（16.5 → 15.2 → 14.5 → 13.0 自动 fallback）
4. `make clean && make package THEOS=/opt/theos THEOS_PACKAGE_SCHEME=...`
5. 从 `packages/*.deb` 收集产物上传 artifact
6. release.yml 下载所有 artifact 创建 GitHub Release

## 维护

- 改构建逻辑：只改本仓库 `build.yml`，所有引用项目自动生效
- 发新版本：打 tag `v1`、`v2`...，引用项目用 `@v1` 锁定
