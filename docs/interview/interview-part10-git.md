# Git 版本控制面试题

> 本篇覆盖 20 道高频 Git 面试题，涵盖基本概念、命令操作、协作流程与高级技巧。所有答案默认基于 Git 2.30+。

---

## 1. Git 基本概念：四个区域与四种对象

### 题目
请描述 Git 的四个工作区域（工作区、暂存区、本地仓库、远程仓库），以及四种核心对象（blob、tree、commit、tag）的作用。

### 核心答案

**四个工作区域：**

| 区域 | 说明 |
|------|------|
| **工作区（Working Directory）** | 你当前编辑文件的目录，对应 `.git` 外的实际文件 |
| **暂存区（Staging Area / Index）** | `.git/index` 文件，记录下次提交要包含的文件快照 |
| **本地仓库（Local Repository）** | `.git/objects`，保存完整的提交历史和对象数据库 |
| **远程仓库（Remote Repository）** |托管在服务器上的 Git 仓库（如 GitHub/GitLab），通过 `origin` 等别名引用 |

数据流向：`工作区 → 暂存区 → 本地仓库 → 远程仓库`

**四种核心对象（均存储在 `.git/objects` 中）：**

| 对象 | 作用 |
|------|------|
| **blob** | 文件内容快照，不含文件名（文件名属于 tree） |
| **tree** | 目录结构，指向若干 blob 或子 tree，记录路径名和权限 |
| **commit** | 指向一个 tree，记录作者、时间、提交信息及父提交 |
| **tag** | 给某个 commit 起别名，通常用于版本发布，带签名可选 |

一个 commit 对象的结构示意：
```
commit ──→ tree（根目录快照）
          ├── blob（file-a.txt 内容）
          └── tree（subdir/）
              └── blob（file-b.txt 内容）
```

### 追问方向
- `git cat-file -t <hash>` 和 `git cat-file -p <hash>` 能查看什么？
- 对象在 `.git/objects` 中是如何存储的（SHA-1 哈希前缀目录）？
- 为什么不把文件名存入 blob 而要存在 tree 里？

### 避坑提示
- 别说"暂存区就是缓存"，要明确它是 `.git/index` 文件这一具体数据结构。
- 四个对象中 **blob 不存文件名**这一点经常被忽略，面试官可能借此追问。

---

## 2. Git 常用命令对比

### 题目
请说明以下命令的区别：`git add`、`git commit`、`git push`、`git pull`、`git fetch`、`git clone`、`git rebase`、`git merge`。

### 核心答案

| 命令 | 作用 | 影响的区域 |
|------|------|-----------|
| `git add <file>` | 把文件快照放入暂存区（生成 blob，更新 index） | 工作区 → 暂存区 |
| `git commit` | 把暂存区内容创建为提交写入本地仓库 | 暂存区 → 本地仓库 |
| `git push` | 把本地仓库的分支/标签推送到远程 | 本地仓库 → 远程仓库 |
| `git pull` | `git fetch` + `git merge`，拉取并合并远程更新 | 远程 → 本地（合并） |
| `git fetch` | 下载远程仓库的更新到本地，但不合并 | 远程 → 本地（不合并） |
| `git clone` | 克隆整个远程仓库到本地（包含所有历史） | 远程 → 本地 |
| `git rebase` | 将当前分支的提交"变基"到目标分支之上，重写提交历史 | 本地仓库（重写历史） |
| `git merge` | 将两个分支的历史合并，**保留**所有提交历史 | 本地仓库 |

`git pull` = `git fetch` + `git merge`（可简化为 `--rebase` 变为 fetch + rebase）

### 追问方向
- `git pull --rebase` 和 `git pull` 什么时候该用哪个？
- `git fetch` 之后如何查看远程分支的更新？
- 为什么 `git clone` 后本地只有一个 `main/master` 分支？

### 避坑提示
- 常见错误：把 `push` 说成"推送代码到服务器"，应说"推送本地分支和提交到远程仓库"。
- `rebase` 和 `merge` 容易混淆，下一题专门对比。

---

## 3. git merge vs git rebase

### 题目
`git merge` 和 `git rebase` 有什么区别？各自适合什么场景？为什么说 rebase 不要用在公共分支？

### 核心答案

**merge：** 把两个分支的提交历史**合并**成一个，生成一个新的 merge commit，保留所有原始提交的时间顺序。

```
A---B---C  (main)
     \
      D---E  (feature)
     
git checkout main && git merge feature

A---B---C---F  (main)
     \     /
      D---E  (feature)
```

**rebase：** 把当前分支的提交"移植"到目标分支的顶部，逐个"重放"提交，**重写提交历史**。

```
A---B---C  (main)
     \
      D---E  (feature)
     
git checkout feature && git rebase main

A---B---C  (main)
          \
           D'---E'  (feature)
```

| 对比项 | merge | rebase |
|--------|-------|--------|
| 历史形态 | 保留分叉历史，体现真实合并过程 | 线性历史，更整洁 |
| 是否重写历史 | 否 | 是 |
| 产生新 commit | merge commit（两个父节点） | 重放后的新 commit（单一父节点） |
| 适用分支 | 公共分支、合并长期分支 | 本地清理、短期特性分支 |

