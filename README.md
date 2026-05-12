# @imba97/skills

个人向 AI Agent 技能集合（Cursor / 兼容 Skills 的客户端均可按各自方式挂载）。

## 技能一览

| 技能 | `name` | 路径 | 说明 |
| --- | --- | --- | --- |
| 基于暂存区的提交信息 | `git-commit` | [`skills/git-commit/SKILL.md`](skills/git-commit/SKILL.md) | 根据暂存区 diff 生成 Conventional Commits 英文 subject（≤50 字符、说明部分不加多余标点），并用 `bash` 代码块输出 `git commit -m` 命令。 |
| 代码优化 | `code-optimization` | [`skills/code-optimization/SKILL.md`](skills/code-optimization/SKILL.md) | 用 `git status` / `git diff HEAD`（或用户指定基线）梳理全部改动，从优雅性、性能、可靠性、模式与全局复用等维度列出待确认优化项（含原因与预计提升），用户勾选后再改代码。 |
| 通用视频创作引导 | `video-planning` | [`skills/video-planning/SKILL.md`](skills/video-planning/SKILL.md) | 分阶段单次一问，把想法落成故事、分镜、拍摄清单与剪辑指引；终稿含发布提示与封面图 prompt。 |

## 安装方式

使用 [skills CLI](https://github.com/imba97/skills)（通过 `pnpx` 调用，无需全局安装 CLI 本体）。

**当前仓库安装全部技能**（写入本项目约定的 skills 目录，适合单个仓库专用）：

```bash
pnpx skills add imba97/skills --skill='*'
```

**全局安装同样一批技能**（`-g`：安装到用户级，各项目/agent 默认可用）：

```bash
pnpx skills add imba97/skills --skill='*' -g
```

`--skill='*'` 表示安装源仓库中的全部技能；若只需其中一个，将 `*` 换成技能目录名即可。
