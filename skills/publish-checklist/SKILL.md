---
name: publish-checklist
description: Pre-publish check for frontend packages — verify and auto-fix required metadata.
---

# 发布前检查

## 何时使用

发布一个**前端依赖包**（或准备在 CI 里 publish）之前，快速确认**会被工具链或包平台读取**的关键元数据齐全且正确，避免：

```
Error: No pnpm version is specified.
  - in the GitHub Action config with the key "version"
  - in the package.json with the key "packageManager"
```

或双源冲突：

```
Error: Multiple versions of pnpm specified:
  - version 12 in the GitHub Action config with the key "version"
  - version pnpm@12.8.1 in the package.json with the key "packageManager"
```

或 OIDC publish 缺通行证：

```
ERR_PNPM_ID_TOKEN_GITHUB_WORKFLOW_INCORRECT_PERMISSIONS: Incorrect permissions for idToken within GitHub Workflows
[WARN] Failed to replace env in config: ${NODE_AUTH_TOKEN} in .npmrc key "_authToken"
```

**不适用**：服务端 / CLI 项目；已有完整 lint 规则的项目；版本号 / changelog / tag 检查请用其它技能。

## 检查清单

| # | 字段 | 必备性 | 缺失或错误时的影响 | 取值 |
| --- | --- | --- | --- | --- |
| 1 | `packageManager` | 强烈建议 | `pnpm/action-setup` / `corepack` 无法推断版本，CI 失败 | `"pnpm@12.8.1"` |
| 2 | `repository` | 必备 | npm 页无源码链接 | `{ "type": "git", "url": "git+https://github.com/<o>/<r>.git" }` |
| 3 | `homepage` | 必备 | 包页缺主页 | `"https://github.com/<o>/<r>#readme"` |
| 4 | `bugs` | 建议 | 包页缺 issue | `{ "url": "https://github.com/<o>/<r>/issues" }` |
| 5 | `license` | 必备 | npm 警告 + 页缺协议 | `"MIT"` 等 SPDX |
| 6 | `author` | 建议 | 页归属空白 | 字符串或 `{ name, email, url }` |
| 7 | `engines.node` | 建议 | Node 版本无友好提示 | `"^20 \|\| >=22"` |
| 8 | `files` | 可选 | 误把源码全发 | `["dist"]` |
| 9 | CI ↔ manifest 一致性（workflow `pnpm/action-setup` ↔ `package.json#packageManager`） | 强烈建议 | 双源 → `Multiple versions of pnpm specified` | **二选一**：仅 workflow 钉；或仅 manifest 钉 |
| 9a | OIDC 通行证（`permissions: id-token: write`） | 强烈建议（仅 Trusted Publisher） | OIDC 被拒 + pnpm fallback 失败 | publish job 下加 `id-token: write`（job 级） |
| 9b | Trusted Publisher filename 一致性（npm 后台 ↔ `.github/workflows/`） | 强烈建议（仅 Trusted Publisher） | OIDC 验证被拒 / publish 401 | 以仓库文件名为准，npm 后台改它 |

> 本版本只对 1–5 做"检查 + 自动修复"。其余只报告。

## 工作流

### 步骤 1：定位

1. `./package.json`（或 `packages/*/package.json`）。
2. 仓库根的 `package.json`（monorepo 根包）。
3. `.github/workflows/*.yml` / `*.yaml`（用于 #9 / 9a / 9b）。

### 步骤 2：解析

JSON 用 `JSON.parse`，避免字符串匹配。

### 步骤 3：逐项检查

按 #1–5、#9–9b：

- 存在且合法 → ✅
- 缺失 / 空 → ❌
- 类型不对 → ⚠️

判定细则：