**为什么 rebase 不要用在公共分支（多人协作的分支）？**

rebase 会**重写提交历史**，生成全新的 SHA-1。其他人已经基于旧的提交 SHA-1 做了工作，rebase 后他们的本地分支会与远程产生分叉，再次 push 时会产生冲突甚至覆盖他人的提交。公共分支（main/master、develop）的历史应该被所有人共享和信任。

**各自适用场景：**
- **merge**：合并公共分支、合并 release 分支、pull request 完成时
- **rebase**：整理本地提交（`git rebase -i`）、将 feature 分支同步到 main 最新代码

### 追问方向
- `git rebase -i` 能做哪些操作（squash、fixup、reorder、edit）？
- rebase 过程中遇到冲突怎么办？
- 已经 push 了的 commit 还能 rebase 吗？后果是什么？

### 避坑提示
- 面试中最常问"rebase 和 merge 哪个更好"——没有绝对答案，要结合团队工作流回答。
- 别说"rebase 会丢失提交"，要说"重写历史"。

---

## 4. Git 冲突的成因与解决方法

### 题目
Git 冲突是如何产生的？如何解决冲突？冲突标记的格式是什么样的？

### 核心答案

**冲突产生原因：** 当两个分支对同一文件的同一处内容进行了不同的修改，且 Git 无法自动合并时，就会产生冲突。典型场景：
- 两个分支修改了同一行代码
- 一个分支删除了文件，另一个分支修改了它
- 分支 A 和 B 都新增了同名但内容不同的函数

**冲突标记格式（出现在工作区文件中）：**

```<<<<<<< HEAD
当前分支（HEAD）的内容
=======
合并进来分支的内容
>>>>>>> feature-branch-name
```

**解决冲突的步骤：**

1. **手动编辑**：删除标记符号，保留最终正确的内容
2. **IDE / VSCode**：使用自带的 merge tool（`git mergetool`）
3. **完成合并**：删除标记 → `git add <file>` → `git commit`

如果使用 VSCode：
- 接受左侧（HEAD）/ 右侧（ incoming）/ 两者合并
- 或手动编辑后 `git add` 标记为已解决

**其他命令：**
```bash
git status          # 查看冲突文件列表
git diff            # 查看冲突详情
git mergetool       # 启动配置的合并工具
git log --merge     # 查看涉及冲突的提交
```

### 追问方向
- 如果有多个文件冲突，是一次性解决还是逐个解决？
- `git rerere` 是什么，有什么作用？
- 冲突解决一半不想合并了，能用 `git merge --abort` 取消吗？

### 避坑提示
- 不要直接删掉冲突标记不解决——这会留下无法提交的半成品状态。
- `git add` 只是标记为已解决，不是提交，必须 `git commit` 才真正完成合并。

---

## 5. Git 分支操作与 HEAD 指针

### 题目
如何创建、切换、删除、重命名 Git 分支？什么是 HEAD 指针？分支命名有哪些规范？

### 核心答案

**基本操作：**

```bash
# 创建
git branch <name>          # 基于当前分支创建（不切换）
git checkout -b <name>     # 创建并切换（新分支基于当前 HEAD）
git switch -c <name>       # 同上，modern Git 推荐用法

# 切换
git checkout <branch>
git switch <branch>

# 删除
git branch -d <branch>     # 安全删除（未合并会警告）
git branch -D <branch>     # 强制删除

# 重命名
git branch -m <new-name>   # 当前分支重命名
git branch -m <old> <new>  # 指定分支重命名
```

**HEAD 指针：** HEAD 是指向当前分支最新提交的指针，它指向的提交即"当前所在位置"。在 detached HEAD 状态（不在任何分支上）时，HEAD 直接指向某个提交。

```
HEAD → main → abc123 (最新提交)
```

可以用 `cat .git/HEAD` 查看当前 HEAD 指向。

**分支命名规范：**

| 类型 | 规范 | 示例 |
|------|------|------|
| 特性分支 | `feature/描述` | `feature/user-auth` |
| 修复分支 | `fixbug/描述` 或 `hotfix/描述` | `hotfix/login-crash` |
| 发布分支 | `release/x.y.z` | `release/2.1.0` |
| 合并请求分支 | `mr/编号` 或 `PR号` | `mr/123` |
| 避免 | 中文字符、空格、`.`、`..` | — |

### 追问方向
- `git branch -d` 和 `-D` 的区别？
- detached HEAD 状态是什么意思？能在这种状态下开发吗？
- 如何让本地分支跟踪（track）远程分支？

### 避坑提示
- 删除分支前确认好当前在哪个分支上，误删主分支后果严重。
- HEAD 是指针不是别名——可以说"HEAD 指向当前分支的最新提交"。

---

## 6. Git stash 暂存修改

### 题目
`git stash` 的用途是什么？如何暂存、恢复、删除 stash？`stash pop` 和 `stash apply` 有什么区别？

### 核心答案

**用途：** 当你需要切换分支但不想提交当前未完成的修改时，用 `git stash` 把工作区修改"藏"起来，之后再恢复。

**核心操作：**

