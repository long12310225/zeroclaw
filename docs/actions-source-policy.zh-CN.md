# 操作源策略（第一阶段）

本文档定义了此仓库的当前GitHub Actions源代码控制策略。

第一阶段目标：在完整SHA固定之前，以最小干扰锁定操作源。

## 当前策略

- 仓库Actions权限：启用
- 允许的操作模式：选定
- 是否需要SHA固定：否（推迟到第二阶段）

选定的允许列表模式：

- `actions/*`（涵盖`actions/cache`、`actions/checkout`、`actions/upload-artifact`、`actions/download-artifact`以及其他第一方操作）
- `docker/*`
- `dtolnay/rust-toolchain@*`
- `DavidAnson/markdownlint-cli2-action@*`
- `lycheeverse/lychee-action@*`
- `EmbarkStudios/cargo-deny-action@*`
- `rustsec/audit-check@*`
- `rhysd/actionlint@*`
- `softprops/action-gh-release@*`
- `sigstore/cosign-installer@*`
- `Checkmarx/vorpal-reviewdog-github-action@*`
- `useblacksmith/*`（Blacksmith自托管运行器基础设施）

## 变更控制导出

使用以下命令导出当前有效策略以进行审计/变更控制：

```bash
gh api repos/zeroclaw-labs/zeroclaw/actions/permissions
gh api repos/zeroclaw-labs/zeroclaw/actions/permissions/selected-actions
```

记录每次策略变更：

- 变更日期/时间（UTC）
- 操作者
- 原因
- 允许列表差异（添加/删除的模式）
- 回滚说明

## 为什么采用此阶段

- 减少来自未经审查的市场操作的供应链风险。
- 保持当前CI/CD功能，迁移开销低。
- 为第二阶段完整SHA固定做准备，不阻塞活跃开发。

## 智能工作流防护措施

由于此仓库具有较高的代理编写变更量：

- 任何添加或更改`uses:`操作源的PR必须包含允许列表影响说明。
- 新的第三方操作需要维护者明确审查后才能加入允许列表。
- 仅针对验证过的缺失操作扩展允许列表；避免广泛的通配符例外。
- 在PR描述中保留操作策略变更的回滚指令。

## 验证检查清单

允许列表变更后，验证：

1. `CI`
2. `Docker`
3. `安全审计`
4. `工作流健全性`
5. `发布`（当安全运行时）

需要注意的故障模式：

- `action is not allowed by policy`

如果遇到此问题，仅添加特定的受信任缺失操作，重新运行并记录原因。

最新扫描记录：

- 2026-02-21：为支持的文件类型添加了手动Vorpal reviewdog工作流，用于有针对性的安全编码检查
    - 添加允许列表模式：`Checkmarx/vorpal-reviewdog-github-action@*`
    - 工作流使用固定源：`Checkmarx/vorpal-reviewdog-github-action@8cc292f337a2f1dea581b4f4bd73852e7becb50d`（v1.2.0）
- 2026-02-17：Rust依赖缓存从`Swatinem/rust-cache`迁移到`useblacksmith/rust-cache`
    - 无需新的允许列表模式（`useblacksmith/*`已在允许列表中）
- 2026-02-16：在`release.yml`中发现隐藏依赖：`sigstore/cosign-installer@...`
    - 添加允许列表模式：`sigstore/cosign-installer@*`
- 2026-02-16：Blacksmith迁移阻止了工作流执行
    - 添加允许列表模式：`useblacksmith/*`用于自托管运行器基础设施
    - 操作：`useblacksmith/setup-docker-builder@v1`、`useblacksmith/build-push-action@v2`
- 2026-02-17：安全审计可重现性/新鲜度平衡更新
    - 添加允许列表模式：`rustsec/audit-check@*`
    - 在`security.yml`中将内联`cargo install cargo-audit`执行替换为固定的`rustsec/audit-check@69366f33c96575abad1ee0dba8212993eecbe998`
    - 取代了#588中的浮动版本提案，同时保持操作源策略明确

## 回滚

紧急解除阻塞路径：

1. 暂时将Actions策略设置回`all`。
2. 识别缺失条目后恢复选定的允许列表。
3. 记录事件和最终允许列表差异。