- `repository`：`type === 'git'` 且 url 合法。
- `packageManager`：`<name>@<version>`，`<name>` ∈ `{ npm, pnpm, yarn, bun }`。
- `#9`：workflow 同时设了 `version` 又在 manifest 设 `packageManager` → ❌。
- `#9a`：publish job 缺 `id-token: write` → ❌。注意 job 级权限覆盖 top-level。
- `#9b`：npm Trusted Publisher 配置的 `Workflow filename` 与仓库 `.github/workflows/` 下文件名不一致 → ❌。

### 步骤 4：自动修复

- 1–5 缺失：按上表"取值"补齐；`repository` / `homepage` / `bugs` 从 `git remote get-url origin` 推断；`license` 默认 `MIT`（与 LICENSE 文件首行冲突时按文件）。
- `#9`：默认保留 `package.json#packageManager`，删 workflow 的 `with.version`。
- `#9a`：默认加 `permissions: { id-token: write, contents: read }` 到 publish job；若用户走传统 NPM_TOKEN 路线（`.npmrc` + `NODE_AUTH_TOKEN`），**不**加 `id-token: write` 而在 repo secrets 设 `NPM_TOKEN`。两条路线二选一。
- `#9b`：**不在仓库内改** — 仓库文件名是源头，npm 后台只能人工改。

**写入**：JSON 载体用 `JSON.parse` → 改 → `JSON.stringify(obj, null, 2)`；保留原缩进与字段顺序；写完再 `JSON.parse` 验证。

**不**自动跑 publish / commit / tag。

### 步骤 5：报告

输出表格：

| 字段 | 状态 | 修复前 | 修复后 |

附 `git diff` 与遗留 ⚠️ 项。

## 不做的事

- 不改 `name` / `version` / `main` / `dependencies` 等业务字段。
- 不自动 `npm publish` / `pnpm publish` / `npm version` / commit / tag。
- 不擅自把 `license` 改成 `MIT` — 法律含义字段必须显式确认。
- 不在没有 git 远程 / LICENSE 文件时"猜" `repository` / `license` — 必须问。

## 示例

### 元数据缺失

> `imba97/bilibili-toy` `./package.json`

| 字段 | 状态 | 修复前 | 修复后 |
| --- | --- | --- | --- |
| `packageManager` | ❌ | — | `"pnpm@9.12.3"` |
| `repository` | ❌ | — | `{ "type": "git", "url": "git+https://github.com/imba97/bilibili-toy.git" }` |
| `homepage` | ❌ | — | `"https://github.com/imba97/bilibili-toy#readme"` |
| `bugs` | ❌ | — | `{ "url": "https://github.com/imba97/bilibili-toy/issues" }` |
| `license` | ❌ | — | `"MIT"` |

### CI 双源冲突（#9）

| 字段 | 状态 | 修复前 | 修复后 |
| --- | --- | --- | --- |
| `#9` | ❌ | workflow `version: 12` + manifest `packageManager: "pnpm@12.8.1"` | 删 workflow `version`；manifest 保留 |

### OIDC 通行证缺失（#9a）

| 字段 | 状态 | 修复前 | 修复后 |
| --- | --- | --- | --- |
| `#9a` | ❌ | publish job 无 `permissions:` | `permissions: { id-token: write, contents: read }` |

### Trusted Publisher filename 不一致（#9b）

| 字段 | 状态 | 修复前 | 修复后 |
| --- | --- | --- | --- |
| `#9b` | ❌ | npm 后台 `release.yml`，仓库 `release.yaml` | **仓库不改**：去 npm 站点把 `release.yml` 改成 `release.yaml` |

## 相关

- `pnpm/action-setup`：v4+ 自动从 `package.json#packageManager` 读版本。<https://github.com/pnpm/action-setup>
- pnpm Trusted Publisher / OIDC：<https://pnpm.io/npmrc>
- npm Trusted Publishers：<https://docs.npmjs.com/generating-and-trusting-tokens/trusted-publishers>
- npm package.json 字段：<https://docs.npmjs.com/cli/v10/configuring-npm/package-json>
- SPDX 列表：<https://spdx.org/licenses/>