```bash
git stash              # 暂存当前工作区和暂存区的修改（生成 stash stack）
git stash push -m "描述"  # 带消息的暂存
git stash list         # 查看所有 stash（按 LIFO 顺序）

# 恢复（两种方式）
git stash apply        # 恢复最近一次 stash，但**不删除** stash 记录
git stash apply stash@{n}  # 恢复指定 stash
git stash pop          # 恢复最近一次 stash，同时**删除** stash 记录

# 删除
git stash drop         # 删除最近一次 stash（不恢复）
git stash drop stash@{n}    # 删除指定 stash
git stash clear       # 清空所有 stash
```

**apply vs pop 的关键区别：**

| 命令 | 是否恢复 | 是否从 stash 列表删除 |
|------|---------|---------------------|
| `stash apply` | ✅ 是 | ❌ 否（保留记录） |
| `stash pop` | ✅ 是 | ✅ 是（同时删除） |

`apply` 适合需要反复恢复同一 stash 的场景（如跨分支多次复用）。

### 追问方向
- stash 能暂存被追踪文件的修改，那未跟踪（untracked）文件呢？
- `git stash -u` 和 `git stash -a` 有什么区别？
- stash 能和 `git rebase` 一起用吗？

### 避坑提示
- `git stash` 默认不包含未跟踪文件，需要 `-u`（untracked）或 `-a`（all，包括被忽略的文件）。
- 如果 stash 后在同分支继续开发并提交，原 stash 仍然存在，不会自动更新。

---

## 7. Git reset 三种模式

### 题目
`git reset` 的 `--soft`、`--mixed`、`--hard` 有什么区别？分别在什么场景使用？

### 核心答案

三种模式控制"已提交的修改"（HEAD 移动后产生的差异）去向：

```bash
git reset [--soft|--mixed|--hard] <commit>
```

| 模式 | HEAD 移动 | 暂存区（Index） | 工作区（Working Directory） |
|------|-----------|----------------|---------------------------|
| `--soft` | ✅ 移动 | ❌ 不变 | ❌ 不变 |
| `--mixed`（默认） | ✅ 移动 | ✅ 重置 | ❌ 不变 |
| `--hard` | ✅ 移动 | ✅ 重置 | ✅ 重置 |

**具体行为：**

- `--soft`：HEAD 移动到目标提交，但之前的修改仍保留在暂存区。常用于将多个 commit 合并成一个。
- `--mixed`（默认）：HEAD 移动到目标提交，暂存区也被清空，但工作区文件不变。修改以"未暂存"状态存在。
- `--hard`：HEAD、暂存区、工作区**全部**重置到目标提交状态。**不可逆，修改会丢失**。

**典型使用场景：**

```bash
# 撤销上次提交，修改仍保留在工作区（mixed 默认）
git reset HEAD~1

# 撤销上次提交，修改保留在暂存区（soft）
git reset --soft HEAD~1

# 彻底撤销，丢弃所有修改（危险！）
git reset --hard HEAD~1
```

### 追问方向
- `git reset --hard` 后发现后悔了，还能找回数据吗？（reflog）
- 如何只撤销暂存区的一部分更改（`git reset <file>`）？
- `git revert` 和 `git reset` 有什么区别？

### 避坑提示
- `--hard` 是**最危险**的 reset 操作，工作中尽量少用，用前确认工作区已提交或备份。
- `HEAD~1` 表示 HEAD 的父提交，`HEAD^` 等价，但 `HEAD~1` 在有多个父节点时更明确。

---

## 8. Git revert 撤销提交

### 题目
`git revert` 是如何撤销提交的？和 `git reset` 有什么区别？各自适用什么场景？

### 核心答案

**`git revert`**：不删除已有的提交历史，而是**生成一个新的提交**来"撤销"指定提交的变更效果。它是"反向操作"的 commit，不会改写历史。

```bash
git revert <commit>   # 撤销指定提交，生成新提交
git revert HEAD~2     # 撤销倒数第3个提交
```

效果示意：
```
原始：A → B → C → D (HEAD)
执行：git revert C
结果：A → B → C → D → C' (HEAD)  # C' 撤销了 C 的变更
```

**`git revert` vs `git reset`：**

| 对比项 | `git revert` | `git reset` |
|--------|-------------|-------------|
| 是否改写历史 | ❌ 否，生成新提交 | ✅ 是，移动 HEAD |
| 是否安全 | ✅ 安全，公共分支可用 | ⚠️ 有风险，只用于本地 |
| 远程协作场景 | ✅ 适合多人协作分支 | ❌ 不应用于已 push 的公共分支 |
| 撤销效果 | 撤销单个 commit 的变更 | 回到某个历史状态 |

**选择原则：**
- 公共分支（已 push 到远程）上撤销 → `git revert`
- 本地分支整理、取消未 push 的提交 → `git reset`

### 追问方向
- `git revert` 也能产生冲突吗？如何解决？
- 如果要撤销连续多个提交（如 B~D），可以一次 revert 吗？
- `git revert -n` 是什么？

### 避坑提示
- `revert` 只"撤销变更"，不删除提交——提交历史仍然是完整的，这在审计场景很重要。
- 说清楚"revert 是对工作量的撤销"——它重新应用了变更的反向操作。

---

## 9. Git cherry-pick 选择性合并

### 题目
什么是 `git cherry-pick`？它和 `git rebase` 有什么区别？适合什么场景？

