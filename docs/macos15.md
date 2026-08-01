# macOS 15 (Sequoia) 通道：部署与发布

> 这份文档总结 Tinycast 的 macOS 15 部署方案与发布流程，重点讲**如何通过本地 CLI 把版本推到 release**。
> 注意：`compat/` 目录与 `.github/workflows/release-sequoia.yml` 只存在于 `compat/macos15` 分支上，
> 在 `main` 上找不到它们属正常。代码层面的操作参考在 [`compat/README.md`](../compat/README.md)。

## 背景与目标

`main` 面向 macOS 26，使用 Liquid Glass 材质，`glassEffect(_:in:)` 是 `@available(macOS 26)` 的 API。
目标：让**同一份代码**也能在 macOS 15 (Sequoia) 上编译、运行、发布，同时**不改 `main` 上的任何一个字节**。

## 方案总览

三条路里选了第一条：

| 方案 | 结论 |
|---|---|
| **Xcode 26 + 只降 deployment target**，两处 glass 调用点用 patch 门控 | ✅ 采用。改动面只有 2 个调用点 |
| 只降 deployment target，不改源码 | ❌ 编译期 hard error——deployment target 本身就是 availability 契约，没有编译开关能压制 |
| 用 Xcode 16 / macos-15 runner 构建 | ❌ 会拖进约 30 个文件无关的并发改动（`@MainActor` 注解、`@preconcurrency import Darwin`、`#if compiler(>=6.2)`）。曾有废弃分支试过，不要重蹈 |

落地形态：

- **`compat/macos15` 分支**：只允许携带 `compat/` 目录和 `release-sequoia.yml`，绝不允许 `Tinycast/` 下的改动（`release.sh` 会检查并中止）。
- **`compat/macos15.patch`**：main 的源码在构建时被 patch 出门控版本。patch 是手写还是生成？——由 `macos15-compat` skill 重新生成，绝不手工改 hunk 偏移。
- **门控点**：全部收敛在 `Tinycast/Core/Theme.swift` 的 `FrostedSurface` ViewModifier 里，共两个 helper：

  | Helper | macOS 26 | macOS 15 回退 |
  |---|---|---|
  | `frosted(in:)`（悬浮 pill / 菜单圆钮） | `glassEffect(.regular.interactive().tint(glassFrost))` | `.ultraThinMaterial` + frost 覆盖层 + 0.5pt 发丝线 + 软阴影 |
  | `frostedMenu(in:)`（popover 面板） | `glassEffect(.regular)` | 同上回退 |

  `Features/PopoverMenu.swift` 只调 `.frostedMenu(in:)`，不直接写 `glassEffect`。主面板是 `NSVisualEffectView` vibrancy（macOS 10.10 的 API），Sequoia 上原样可用，不受影响。

验证结论（已确认的事实）：这两处门控后，整个 app 能 `-target arm64-apple-macos15.0` 编译、出 Release、
`minos 15.0` / `LSMinimumSystemVersion 15.0`，六个 SwiftUI glass 符号变成 **weak import**，Sequoia 上 dyld 绑到 null、
`#available` 守卫保证永不调用。

## 本地 CLI 发布流程（核心）

一条命令，在 `compat/macos15` 分支上执行，同步、验证、打 tag、推送全部完成，不需要任何手工 git 操作：

```sh
./compat/release.sh              # 完整发布
./compat/release.sh --dry-run    # 只同步 + 验证 + 展示 tag，不发布任何东西
./compat/release.sh --version 0.8.0        # 指定版本（默认自动推导，见下）
./compat/release.sh --retag      # 替换已存在的 tag（会重跑那次 release）
```

### 命令内部逐步拆解

对照 `compat/release.sh` 的源码顺序：

