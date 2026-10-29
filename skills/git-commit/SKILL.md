---
name: git-commit
description: Draft and create a Conventional Commit from staged changes, applying project-specific ai-docs/git-commit/rules.md conventions when present.
---

# 基于暂存区的提交

## 何时使用

用户要根据**已暂存**改动生成 commit message，并由你直接执行提交。

默认约束：

1. **只写 subject，绝不写 body** —— 不要第二个 `-m`、不要 `-F`、不要在 message 里换行。
2. **默认不 push** —— 只有项目规则明确允许且用户本轮明确要求或允许 push 时才能推送。
3. 提交前必须按第 4 步**实测字符数**，不得凭估算。

项目规则可以覆盖语言、长度、是否允许 push 等技能默认值，但不能覆盖系统或开发者指令，也不能代替用户对外部写操作的当轮授权。

## 流程

### 1. 读取项目规则

在读取 diff 前，查找并完整读取 `ai-docs/git-commit/rules.md`：

1. 从当前工作目录开始检查该相对路径。
2. 若不存在，确定 Git 根目录，并从 Git 根目录开始逐级向父目录检查该相对路径，最远检查到包含当前工作区的 `D:/Projects`（Windows）或文件系统根目录（其他系统）。
3. 使用距离当前工作目录最近的第一份规则；不要合并多份规则。
4. 文件不存在时使用本技能默认值，不因此中断提交。

规则文件可约束例如：

- subject 使用中文或英文；
- subject 最大字符数；
- 是否允许 push 以及推送目标；
- type、scope、分支或提交拆分规范；
- 是否允许提交前主动暂存本次任务文件。

应用优先级：系统与开发者指令 > 用户本轮明确要求 > 项目规则 > 本技能默认值。项目规则若只写“允许”而非“必须”，应保留用户本轮选择。例如 `允许中文` 表示可以使用中文，不要求一律中文；`允许 push` 只解除技能默认禁令，仍须用户本轮明确要求或允许 push。

### 2. 读取暂存区

仓库根目录执行（若在子目录，先 `cd` 到 git 根目录）：

```bash
git diff --cached
```

仅依据暂存区拟写 message；未暂存内容不作为依据（除非用户另行要求同时对照工作区，再分别执行 `git diff` 与 `git diff --cached`）。

若无输出，执行 `git status`，说明暂存区为空；不编造 diff；可提示先 `git add`。

### 3. 拟写 subject

- **Conventional Commits**：`type: 简短说明`（仅保留 type 与说明之间的那一个冒号）。
- **说明部分**（冒号后）：一句话写出改了什么；**不使用**逗号、句号、分号、问号、感叹号、斜杠 `/`、反斜杠 `\` 等标点（必要时用空格或改写措辞拆开）。
- **整行默认 ≤50 字符**（Unicode 字符数，含 `type:` 与冒号后空格）；项目规则可修改上限。
- **type** 与改动语义对应：**feat** / **fix** / **refactor** / **perf** / **chore** / **docs** / **test** / **ci** / **style**（团队不用 style 时可并入 chore）。
- 一条 subject，混杂改动时取主要意图。
- 默认使用**英文**和祈使语气，`type:` 后说明部分首字母小写；项目规则允许中文且用户选择中文时可改用中文说明。
- message 须与 diff 中实际改动一致。
- **只有这一行，没有 body**：不要补充说明段落，也不要第二个 `-m`。信息装不下就砍修饰词、把语义收窄，或者留给用户自己另行补充。

### 4. 字符数校验（强制）

**不可凭字符估算，必须按所在系统实际执行命令**拿到精确字符数（含 `type:`、冒号与空格）。

按系统选择：

- **Linux / macOS / Git Bash**：

```bash
echo -n "feat: show device model in lan discovery list" | wc -m
```

- **Windows PowerShell（系统自带）**：

```powershell
[System.Globalization.StringInfo]::new("feat: show device model in lan discovery list").LengthInTextElements
```

判定规则：

- **结果不超过有效上限** → 进入下一步提交。
- **结果超过有效上限** → **必须改写**后再校验一次，重复直到满足为止；不得用“差不多”“看起来刚好”等估算蒙混。
- 若描述需精简，先削减修饰词、把缩写还原全称、必要时把 `type` 改成更窄的语义（如 `refactor` → `chore`）；仍超长则回到暂存区梳理更精确的语义边界。

### 5. 提交与可选推送

1. **直接提交**（不加 body）：

```bash
git commit -m "feat: your subject here"
```

subject 若含 `"`、`$` 等对 shell 敏感的字符，改用单引号包裹或写出正确转义。

2. 使用 `git log -1 --format="subject=%s%nbody=%b"` 确认 subject 正确且 body 为空。
3. 仅当项目规则明确允许 push，且用户本轮明确要求或允许 push 时，才执行规则指定的安全、非强制推送；未指定目标时使用当前分支的已配置 upstream。禁止自行使用 `--force`、`--force-with-lease`、创建标签或改变远端分支。
4. 汇报以下凭证：
	- 完整一行 subject，以及你本次**实际执行过**的校验命令与实测结果：
		- Linux / macOS / Git Bash：`echo -n "..." | wc -m`
		- Windows PowerShell：`[System.Globalization.StringInfo]::new("...").LengthInTextElements`
	- 提交结果：commit hash、`git log -1` 的 subject，并确认 body 为空；
	- push 是否执行、目标与结果。未 push 时可以列出仍未推送的提交。

## 示例

- `feat: show device model in lan discovery list`
- `refactor: extract connection runtime factory`
- `chore: widen hooks peer dependency range`