### 核心答案

**`git cherry-pick`**：将某个（或某些）特定 commit"摘取"过来，在当前分支上生成**内容相同但 SHA-1 不同**的新提交。

```bash
git cherry-pick <commit-hash>       # pick 单个提交
git cherry-pick <commit-A>..<commit-B>  # pick 一段范围（含 A 不含 B）
git cherry-pick --continue           # 解决冲突后继续
git cherry-pick --abort               # 取消 cherry-pick
```

**与 rebase 的区别：**

| 对比项 | `cherry-pick` | `rebase` |
|--------|--------------|---------|
| 作用范围 | 单/多个特定 commit | 整个分支的提交序列 |
| 目标 | 把 commit 复制到当前分支 | 把当前分支整体移动到另一分支 |
| 历史 | 可能产生重复的变更内容（不同 SHA-1） | 重写历史，保持线性 |
| 适用场景 | 选择性 hotfix、backport、合并特定功能 | 同步整个分支的最新代码 |

**典型场景：** 你在 feature 分支上做了提交 A 和 B，希望把 A 的 hotfix 合并到 main，但不包含 B（feature 尚未完成）。

```bash
git checkout main
git cherry-pick abc123  # 把 feature 的提交 A 摘到 main
```

### 追问方向
- cherry-pick 遇到冲突怎么办？
- cherry-pick 的提交 SHA-1 为什么不同？
- 如何保留原来的 commit message（`git cherry-pick -x`）？
- 能否 cherry-pick merge commit？

### 避坑提示
- cherry-pick 生成的是"新提交"，不是"移动"——同一个变更可能在两条分支都有提交。
- cherry-pick 会保留原提交的时间戳（author date），但生成新的提交时间（committer date）。

---

## 10. Git log 与 reflog

### 题目
如何查看 Git 提交历史？`git log` 有哪些常用格式化选项？图形化查看分支？`git reflog` 和 `git log` 有什么区别？

### 核心答案

**`git log` 基本用法：**

```bash
git log                    # 默认格式，显示所有分支
git log --oneline          # 每行一个提交（简短哈希 + 标题）
git log --graph            # ASCII 图形化分支历史
git log --graph --oneline  # 组合使用
git log -n 5               # 最近 5 条
git log --author="name"    # 过滤作者
git log --since="2024-01-01"   # 时间过滤
git log --grep="fix"       # 搜索提交信息关键词
git log -p <file>          # 显示指定文件的变更历史
git log --stat             # 显示变更统计（多少文件增减）
```

**常用格式化：**

```bash
git log --format="%H|%an|%ae|%s"           # 完整哈希|作者名|邮箱|提交信息
git log --format="%Cred%h%Creset|%s"       # 带颜色的短哈希
git log --format="%d %h %s"                # 显示分支/标签
git log --all --decorate                    # 装饰所有引用的 refs
```

`.gitconfig` 配置别名：
```ini
[alias]
  lg = log --graph --oneline --decorate --all
```

**`git reflog` vs `git log`：**

| 对比项 | `git log` | `git reflog` |
|--------|----------|-------------|
| 内容 | 提交历史（commit DAG） | 本地所有 HEAD 移动记录 |
| 范围 | 当前分支的提交 | 所有操作（包括 checkout、reset、rebase） |
| 生命周期 | 随分支历史保留（可 gc） | 默认保留 90 天 |
| 用途 | 查看代码变更历史 | 恢复误删分支、找回丢失的 commit |

**典型 reflog 恢复场景：**
```bash
# 误 reset 后找回原来的 commit
git reflog
# 输出：abc123 HEAD@{1}: reset: moving to HEAD~1
#       def456 HEAD@{2}: commit: my work
git checkout def456  # 恢复到 def456
```

### 追问方向
- `git log --follow <file>` 跟踪文件重命名历史
- `git log branchA..branchB` 显示什么？
- reflog 能找回被 `git gc` 清理掉的 commit 吗？

### 避坑提示
- `reflog` 是本地记录，换克隆就消失了，不能依赖它来"同步"团队协作。
- `--graph` 显示的图形是 ASCII art，不是真正的分支图。

---

## 11. Git bisect 二分查找定位 Bug

### 题目
什么是 `git bisect`？它是如何工作的？请描述二分查找定位 bug 的完整流程。

### 核心答案

**原理：** Git bisect 通过**二分查找**在提交历史中自动定位首次引入 bug 的提交。将已知正常状态和已知有问题状态标记为端点，Git 每次取中间提交让你测试，O(log n) 次即可找到。

**完整流程：**

```bash
# 1. 启动 bisect
git bisect start

# 2. 标记当前提交为"坏"（有 bug）
git bisect bad

# 3. 标记一个已知好的提交（无 bug）
git bisect good abc123

# Git 会自动 checkout 到中间提交
# 4. 测试后告知结果
git bisect good   # 如果这个提交没问题
# 或
git bisect bad    # 如果这个提交已经有 bug

# 5. 重复步骤 4~5，Git 自动切换到下一个中间提交
# ...

# 6. Git 报告第一个"坏"提交
# bisect good|bad in '<='/'>=' commits

# 7. 结束 bisect，恢复到原分支
git bisect reset
```

