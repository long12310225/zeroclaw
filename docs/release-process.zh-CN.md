# ZeroClaw发布流程

此运行手册定义了维护者的标准发布流程。

最后验证：**2026年2月21日**。

## 发布目标

- 保持发布可预测和可重复。
- 仅从已存在于`main`中的代码发布。
- 在发布前验证多目标工件。
- 即使PR量大也要保持定期发布节奏。

## 标准节奏

- 补丁/次要版本：每周或双周。
- 紧急安全修复：临时发布。
- 永远不要等待非常大的提交批次积累。

## 工作流程契约

发布自动化位于：

- `.github/workflows/pub-release.yml`
- `.github/workflows/pub-homebrew-core.yml`（手动Homebrew公式PR，机器人拥有）

模式：

- 标签推送`v*`：发布模式。
- 手动触发：仅验证或发布模式。
- 每周计划：仅验证模式。

发布模式防护措施：

- 标签必须匹配类似semver的格式`vX.Y.Z[-suffix]`。
- 标签必须已存在于origin上。
- 标签提交必须可从`origin/main`到达。
- 在GitHub Release发布完成之前，必须有匹配的GHCR镜像标签（`ghcr.io/<owner>/<repo>:<tag>`）。
- 工件在发布前经过验证。

## 维护者程序

### 1) 在`main`上预检

1. 确保最新的`main`上所需的检查通过。
2. 确认没有高优先级事件或已知回归问题开放。
3. 确认最近的`main`提交上安装程序和Docker工作流程健康。

### 2) 运行验证构建（不发布）

手动运行`Pub Release`：

- `publish_release`: `false`
- `release_ref`: `main`

预期结果：

- 完整的目标矩阵构建成功。
- `verify-artifacts`确认所有预期的归档文件存在。
- 不发布GitHub Release。

### 3) 创建发布标签

从与`origin/main`同步的干净本地检出：

```bash
scripts/release/cut_release_tag.sh vX.Y.Z --push
```

此脚本强制执行：

- 干净的工作树
- `HEAD == origin/main`
- 非重复标签
- 类似semver的标签格式

### 4) 监控发布运行

标签推送后，监控：

1. `Pub Release`发布模式
2. `Pub Docker Img`发布作业

预期发布输出：

- 发布归档文件
- `SHA256SUMS`
- `CycloneDX`和`SPDX` SBOM
- cosign签名/证书
- GitHub Release说明 + 资产

### 5) 发布后验证

1. 验证GitHub Release资产可下载。
2. 验证发布的版本（`vX.Y.Z`）和发布提交SHA标签（`sha-<12>`）的GHCR标签。
3. 验证依赖发布资产的安装路径（例如引导二进制下载）。

### 6) 发布Homebrew Core公式（机器人拥有）

手动运行`Pub Homebrew Core`：

- `release_tag`: `vX.Y.Z`
- `dry_run`: 先`true`，然后`false`

非dry-run所需的仓库设置：

- 密钥：`HOMEBREW_CORE_BOT_TOKEN`（来自专用机器人账户的令牌，不是个人维护者账户）
- 变量：`HOMEBREW_CORE_BOT_FORK_REPO`（例如`zeroclaw-release-bot/homebrew-core`）
- 可选变量：`HOMEBREW_CORE_BOT_EMAIL`

工作流程防护措施：

- 发布标签必须匹配`Cargo.toml`版本
- 公式源URL和SHA256从标记的tarball更新
- 公式许可证规范化为`Apache-2.0 OR MIT`
- PR从机器人fork打开到`Homebrew/homebrew-core:master`

## 紧急/恢复路径

如果标签推送发布在工件验证后失败：

1. 在`main`上修复工作流程或打包问题。
2. 使用以下参数重新运行手动`Pub Release`发布模式：
   - `publish_release=true`
   - `release_tag=<现有标签>`
   - `release_ref`在发布模式下自动固定到`release_tag`
3. 重新验证已发布的资产。

## 操作说明

- 保持发布变更小且可逆。
- 更倾向于每个版本一个发布问题/检查清单，以便交接清晰。
- 避免从临时功能分支发布。