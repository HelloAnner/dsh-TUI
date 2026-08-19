# dsh-TUI 协作工作流（本仓库专用约定）

## 仓库角色

| 远程 | 地址 | 角色 |
| --- | --- | --- |
| `origin` | `git@github.com:HelloAnner/dsh-TUI.git` | 我的 fork，功能分支推送与 PR 源 |
| `upstream` | `https://github.com/ccch1mneyyy/dsh-TUI.git` | 上游主仓库，PR 目标，分支基线 |

- 默认分支：`main`。**所有功能/修复分支一律基于 `upstream/main` 切出**，不要基于本地/origin 的 `main`（原因见下节）。
- PR 统一提交到 `upstream` 的 `main`。

## 个人文件与 PR 隔离（核心约定）

`AGENTS.md`、`.pi/`、`.worktrees/` 是个人/本地开发资产，**绝不允许出现在提交给 upstream 的 PR 里**。实现方式：

- **fork 的 `main`**（origin/main）：在上游内容之上多一个"个人资产"提交，跟踪 `AGENTS.md`、`.pi/`，`.gitignore` 额外忽略 `.worktrees/`。同步上游时用 rebase 保持该提交置顶：
  ```bash
  git checkout main && git fetch upstream
  git rebase upstream/main
  git push --force-with-lease origin main
  ```
- **功能分支**：从 `upstream/main` 切出，天然不含任何个人文件，PR diff 干净，`.gitignore` 也是上游原版。
- **`.git/info/exclude`**（仓库级、不提交、对所有 worktree 生效）已加入：
  ```
  .worktrees/
  .pi/
  .pi
  AGENTS.md
  ```
  保证 worktree 里 symlink 进来的个人文件不会被误 `git add`（`.pi` 不带斜杠一条专门匹配 symlink——symlink 在 git 眼里是文件不是目录）。
- 副作用：在 fork `main` 上，`.pi/` 下**新增**文件需 `git add -f .pi/...`（已被跟踪的文件正常 `git add`/commit 不受影响）。
- 提交 PR 前自检：`git diff upstream/main...HEAD --stat` 中不得出现 `.pi/`、`AGENTS.md`、`.gitignore`、`.worktrees/`。

## upstream 分支勘察记录（截至 2026-08-19，via `gh api repos/ccch1mneyyy/dsh-TUI/branches` + git diff）

| 分支 | 领先/落后 main | 功能 |
| --- | --- | --- |
| `main` | — | 默认分支，release 线 |
| `bot-star-history` | +1 / -0 | 机器人分支：自动更新 README 的 Star History 趋势图（`assets/star-history/*`，`[skip ci]`），无代码价值 |
| `feat/rc6-compat` | +1 / -6 | 兼容 dsh 核心 `0.1.0-rc.6` 线：peer 依赖范围放宽、契约验证双线（rc.5/rc.6）、CI 增加 rc.6 门禁（`scripts/verify-rc6-compat.mjs`） |
| `feat/double-esc-tree` | +1 / -43 | 双击 Esc 会话树替换 RewindPicker：pi 式家族树浏览/搜索/回退。大改动（68 文件，+6.9k 行），新增 `SessionTreePanel.tsx`、`dsh-adapter/sessionTree.ts`，重写 `channel.ts` |
| `dev-liangshen-easter-egg` | +1 / -76 | 梁神模式彩蛋：`/easter-egg` 开关 + 全屏滚动庆祝动画，新增 `LiangshenEasterEgg.tsx`、`easterEggPrefs.ts` 及验证脚本 |
| `ui` | +9 / -24 | 长期 UI 集成分支：ink 渲染修复（流式长输出单槽前缀缓存、sticky clamp 空白段）、`/update` 死锁防护、working-activity、v0.8.1 |
| `exp-expr` | +0 / -30 | 已过期：dsh-ecosystem-spec 以 submodule 挂载的实验，勿基于其开发 |

重新勘察：`git fetch upstream --prune` 后对分支执行 `git log --oneline upstream/main..upstream/<分支>` 与 `git diff --stat upstream/main...upstream/<分支>`。

## 标准开发工作流（修 bug / 新功能通用）

### 0. 查重：确认上游没有人在做（gh cli）

动手前先搜 upstream 的 issue、PR 和近期 commit，避免重复劳动或与在途工作冲突：

```bash
# 相关 open issue（把 <关键词> 换成功能点/报错信息）
gh search issues --repo ccch1mneyyy/dsh-TUI "<关键词>" --state open

# 相关 open PR
gh search prs --repo ccch1mneyyy/dsh-TUI "<关键词>" --state open

# 上游近期 commit 是否已修/已实现
git fetch upstream
git log --oneline --grep="<关键词>" upstream/main -20

# 顺带看 upstream 活跃分支（对照本文档的勘察表，注意是否有新分支）
gh api repos/ccch1mneyyy/dsh-TUI/branches --jq '.[].name'
```

- 已有 open PR 覆盖 → 优先 review / 接续该 PR，不另起炉灶。
- 已有 issue 认领 → PR 正文用 `Closes #N` 关联。
- upstream/main 已修 → 直接同步，不用开发。

### 1. 同步基线

```bash
git checkout main
git fetch upstream --prune
git rebase upstream/main && git push --force-with-lease origin main
```

### 2. 从 upstream/main 派生 worktree

worktree 统一放当前仓库的 `.worktrees/` 下（已 gitignore）：

```bash
# <type> = feat | fix | chore | docs；<name> 用短横线小写
git worktree add .worktrees/<name> -b <type>/<name> upstream/main

# 把个人 agent 配置 symlink 进 worktree（info/exclude 已保证不会被提交）
ln -s ../../.pi .worktrees/<name>/.pi
ln -s ../../AGENTS.md .worktrees/<name>/AGENTS.md

cd .worktrees/<name> && pnpm install
```

### 3. 开发

- 小步提交，commit message 遵循上游风格：`<type>(<scope>): <中文描述>`（参考 `git log upstream/main`）。
- 改动涉及 i18n 时同步更新中英文文案与文档（`docs/*.md` / `*.en.md` 成对）。
- 不改动 `.gitignore`，不引入个人文件。

### 4. 验证

```bash
npm run build          # 编译 + verify:build 全套门禁（boundary/contract/manifest/patch-surface/...）
npm run smoke          # 冒烟
# 涉及特定功能时运行对应 scripts/verify-*.mjs
```

### 5. 推送 fork 并提交 PR

```bash
git push -u origin <type>/<name>

gh pr create --repo ccch1mneyyy/dsh-TUI --base main \
  --head HelloAnner:<type>/<name> \
  --title "<type>(<scope>): <描述>" --body "<动机 / 方案 / 验证方式>"
```

- PR 正文写清动机、方案、验证步骤；关联 issue 用 `Closes #N`。
- CI 红了就本地复现修复后 push 同名分支，PR 自动更新。

### 6. 合并后清理

```bash
cd /Users/anner/dsh-TUI
git worktree remove .worktrees/<name>
git branch -D <type>/<name>
git fetch upstream --prune && git rebase upstream/main && git push --force-with-lease origin main
git push origin --delete <type>/<name>   # 可选
```

## 注意

- `pnpm-lock.yaml` 与 `pnpm-workspace.yaml` 属上游资产，功能分支里只在必要时改动。
- 与上游分支功能重叠时（见勘察表），先考虑 review/接续该分支，而不是另起炉灶。
