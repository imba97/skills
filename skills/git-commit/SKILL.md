---
name: git-commit
description: Draft a Conventional Commits subject (≤50 chars, counted) from the staged diff, commit it directly with no body, and never push.
---

# 基于暂存区的提交（subject ≤50 字符，无 body，不 push）

## 何时使用

用户要根据**已暂存**改动生成 commit message，**并由你直接执行提交**。

三条硬性约束，任何情况下都不例外：

1. **只写 subject，绝不写 body** —— 不要第二个 `-m`、不要 `-F`、不要在 message 里换行。
2. **绝不 push** —— 不执行 `git push`、`git push --follow-tags`、`gh` 等任何推送动作；推不推、什么时候推，由用户自己决定。
3. 提交前必须按第 3 步**实测字符数**，不得凭估算。

## 流程

### 1. 读取暂存区

仓库根目录执行（若在子目录，先 `cd` 到 git 根目录）：

```bash
git diff --cached
```

仅依据暂存区拟写 message；未暂存内容不作为依据（除非用户另行要求同时对照工作区，再分别执行 `git diff` 与 `git diff --cached`）。

若无输出，执行 `git status`，说明暂存区为空；不编造 diff；可提示先 `git add`。

### 2. 拟写 subject

- **Conventional Commits**：`type: 简短说明`（仅保留 type 与说明之间的那一个冒号）。
- **说明部分**（冒号后）：一句话写出改了什么；**不使用**逗号、句号、分号、问号、感叹号、斜杠 `/`、反斜杠 `\` 等标点（必要时用空格或改写措辞拆开）。
- **整行 ≤50 字符**（Unicode 字符数，含 `type:` 与冒号后空格）。
- **type** 与改动语义对应：**feat** / **fix** / **refactor** / **perf** / **chore** / **docs** / **test** / **ci** / **style**（团队不用 style 时可并入 chore）。
- 一条 subject，混杂改动时取主要意图。
- **英文**，祈使语气，`type:` 后说明部分首字母小写；专有名词、缩写保持合理大小写。
- message 须与 diff 中实际改动一致。
- **只有这一行，没有 body**：不要补充说明段落，也不要第二个 `-m`。信息装不下就砍修饰词、把语义收窄，或者留给用户自己另行补充。

### 3. 字符数校验（强制）

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

- **结果 ≤ 50** → 进入下一步提交。
- **结果 > 50** → **必须改写**到 ≤50 后再校验一次，重复直到满足为止；不得用"差不多"、"看起来刚好"等估算蒙混。
- 若描述需精简，先削减修饰词、把缩写还原全称、必要时把 `type` 改成更窄的语义（如 `refactor` → `chore`）；仍超长则回到暂存区梳理更精确的语义边界。

### 4. 提交与汇报

1. **直接提交**（不加 body、不 push）：

```bash
git commit -m "feat: your subject here"
```

subject 若含 `"`、`$` 等对 shell 敏感的字符，改用单引号包裹或写出正确转义。

2. 汇报两件事，作为凭证：
	- 完整一行 subject，以及你本次**实际执行过**的校验命令与实测结果：
		- Linux / macOS / Git Bash：`echo -n "..." | wc -m`
		- Windows PowerShell：`[System.Globalization.StringInfo]::new("...").LengthInTextElements`
	- 提交结果：`git log -1` 的 subject，并确认 body 为空。
3. **明确说明没有 push**：可以列出仍未推送的提交（如 `git log --oneline origin/main..HEAD`），把推送的决定留给用户。

## 示例

- `feat: show device model in lan discovery list`
- `refactor: extract connection runtime factory`
- `chore: widen hooks peer dependency range`
