---
name: publish-checklist
description: Pre-publish check for frontend packages — verify and auto-fix required metadata.
---

# 发布前检查

## 何时使用

发布一个**前端依赖包**（或准备在 CI 里 publish）之前，需要快速确认**会被工具链或包平台读取**的关键元数据齐全且正确，避免类似：

```
Error: No pnpm version is specified.
Please specify it by one of the following ways:
  - in the GitHub Action config with the key "version"
  - in the package.json with the key "packageManager"
  - in the package.json with the key "devEngines.packageManager"
```

这类「工具链静默失败 / GitHub 页面缺信息 / npm 页面缺链接」的问题。

**不适用**：

- 服务端、CLI 等非"前端依赖"项目（字段集与规则可能不同）。
- 已有完整发布流水线与 lint 规则的项目，本技能可作为补充而不是替代。
- 想做**版本号 / changelog / tag** 类检查，请用其它技能。

## 检查清单

围绕「工具链/平台必读 + 维护者手动友好」两类，输出下列字段现状与是否需要修复：

| # | 字段 | 必备性 | 缺失或错误时的影响 | 取值建议 |
| --- | --- | --- | --- | --- |
| 1 | `packageManager` | 强烈建议 | `pnpm/action-setup`、`corepack` 等无法确定包管理器版本，CI 失败（见上面报错） | `"pnpm@9.12.3"` 等 `name@version` 字符串；版本号与团队一致 |
| 2 | `repository` | 必备 | npm/GitHub 页面无源码链接；`npm publish` 在某些策略下也会报错 | `{ "type": "git", "url": "git+https://github.com/<owner>/<repo>.git" }` |
| 3 | `homepage` | 必备 | 包页缺项目主页链接 | `"https://github.com/<owner>/<repo>#readme"`（仓库型）或文档站 |
| 4 | `bugs` | 建议 | 包页缺 issue 入口 | `{ "url": "https://github.com/<owner>/<repo>/issues" }` |
| 5 | `license` | 必备 | npm 警告 + 包页缺协议声明 | `"MIT"` 等 SPDX 标识，或 `{ "type": "MIT", "url": "..." }` |
| 6 | `author` | 建议 | 包页归属空白 | 字符串或 `{ "name", "email", "url" }` |
| 7 | `engines.node` | 建议 | 用户装了错版本 Node 不会得到友好提示 | `"^18.0.0" \|\| ">=20"` 等 |
| 8 | `files` | 可选 | 误把 `node_modules`、`.git`、源码全部发包 | `["dist"]` 等白名单 |

> 本技能**当前版本**只对前 5 项（`packageManager` / `repository` / `homepage` / `bugs` / `license`）做"检查 + 自动修复"。其余项只报告、不改。

## 工作流

### 步骤 1：定位元数据文件

1. 当前目录就是包根目录 → 直接读取 `./package.json`。
2. 单仓多包（`packages/<name>`）→ 先问用户要检查哪一个包，或读取全部并分别报告。
3. monorepo 工具脚本要发**根包** → 读取仓库根 `package.json`。
4. 未来扩展到其它载体（如 `.npmrc`、CI 配置、`LICENSE` 文件等）时，按各自规则定位。

定位方式：

- 工作区有 git：先用 `git rev-parse --show-toplevel` 拿到仓库根，再按目录层级走。
- 没有 git：直接读用户给定路径。

### 2. 读取并解析

读取对应元数据文件，用合适的解析器得到结构化对象。JSON 载体用 `JSON.parse`，避免字符串匹配，避免把注释、字段顺序混淆。

### 3. 逐项检查

按「检查清单」表格中编号 1–5 的字段：

- 字段**存在且值合法** → 该项 ✅
- 字段**缺失或空字符串** → 该项 ❌ 缺失
- 字段**类型不对**（例如 `license` 是对象但缺 `type`） → 该项 ⚠️ 不规范

判定细则：