**自动化 bisect（跳过手动测试）：**

```bash
git bisect start
git bisect bad HEAD
git bisect good abc123
git bisect run <test-command>
# Git 自动运行测试命令，退出码 0=good，非 0=bad
```

示例：`git bisect run make test` 或 `git bisect run npm test`

### 追问方向
- bisect 能定位哪些类型的 bug？（回归问题，非确定性 bug 效果差）
- `git bisect skip` 是什么场景用？
- bisect 过程中可以中途退出吗？

### 避坑提示
- bisect 前提是：bug 是确定性的——在同一个提交中"有 bug"的结果是稳定的。
- 如果最近多次提交都无法测试，可以 `git bisect skip` 跳过，Git 会选其他提交。

---

## 12. Git 钩子（Hooks）

### 题目
Git 钩子是什么？常见的客户端钩子（pre-commit、prepare-commit-msg、commit-msg、post-commit）和服务器端钩子分别有什么作用？如何在 CI/CD 流水线中使用？

### 核心答案

Git 钩子是 `.git/hooks/` 目录下在特定 Git 事件触发时自动执行的脚本（shell、Python 等）。

**常见客户端钩子：**

| 钩子 | 触发时机 | 典型用途 | 能否阻止提交 |
|------|---------|---------|------------|
| `pre-commit` | `git commit` 执行前，暂存区快照生成后 | 运行 lint、单元测试、格式化检查 | ✅ 可以（exit 非 0） |
| `prepare-commit-msg` | 提交信息编辑器打开前 | 提供默认模板、填充 issue 编号 | 否 |
| `commit-msg` | 提交信息写入后 | 验证提交信息格式（conventional commit） | ✅ 可以 |
| `post-commit` | 提交完成后 | 触发通知、更新 TODO 列表 | 否 |

**常见服务器端钩子：**

| 钩子 | 触发时机 | 典型用途 |
|------|---------|---------|
| `pre-receive` | 客户端 push 到服务器 refs 之前 | 验证分支策略、权限检查 |
| `update` | 每一个 refs 要更新时 | 拒绝非 fast-forward 合并 |
| `post-receive` | push 完成后 | 触发 CI/CD、发送通知、部署 |

**在 CI/CD 中的使用：**

```yaml
# .gitlab-ci.yml 或 GitHub Actions
lint-job:
  script:
    - npm run lint
    - npm run test
  hooks:
    - event: pre-commit
      tools: [eslint, jest]
```

**启用自定义钩子：**
```bash
# 在项目根目录创建 .git/hooks/
# Git 自动识别 .git/hooks/pre-commit 等脚本
```

### 追问方向
- `pre-push` 钩子存在吗？属于哪类？
- 团队成员可能跳过 hooks 吗（`--no-verify`）？如何防范？
- Husky 是做什么的？

### 避坑提示
- `.git/hooks/` 中的示例文件（`.sample` 后缀）默认不执行，需要去掉后缀并赋予执行权限。
- `--no-verify` 可以绕过 pre-commit 和 commit-msg，所以服务器端钩子才是真正的防线。

---

## 13. Git Flow 工作流

### 题目
什么是 Git Flow？请描述其分支模型、优缺点。

### 核心答案

**Git Flow**（Vincent Driessen 2010 年提出）是最成熟的 Git 分支模型，定义了一组严格的分支命名和生命周期规则。

**分支类型：**

