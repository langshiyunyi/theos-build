# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库性质

本仓库**不是 Theos 项目**（无 `Makefile`/`control`/源码），而是一个**可复用 GitHub Actions 工作流仓库**，为所有 Theos 越狱项目集中提供云端构建/发布逻辑。下游项目通过 `uses: langshiyunyi/theos-build/.github/workflows/build.yml@v1` 引用，自身只需约 10 行调用文件。

仓库仅含 4 个文件：
- `.github/workflows/build.yml` — 可复用构建 workflow（`workflow_call`）
- `.github/workflows/release.yml` — 可复用发布 workflow（`workflow_call`，内部调用 build.yml）
- `README.md` — 参数参考
- `USAGE.md` — 完整教程

## 架构

```
theos-build 仓库 (tag: v1)
  build.yml   ← workflow_call 可复用构建
  release.yml ← workflow_call 可复用发布（调用 build.yml 后创建 GitHub Release）
        ▲
        │ uses: ...@v1
  下游 Theos 项目（各自有 Makefile + control + 源码）
```

**版本锁定**：调用方用 `@v1` tag 引用。本仓库改动通过移动 `v1` tag（`git tag -f v1 <commit> && git push -f origin v1`）传播，已锁定 `@v1` 的项目自动跟进。

## build.yml 流水线

`prepare` job（macos-latest）→ 读取调用方 `control` 文件的 `Package:` 行得到包名 → 构建 scheme 矩阵（`both` 拆成 `["rootless","roothide"]`）。

`build` job（macos-latest，matrix over schemes）：
1. 缓存 `~/theos`（key: `theos-macos-{theos-ref}-v2`）
2. 未命中缓存则 clone `roothide/theos`（roothide 官方 fork，兼容官方 Theos 且支持 roothide scheme）
3. `brew install ldid`（签名工具）
4. 下载 iOS SDK（优先指定版本；否则 16.5 → 15.2 → 14.5 fallback）
5. `make clean`（允许失败）→ `make package THEOS=~/theos THEOS_PACKAGE_SCHEME={scheme}`
6. 从 `packages/*.deb` 收集到 `artifacts/`
7. 上传 deb artifact，命名 `{pkg-name}-{scheme}`

## release.yml 流水线

`build` job → `uses: ./.github/workflows/build.yml`（`artifact-retention: 7`）。
`release` job（ubuntu-latest，`permissions: contents: write`）→ 下载所有 artifact → 汇集 `.deb` 到 `release/` → tag 名取 `inputs.tag-name` 或 `github.ref_name` → `softprops/action-gh-release@v2` 创建 Release（`generate_release_notes: true`）。

## 下游项目接入要求

调用方仓库根目录必须：
- 有标准 Theos `Makefile`
- 有 `control` 文件（用于读取包名命名 artifact）
- `Makefile` 中用 `THEOS ?= ...` 而非 `THEOS = ...`（允许 workflow 通过命令行覆盖）

## 修改与测试

**改构建逻辑**：只改本仓库 `build.yml`（或 `release.yml`），所有引用 `@v1` 的下游项目在 tag 移动后自动生效。

**测试工作流自身**：
- GitHub Actions 页面手动 `Run workflow`（`workflow_dispatch`）
- 或打测试 tag：`git tag v0.0.x-test && git push origin v0.0.x-test`

**发新版本**：打 `v1`/`v2`... tag，下游用 `@v1` 锁定。

## 已知文档与代码不一致

`README.md` 的"构建逻辑"段仍写"Ubuntu runner 安装 Theos 依赖 + ldid（源码编译）"和"缓存 `/opt/theos`"，但实际 `build.yml` 已在 commit `c0ef213` 切换为 **macOS runner + `brew install ldid`**，路径为 `~/theos`。改 build 逻辑时注意同步 README.md。

## key inputs 速查

| build.yml | 默认 | 说明 |
|---|---|---|
| `scheme` | `both` | `rootless` / `roothide` / `both` |
| `sdk-version` | 空 | 指定如 `16.5`，空则自动 fallback |
| `make-target` | `package` | make 目标 |
| `extra-make-flags` | 空 | 额外 make 参数 |
| `artifact-retention` | `30` | artifact 保留天数 |
| `theos-ref` | `master` | Theos 仓库分支/tag/commit |

release.yml 继承以上全部，另加 `tag-name`（空则用触发 tag）、`prerelease`（默认 `false`）。