- `repository`：`type === 'git'` 且 `url` 是合法 git URL（`git+https://...`、`https://github.com/.../.git`、`git@github.com:...`）。
- `bugs`：`string` 或 `{ url }`，URL 是合法的 issues URL。
- `homepage`：合法 URL。
- `license`：`string`（SPDX）或对象含 `type`。
- `packageManager`：形如 `<name>@<version>`，`<name>` ∈ `{ npm, pnpm, yarn, bun }`。

### 4. 自动修复（用户授权后）

输出**修复前 vs 修复后**的 diff，逐字段让用户确认或一次性确认。修复规则：

- `packageManager` 缺失：默认填 `pnpm@<用户当前 pnpm 版本>`。版本用 `pnpm --version` 现读，不要猜测。如果用户机器没有 pnpm，询问其团队统一版本。
- `repository` 缺失：从 git 远程推断。
  - 执行 `git remote get-url origin`，去除 `git@github.com:` 前缀、补 `.git`、加 `git+` 前缀。
  - 拿不到远程（无 git / 无 origin）→ **询问用户** owner 与 repo，不再瞎填。
- `homepage` 缺失：基于 `repository` 生成 `https://github.com/<owner>/<repo>#readme`。
- `bugs` 缺失：基于 `repository` 生成 `https://github.com/<owner>/<repo>/issues`。
- `license` 缺失：默认 `MIT`。若仓库已有 `LICENSE` 文件且首行符合 SPDX 头（如 `MIT License`），按文件内容填。

写入策略：

- JSON 载体：`JSON.parse` → 修改字段 → `JSON.stringify(obj, null, 2)` 写回。
- **保留原有缩进（2 或 4 空格）和字段顺序**：先把原文件按行拆，再逐字段重写，比 stringify 更稳。若你选用 stringify，请在最后用 `prettier`/`dprint` 跑一次以匹配仓库风格。
- 写完后再解析一次，确认合法。

### 5. 给出报告

最终输出一个表格：

| 字段 | 状态 | 修复前 | 修复后 |
| --- | --- | --- | --- |

并附：

- 改动的 `git diff`（如果是仓库内修改）。
- 还遗留的 ⚠️ 项与建议。

## 不做的事

- **不**改 `name`、`version`、`main`/`exports`、`dependencies` 等业务字段——那是发布版本号与构建产物的事，不是元数据。
- **不**自动跑 `npm publish` / `pnpm publish` / `npm version`。
- **不**写 CHANGELOG、不打 tag、不 commit；按需提示用户另用相关技能。
- **不**自作主张改 `license` 为 `MIT` 而不告知用户；这是法律含义字段，必须显式确认。
- **不**在没有 git 远程、没有 LICENSE 文件的情况下"猜" `repository` / `license` 的 owner 或协议——必须问。

## 示例

### 输出示例（部分缺失）

> 仓库：`imba97/bilibili-toy`
> 读取：`./package.json`

| 字段 | 状态 | 修复前 | 修复后 |
| --- | --- | --- | --- |
| `packageManager` | ❌ 缺失 | — | `"pnpm@9.12.3"` |
| `repository` | ❌ 缺失 | — | `{ "type": "git", "url": "git+https://github.com/imba97/bilibili-toy.git" }` |
| `homepage` | ❌ 缺失 | — | `"https://github.com/imba97/bilibili-toy#readme"` |
| `bugs` | ❌ 缺失 | — | `{ "url": "https://github.com/imba97/bilibili-toy/issues" }` |
| `license` | ❌ 缺失 | — | `"MIT"` |

确认是否应用？默认 ✅ 等用户回 "是 / 跳过 license / 全应用"。

### 全部齐全

直接给一句话："元数据检查通过，5 项全部 ✅，无需改动。"

## 相关

- `pnpm/action-setup` 文档：`packageManager` 是其推断 pnpm 版本的三种方式之一。
- npm package.json 官方字段说明：<https://docs.npmjs.com/cli/v10/configuring-npm/package-json>
- SPDX 协议列表：<https://spdx.org/licenses/>