| 分支 | 命名 | 生命周期 | 合并目标 |
|------|------|---------|---------|
| main/master | `main` | 永久 | 仅从 release/hotfix 合并 |
| develop | `develop` | 永久 | 从 feature 合并，合并到 release |
| feature/* | `feature/xxx` | 短期 | 仅合并到 develop |
| release/* | `release/x.y.z` | 短期 | 合并到 main + develop |
| hotfix/* | `hotfix/xxx` | 极短期 | 合并到 main + develop |

**流程示意：**
```
main  ────●────────────────●─────── (生产版本)
          │                   │
develop ──●───●──●──●────●───●──●── (开发主干)
          │       │    │   │
feature ──┼───────┼────┘   │
          │       │        │
release ──┴───────●────────●── (发布分支)
          │       │        │
hotfix ───┴───────●────────┴── (热修复分支)
```

**优点：**
- 分支职责清晰，适合有固定发布周期的大型团队
- 支持并行开发多个版本（多个 release 分支）
- hotfix 流程独立，不阻塞正常开发节奏

**缺点：**
- 分支过于复杂，维护成本高
- 对于持续部署的团队（CD），main + develop 双分支显得多余
- 小团队、快速迭代项目使用过重

### 追问方向
- Git Flow 的 develop 分支可以省略吗？
- feature 分支间如何共享代码？（merge develop 或 cherry-pick）
- Git Flow 和语义化版本（semver）如何配合？

### 避坑提示
- Git Flow 适合**有固定发布周期**（如每月发版）的**大型团队**。
- 持续交付的团队可能觉得它太重，GitHub Flow 或 trunk-based 更适合。

---

## 14. GitHub Flow vs GitLab Flow vs Trunk-Based Development

### 题目
请对比 GitHub Flow、GitLab Flow 和 Trunk-Based Development 三种工作流。

### 核心答案

**GitHub Flow：** 极简模型，适合 Web 服务持续部署。
- 只有 `main`（主干）+ 特性分支
- PR 评审 → 合并到 main → 直接部署
- 无 release 分支，代码即生产

```
main ───●──●──●──●──●──●──●  (PR 合并后自动部署)
          │  │
feature ──●──●
```

**GitLab Flow：** 在 GitHub Flow 基础上增加了环境分支（pre-production、production）和 release 分支策略。
- 左侧时间轴为环境分支，右侧为 release 分支
- 支持"部署到不同环境"的团队（如移动端）

```
main ───●──●──●──●──●  (主干)
          │    │
production ────●──●    (生产环境分支，落后于 main)
```

**Trunk-Based Development（TBD）：** 所有开发者共享单一主干（trunk），高频（如每天）提交到 main。
- 特性开关（feature toggle）隔离未完成功能
- 无长期特性分支，分支最多存活 1-2 天
- 高频率集成，减少合并冲突

| 对比项 | GitHub Flow | GitLab Flow | TBD |
|--------|------------|------------|-----|
| 分支数量 | 极简（main + feature） | 中等（含环境/release 分支） | 最简（单一 trunk） |
| 发布方式 | 持续部署 | 环境 promotion | 特性开关 + 持续部署 |
| 回滚难度 | 易（直接 revert/rebase） | 中（多环境需逐层回） | 易（trunk 直推） |
| 适用团队 | 小团队、CD 团队 | 中大型、复杂发布流程 | 大型高频协作团队 |
| 代码隔离 | PR 分支 | PR 分支 | 特性开关 |

### 追问方向
- TBD 中如何处理"大特性"长期无法合并的问题？
- 特性开关的优缺点？
- GitHub Flow 如何处理 hotfix？

### 避坑提示
- 选择工作流要匹配**团队规模**和**发布频率**，不是越复杂越好。
- GitHub Flow 的核心约束是：**main 分支始终可部署**。

---

## 15. SSH Key 与 Git 认证

### 题目
如何生成和配置 SSH Key 克隆 Git 仓库？HTTPS 和 SSH 认证方式有什么区别？

### 核心答案

**生成 SSH Key：**

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
# 或 RSA（传统）
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

默认保存在 `~/.ssh/id_ed25519`（公钥 `id_ed25519.pub`）。

**添加到 SSH Agent：**

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

**在 GitHub/GitLab 添加公钥：**
复制 `~/.ssh/id_ed25519.pub` 内容，粘贴到平台 Settings → SSH Keys。

**克隆方式对比：**

| 对比项 | HTTPS | SSH |
|--------|-------|-----|
| 地址格式 | `https://github.com/user/repo.git` | `git@github.com:user/repo.git` |
| 认证方式 | 用户名 + Token / 密码 | 公私钥对（自动验证） |
| 每次操作需输入 | Token 或密码 | 不需要（SSH Agent 持有密钥） |
| 防火墙兼容性 | 通常可通行 | 需要 22 端口开放 |
| Token 过期 | 需要定期刷新 | 永久（除非撤销） |
| 私有仓库 | 需要个人 Token | 需要部署公钥 |

**典型 SSH 配置（~/.ssh/config）：**

```config
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    AddKeysToAgent yes
```

### 追问方向
- 为什么推荐使用 `ed25519` 而非 `rsa`？
- SSH Key 克隆报错 "Permission denied (publickey)" 如何排查？
- 多平台（GitHub + GitLab）如何配置不同 SSH Key？

### 避坑提示
- 不要在 SSH Key 生成时设置密码（passphrase）除非你能接受每次 ssh-agent 询问。
- HTTPS clone 后可以改用 SSH remote：`git remote set-url origin git@github.com:user/repo.git`

---

## 16. .gitignore 规则与常见问题

### 题目
`.gitignore` 为什么不生效？如何编写 .gitignore 规则？如何强制添加已忽略的文件？

### 核心答案

**.gitignore 生效的前提：**
1. 文件在 `.gitignore` 规则**写入之前**未被 Git 跟踪（untracked）
2. 文件确实符合 `.gitignore` 的模式匹配
3. `.gitignore` 文件本身在仓库中，且被提交

**常见原因导致不生效：**

| 原因 | 解释 | 解决方法 |
|------|------|---------|
| 文件已被 tracked | 文件在 `.gitignore` 前已提交过 | 需要先 `git rm --cached` 移除跟踪 |
| 缓存未清除 | Git 对已跟踪文件的变化有缓存 | `git rm -r --cached .` 清除缓存 |
| 规则写错 | 模式语法错误 | 确认 glob 模式正确 |
| 全局 `.gitignore` 冲突 | `core.excludesFile` 配置了全局忽略 | 检查 `git config --global core.excludesFile` |

**`.gitignore` 规则写法：**

```
# 注释
*.log           # 忽略所有 .log 文件
!error.log     # 但不忽略 error.log（取反）
node_modules/  # 忽略整个目录
build/         # 忽略 build 目录
.DS_Store      # 忽略特定文件
/root.txt      # 仅仓库根目录的 root.txt
dir/*.txt      # dir 目录下的 .txt 文件
**/temp/       # 任意层级下的 temp 目录
```

**强制添加已忽略文件：**

```bash
git add -f <file>     # 强制添加（--force）
git check-ignore -v <file>  # 调试：显示哪个规则忽略了该文件
```

### 追问方向
- 全局 `.gitignore` 和项目级 `.gitignore` 的优先级？
- `.gitignore` 中的 `/` 前缀和 `**/` 有什么区别？
- 如何让 Git 忽略文件权限变更（`filemode`）？

### 避坑提示
- 写 `.gitignore` 前先问：文件是否已经提交过？已提交的文件必须先 `git rm --cached`。
- 用 `git check-ignore -v` 调试规则，不要盲目猜测。

---

## 17. Git 子模块（submodule）

### 题目
什么是 Git 子模块？如何添加、更新、删除子模块？子模块适合什么场景？

### 核心答案

**定义：** Git 子模块（submodule）允许在一个 Git 仓库中嵌入另一个独立的 Git 仓库，被嵌入的仓库称为"子模块"。

**添加子模块：**

```bash
git submodule add https://github.com/user/lib.git libs/lib
# 创建 .gitmodules 配置文件
```

`.gitmodules` 内容示例：
```ini
[submodule "libs/lib"]
    path = libs/lib
    url = https://github.com/user/lib.git