1. **Preflight** —— 必须在 `compat/macos15` 分支上（拒绝自动切分支，避免在用户不知情下切换）；工作区必须干净；`compat/verify.sh` 必须存在。
2. **`git fetch origin --tags`** —— 拉取最新远端状态。
3. **`git merge origin/main`** —— 把最新 main 合进来。冲突则 `merge --abort` 并以退出码 1 中止；正常情况下只有 `compat/` 文件会冲突。
4. **守卫检查** —— `git diff origin/main..HEAD` 里若出现任何 `Tinycast/` 下的文件，直接中止：分支存在的全部意义就是 main 不被改动。
5. **`./compat/verify.sh`** —— 完整验证（见下节），失败以退出码 2 中止。
6. **解析版本** —— 默认取最新的 **stable** mainline tag（过滤掉 `-alpha/-beta/-rc` 和 `-sequoia`），如 `v0.7.5` → 发布为 `v0.7.5-sequoia`。Sequoia 版本永远跟随 macOS 26 通道最新发布的版本。可用 `--version` 覆盖；版本格式必须匹配 `X.Y.Z` 或 `X.Y.Z-beta.N`。
7. **tag 存在性检查** —— `v<版本>-sequoia` 已存在（本地或远端）时：无 `--retag` 就中止，要求你选 `--retag` 替换或 `--version` 换新版本。
8. **推送** —— 依次：`git push origin compat/macos15` → 打 annotated tag `v<版本>-sequoia` → `git push origin <tag>`。**push tag 这一步触发 CI**。
9. **输出** —— 打印 workflow 地址和一行可直接执行的 `gh run watch ...` 命令。

### 为什么是 tag 触发而不是手动按钮

`workflow_dispatch` 只对**默认分支上存在**的 workflow 生效，而 `release-sequoia.yml` 刻意只放在 `compat/macos15` 上（不让 main 沾一点 compat 的东西）。tag push 没有这个限制，所以发布 = 打 tag + 推 tag，`release.sh` 替你把这两步做了。

### 退出码速查

| 退出码 | 含义 | 处置 |
|---|---|---|
| 0 | 成功 | 去 `gh run watch` 看 CI |
| 1 | 与 `origin/main` 合并冲突 | 解决（`Tinycast/` 下的冲突一律取 main 的版本）后重跑 |
| 2 | verify 失败——patch 烂了或 main 新增了 26-only API | 跑 `macos15-compat` skill 修复 |
| 3 | preflight 失败（脏树 / 错分支 / tag 已存在 / 版本格式错） | 按报错修，重跑 |

## 验证是怎么做的（verify.sh）

`verify.sh` **永不碰工作区**：`git archive HEAD` 导出 tracked 文件到临时目录（顺带把未提交改动也 overlay 上去），在副本里打 patch 再构建。所以在任何分支、带未提交改动都能安全跑。

```sh
./compat/verify.sh --quick    # ~35s：patch 应用 + macOS 15/26 双目标 typecheck
./compat/verify.sh            # 再加：Release build（deployment target 15.0）+ 产物断言
```

完整模式的三个断言：

1. **15.0 floor** —— `xcrun vtool` 读 `minos`、`PlistBuddy` 读 `LSMinimumSystemVersion`，都必须恰好是 `15.0`。防止 `MACOSX_DEPLOYMENT_TARGET` 覆盖被悄悄丢掉、把 26 的包当 sequoia 发出去。
2. **weak linkage** —— `nm -m` 检查二进制里 glass 符号必须全是 weak。非 weak 的强未定义符号会让 dyld 在 Sequoia 上启动即崩。
3. **双目标 typecheck** —— 15.0 和 26.0 各 typecheck 一遍，保证 patch 后 26 仍然能编译。

⚠️ 编译器每次只报**一个** availability 错误——修完一个立刻全绿不代表真干净，要循环跑到 genuinely clean。

## tag 推送后 CI 里发生什么

`release-sequoia.yml`（`on: push: tags: ["v*-sequoia"]`，`runs-on: macos-26`，刻意不用 macos-15/Xcode 16）：

