---
name: git-commit
description: 根据暂存区 git diff 生成 Conventional Commits 英文 subject（≤50 字符、说明部分不加标点），用 bash 代码块输出可复制的 git commit -m 命令。
---

# 基于暂存区的提交信息（≤50 字符）

## 何时使用

用户要根据**已暂存**改动生成 commit message，并得到可复制的 `git commit -m "..."` 命令。

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

### 3. 输出

1. 给出完整一行 subject。
2. 使用 `bash` 代码块输出可复制命令：

```bash
git commit -m "feat: your subject here"
```

subject 若含 `"`、`$` 等对 shell 敏感的字符，代码块里改用单引号包裹或写出正确转义。

需要 body 时，在同一段落用文字说明可用第二个 `-m` 或编辑器追加（body 不受 50 字符限制）。

## 示例

- `feat: show device model in lan discovery list`
- `refactor: extract connection runtime factory`
- `chore: widen hooks peer dependency range`