```

**克隆带子模块的仓库：**

```bash
git clone --recurse-submodules <url>
# 或先克隆，再初始化
git clone <url>
git submodule init
git submodule update
```

**更新子模块：**

```bash
cd libs/lib
git checkout v1.2.0
cd ../..
git add libs/lib
git commit -m "update submodule to v1.2.0"
```

**删除子模块（完整流程）：**

```bash
# 1. 反初始化
git submodule deinit -f libs/lib

# 2. 从 .git/modules 中移除
git rm -f libs/lib

# 3. 提交
git commit -m "remove submodule"
```

**适用场景：**
- 第三方库独立版本管理（如 UI 组件库、工具库）
- 需要精确控制版本、不想用包管理器的场景
- 多项目共享同一底层依赖

**缺点：**
- 操作繁琐（clone、update 都需要额外命令）
- 容易出现子模块"游离"状态（指向旧提交）
- CI/CD 配置复杂

### 追问方向
- 子模块的 HEAD 游离了（detached HEAD）怎么修复？
- 如何让子模块自动更新到远程最新？
- submodule 和 subtree（subtree merge）有什么区别？

### 避坑提示
- 子模块状态是**指向特定 commit**，不是分支——这一点最容易引发困惑。
- 多人协作时，子模块更新后其他人 pull 代码需要额外 `git submodule update` 才会同步。

---

## 18. Git bisect 自动化定位

### 题目
如何实现 `git bisect` 的自动化定位？请描述完整流程以及常见的自动化测试命令写法。

### 核心答案

自动化 bisect 使用 `git bisect run <command>`，Git 执行命令后根据退出码判断（0=good，非0=bad）。

**前置条件：**
- 有一个可以自动化执行的测试脚本/命令
- 测试结果必须是确定性的

**完整流程：**

```bash
# 1. 启动 bisect 并标记端点
git bisect start

# 2. 标记当前 HEAD（有 bug）
git bisect bad

# 3. 标记一个已知好的版本
git bisect good v1.0.0

# 4. 运行自动化测试
git bisect run make test
# 或
git bisect run npm run integration-test
# 或
git bisect run python test_suite.py
```

**退出码约定：**
- `0` — 测试通过（good）
- `1~127` — 测试失败（bad）
- `125` — 跳过该提交（无法测试）
- `127` — 命令未找到

**编写测试脚本示例（Python）：**

```python
# test_bug.py
import subprocess
import sys

result = subprocess.run(["python", "app.py"], capture_output=True)
if "expected output" in result.stdout.decode():
    sys.exit(0)  # good
else:
    sys.exit(1)  # bad
```

```bash
git bisect run python test_bug.py
```

**典型 CI 集成（GitHub Actions）：**

```yaml
bisect:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v3
    - name: bisect
      run: |
        git bisect start
        git bisect bad HEAD
        git bisect good ${{ env.KNOWN_GOOD_COMMIT }}
        git bisect run make test