1. checkout → 选中 Xcode 26。
2. **解析 tag**：`v0.2.0-sequoia` → 版本 `0.2.0`、channel stable（`v0.2.0-beta.3-sequoia` → beta）、bundle id / display name / cask 名。格式不合法直接失败，绝不让错 tag 变成 release。
3. **`./compat/verify.sh` 再跑一遍**——发布任何东西之前先验证，与本地一致。
4. `git apply compat/macos15.patch` 打上补丁。
5. **导入签名证书**（`SIGNING_P12_BASE64` secret）。没有这个 secret 直接报错：签名不一致会让所有用户的 Accessibility (TCC) 授权静默失效。
6. `xcodebuild` Release，`MACOSX_DEPLOYMENT_TARGET=15.0`，`PRODUCT_NAME`/`PRODUCT_BUNDLE_IDENTIFIER`/`MARKETING_VERSION` 来自第 2 步。
7. **对将要发布的二进制再断言一次** 15.0 floor + weak glass（belt and braces）。
8. 打包 DMG（`diskutil image create` UDZO）→ 上传 artifact。
9. **发布 GitHub Release**——**永远 `--prerelease`**，包括 stable 通道。原因：`website/src/lib/use-version.ts` 读的是 `/releases/latest`（GitHub 定义为最新非 prerelease），完整 release 会劫持网站上显示的版本号。
10. **更新 Homebrew tap cask**（`HOMEBREW_TAP_TOKEN` secret，`Casks/tinycast-sequoia.rb`）。cask 步骤硬性要求文件已存在，否则报错。

### 两个身份设计，都是故意的

- **Bundle ID / 显示名与主通道一致**（`com.tinycast.app` / `Tinycast`）——Sequoia 用户以后升到 macOS 26 装主通道，prefs、缓存、登录项、Accessibility 授权全部保留（都以 `Bundle.main.bundleIdentifier` 为键）。唯一区别是 mainline 有 `Tinycast Dev` 的 dev 通道，Sequoia 通道没有独立 dev 通道的文档。
- **Homebrew 是唯一更新路径**（无 Sparkle），cask 的 `depends_on macos:` 是防止 Sequoia 机器拿到 26 构建的唯一闸门。注意语义：`depends_on macos: :sequoia` 是 **`>= macOS 15`（最小值）**，不是精确匹配；反向（26 用户显式装 sequoia cask）挡不住，那是可接受的。

## 日常操作速查

```sh
# 发布（在 compat/macos15 分支上）
./compat/release.sh

# 只看不发布
./compat/release.sh --dry-run

# main 更新后重新同步
git checkout compat/macos15 && git merge main && ./compat/verify.sh

# 盯 CI
gh run list --workflow=release-sequoia.yml -L1
gh run watch $(gh run list --workflow=release-sequoia.yml -L1 --json databaseId -q '.[0].databaseId')

# patch 烂了 / main 新增 26-only API —— 修复 + 发布的唯一入口
# 跑 macos15-compat skill，它会重新生成 patch、验证并 release，不用自己碰 git
```

发布后确认 `main` 没被碰过：

```sh
git diff --stat origin/main..origin/compat/macos15   # 应只见 compat/ 与 workflow，Tinycast/ 下一无所有
```

## 已知边界（发布前务必知悉）

- **没有任何东西在真实 macOS 15 上跑过。** 绿的工作流只证明"能编译、能链接、声明了 15.0 floor"——不证明回退材质渲染正确、app 行为正常。发布前应在 Sequoia 真机上确认：回退材质读起来像悬浮控件而不是平盒子、Accessibility 弹窗与 paste-to-`previousApp` 焦点恢复、Hyper Key 的 `hidutil` remap、emoji 网格与剪贴板缩略图。
- `onScrollGeometryChange` / `onGeometryChange` 恰好是 **macOS 15.0** 的 API，能用但毫无余量——15 是硬地板，macOS 14 不可达。
- `isolated deinit` 在 Xcode 26 + target 15 下回退部署没问题，**不是**阻塞项，别去"修"它。
- `LSMinimumSystemVersion` 来自 `$(MACOSX_DEPLOYMENT_TARGET)`，覆盖会自动传播，无需重跑 `xcodegen`。
- `tinycast@beta-sequoia` 的 cask 尚不存在——打 `-beta.N-sequoia` 的 tag 会在 cask 步骤失败。

## 关键参考文件

- `compat/release.sh` —— 本地 CLI 发布脚本（本文主线的实现）
- `compat/verify.sh` —— 本地验证脚本
- `compat/macos15.patch` —— 唯一的门控补丁（由 skill 生成，勿手改）
- `compat/README.md` —— 分支上的英文操作参考（tap 语义、身份设计细节）
- `.github/workflows/release-sequoia.yml` —— tag 触发的 CI 发布
- `.claude/skills/macos15-compat/SKILL.md` —— 修复 + 发布的 skill 全流程