```

### 追问方向
- 测试脚本不确定（有时通过有时失败）bisect 还能用吗？
- 如何在 bisect 过程中跳过无法测试的提交？
- `git bisect replay` 是什么？

### 避坑提示
- 自动化 bisect 的核心是**测试命令退出码**必须准确反映"是否包含 bug"。
- 如果测试框架本身有 bug，bisect 的结果也会不准确。

---

## 19. Git 性能优化：大型仓库处理

### 题目
如何处理大型 Git 仓库的克隆性能问题？`--depth` 浅克隆有什么限制？`git fetch` 时如何获取所有 ref？

### 核心答案

**大型仓库的性能瓶颈：**
- 完整克隆历史巨大（Linux kernel 仓库 > 20GB）
- 全量克隆耗时长、占用大量磁盘

**解决方案：**

**1. 浅克隆（--depth）：**

```bash
git clone --depth 1 https://github.com/user/large-repo.git
# 仅克隆最近一次提交，下载量大幅减少
```

**限制：**
- 没有完整的提交历史，`git log` 仅显示浅历史
- 无法 `git merge` 来自完整仓库的分支（会报 `shallow repository` 错误）
- `git pull` 可能失败，需要 `git fetch --unshallow` 转为完整仓库
- 无法查看和 checkout 浅克隆不包含的历史 commit

**2. 克隆特定分支：**

```bash
git clone --depth 1 --branch main --single-branch https://github.com/user/repo.git
```

**3. 稀疏检出（sparse checkout）：**

```bash
git clone --no-checkout https://github.com/user/repo.git
cd repo
git sparse-checkout set src/components
git checkout main
# 只检出 src/components 目录，不含其他文件
```

**4. fetch 所有 ref：**

```bash
git fetch --all
# 默认会下载所有分支和标签的更新
git fetch origin --tags --force
```

**5. 增量更新（不需要完整克隆）：**

```bash
git init
git remote add origin https://github.com/user/repo.git
git fetch --depth 1 origin main
git checkout main
```

**`git fetch --unshallow`** 将浅克隆转为完整仓库，但需要远程支持。

### 追问方向
- 浅克隆的 `.git/shallow` 文件是什么作用？
- `git clone --mirror` 和 `--bare` 有什么区别？
- 大型仓库中 `git gc` 和 `git repack` 有什么作用？

### 避坑提示
- `--depth 1` 克隆后不要急于做 `git merge` 远程分支，浅克隆的合并能力受限。
- 稀疏检出（sparse-checkout）是处理 monorepo 的利器，值得掌握。

---

## 20. Git 协同开发：PR / MR 流程与 Code Review

### 题目
描述 Pull Request（PR）和 Merge Request（MR）的完整流程，以及 Code Review 的关键环节。

### 核心答案

**PR/MR 本质相同：** GitHub 用 Pull Request，GitLab 用 Merge Request。都是"请他人审阅后将我的分支合并到目标分支"的协作机制。

**完整协作流程：**

```
1. fork（可选，GitHub 公开项目常用）
         ↓
2. 从 main 创建特性分支
         ↓
3. 在特性分支开发并提交
         ↓
4. push 到自己仓库的远程分支
         ↓
5. 发起 PR/MR（指向目标分支 main）
         ↓
6. Code Review + 讨论
         ↓
7. 根据反馈修改（ amend / new commit）
         ↓
8. 合并（merge）到目标分支
         ↓
9. 删除特性分支
```

**Code Review 关键环节：**

| 环节 | 说明 |
|------|------|
| **创建 PR** | 填写标题、描述、关联 Issue、标注 Reviewer |
| **Reviewer 检查** | 逻辑正确性、代码风格、安全漏洞、测试覆盖 |
| **讨论** | 在具体代码行下评论（inline comment） |
| **修改** | 作者根据反馈提交修复（鼓励新 commit，而非 force push） |
| **批准（Approve）** | Reviewer 同意合并 |
| **合并** | 方式可选：merge commit / squash / rebase |
| **清理** | 删除源分支 |

**Merge 策略选择：**

| 策略 | 命令 | 效果 | 适用场景 |
|------|------|------|---------|
| Merge commit | `git merge --no-ff` | 保留完整分支历史 | 公共分支、团队需要追溯合并来源 |
| Squash merge | `git merge --squash` | 所有 commits 合并为一个 | feature 分支提交记录零碎 |
| Rebase + merge | `git rebase` + merge | 线性历史 | 保持干净的 main 历史 |

**PR 描述模板（建议）：**

```markdown
## 改动概述
<!-- 一句话描述改了什么 -->

## 改动原因
<!-- 为什么需要这个改动 -->

## 影响的范围
<!-- 哪些文件/模块会受影响 -->

## 测试方式
<!-- 如何验证这个改动没问题 -->

## 截图 / 录屏（UI 改动）
```

### 追问方向
- PR 合并前需要至少几个 Approve？
- 合并后发现问题如何回滚？（`git revert` 或 `git reset`）
- GitHub 的 "Draft PR" 是什么？和普通 PR 有什么区别？
- 如何处理长期不活跃的 PR？

### 避坑提示
- PR 的 scope 不要太大（建议 < 400 行改动），太大的 PR 难以有效 Review。
- 不要在 PR 合并后 force push 源分支（除非你和团队约定好了）。

---

## 附录：面试准备清单

| 主题 | 核心命令 | 常考点 |
|------|---------|-------|
| 区域与对象 | `git cat-file`、`git ls-tree` | 四种对象的存储关系 |
| 合并策略 | `merge`/`rebase`/`cherry-pick` | 何时用哪个，历史影响 |
| 撤销操作 | `reset`/`revert`/`reflog` | 公共分支 vs 本地分支 |
| 分支管理 | `branch`/`checkout`/`switch` | HEAD 指针、detached HEAD |
| 协作流程 | PR/MR + Code Review | 合并策略、Review 规范 |
| 性能 | `--depth`/`sparse-checkout`/`gc` | 大型仓库处理方案 |

---

*面试建议：Git 面试的核心在于"理解原理 + 场景判断"。死记命令不够，必须理解每条命令对 Git 对象、HEAD、ref 的实际影响。遇到"为什么"的问题，从"Git 是一棵提交 DAG"这个本质出发推导答案。*
