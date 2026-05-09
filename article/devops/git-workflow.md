# Git 工作流深入 —— 从对象模型到 Trunk-based 实战

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [DevOps 速记]() 里 Git 那一节够了。
> 这里要回答的，是**误 push 强覆盖了同事代码、rebase 解冲突丢代码、半夜被"主干被破坏"告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. Git 的 4 种对象（blob / tree / commit / tag）分别存什么？为什么 Git 是个 "content-addressable storage"，而不是 "snapshot diff"？同一份文件被 100 个 commit 引用，磁盘上存几份？
2. `git rebase main` 和 `git merge main` 在**最终代码内容**上能完全等价吗？区别在 **commit 拓扑** 上具体是什么？为什么团队 Code Review 的体感差距巨大？
3. 你 `git reset --hard` 然后 `git gc --prune=now --aggressive`，原来的 commit 还能用 `git reflog` 救回来吗？reflog 救命的边界到底在哪？
4. squash 一段 10 个 commit 的 feature 分支看起来很干净，但**代价**是什么？bisect 的时候你会哭吗？
5. `git cherry-pick A B C` 和 `git rebase --onto target base..C` 在效果上等价吗？什么场景下后者必须用前者顶不上？
6. Git Flow / GitHub Flow / GitLab Flow / Trunk-based —— 选错的代价是什么？团队规模 / 发布节奏 / 多版本并行三个维度怎么决定？
7. 一个 200MB 的 PSD 不小心 commit 进了仓库，下个迭代你删掉了它。仓库会变小吗？为什么？怎么真正清掉？LFS 在底层怎么实现的？
8. Monorepo（Bazel / Nx / Lerna）vs 多仓 polyrepo —— 不只是"放一起还是分开"，哪些工程问题是 monorepo **必然要解决** 的？哪些是它**反而引入** 的？

如果有半数答不上，往下看。

---

## 1. 底层模型：Git 是个内容寻址数据库

绝大多数人对 Git 的认知止于"add / commit / push / pull"四件套。这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到 `.git/` 目录的物理结构。

### 1.1 Git 是个 content-addressable storage

Git 不存"文件的修改历史"，它存**文件的全部历史快照**，靠**内容寻址**去重。

```bash
# 找一个 git 仓库，做几个实验
echo "hello git" > a.txt
git hash-object a.txt
# 输出：8d0e41234f24b6da002d962a26c2495ea16a425f
```

这个 40 位 hex 字符串就是该文件内容的 **SHA-1**（新版本可选 SHA-256）。它是**整个 Git 系统的"主键"**。

```bash
git hash-object -w a.txt   # -w 写入对象库
ls .git/objects/8d/
# 8d0e41234f24b6da002d962a26c2495ea16a425f
```

文件内容 → SHA-1 → 存到 `.git/objects/<前2位>/<后38位>`。

**反直觉的点 1**：相同内容的文件，**全仓库只存一份**。100 个 commit 里都有的 `LICENSE` 文件，磁盘上就 1 个 blob 对象。

**反直觉的点 2**：Git 完全不知道"重命名"。它只知道 "blob X 出现在 commit A 的 tree 里"。`git log --follow` 是事后**启发式**推断的，不是元数据记录。

### 1.2 4 种对象：blob / tree / commit / tag

```
                     +--------+
                     |  tag   |   (可选：annotated tag 才有 tag 对象)
                     +---+----+
                         |
                         v
                     +--------+        +--------+
                     | commit | -----> | parent | ----> ...
                     +---+----+        +--------+
                         |
                         v
                     +--------+
                     |  tree  |  (代表一个目录)
                     +---+----+
                       /  |  \
                      v   v   v
                  +-----+ +-----+ +-----+
                  | blob| | tree| | blob|
                  +-----+ +--+--+ +-----+
                            |
                            v
                          +-----+
                          | blob|
                          +-----+
```

| 对象 | 内容 | 类比 |
| --- | --- | --- |
| **blob** | 文件内容（不含文件名） | 文件系统里的 inode 数据块 |
| **tree** | 一组 (mode, name, sha) 项 | 文件系统的目录项 |
| **commit** | tree sha + parents + author + message | 一次提交的"指针" |
| **tag** | commit sha + tagger + message | 给某个 commit 打的标签 |

亲手验证：

```bash
git cat-file -t HEAD            # commit
git cat-file -p HEAD            # 看 commit 的内容
# tree 5d2f...
# parent abc123...
# author Liu <kafanqi@gmail.com> 1715000000 +0800
# committer Liu <kafanqi@gmail.com> 1715000000 +0800
#
# fix some bug

git cat-file -p 5d2f...         # 看 tree
# 100644 blob 8d0e41... README.md
# 040000 tree 7c1f...    src
```

`commit` 不存文件，只存"指向当时整个项目快照的 tree"。这就是为什么 `git checkout <commit>` 能瞬间还原任意时刻的文件树 —— **它是直接挂载一个完整快照**，不是回放 diff。

> 口诀：**Git 是快照，不是差异**。SVN 才是差异（delta-based）。Git 内部有 packfile 才会做 delta 压缩，那是**存储优化**，不是模型。

### 1.3 refs：branch / tag / HEAD 的本质

打开 `.git/refs/`：

```bash
.git/
├── HEAD                    # 文本文件：ref: refs/heads/main
├── refs/
│   ├── heads/
│   │   ├── main            # 文本文件：abc123def456...
│   │   └── feature/foo     # 文本文件：789ace...
│   ├── tags/
│   │   └── v1.0.0
│   └── remotes/origin/
│       └── main
└── packed-refs             # 大量 ref 时合并成一个文件
```

**branch 就是一个文件**，里面**就一行**：某个 commit 的 SHA。

这意味着：
- "切分支"`git checkout feature/foo` 实际是把 HEAD 的内容改成 `ref: refs/heads/feature/foo`
- "新建分支"`git branch new-br` 就是新建一个文件，写入当前 HEAD 指向的 commit SHA
- "删除分支"`git branch -D` 就是删个文件 —— commit 还在，只是没有 ref 指着了

**tag 分两种**：
- **lightweight tag**：和 branch 一样，就是 `refs/tags/<name>` 里写一个 SHA
- **annotated tag**：先创建一个 `tag` 对象（带 tagger / message / 签名），ref 指向这个 tag 对象

发版用 annotated（`git tag -a v1.0.0 -m "..."`），内部临时打点用 lightweight。

### 1.4 packfile 与 GC

裸的"一文件一对象"在大仓库下太碎。Git 会定期把对象合并成 **packfile**：

```bash
.git/objects/
├── 8d/  # loose objects（散对象）
├── pack/
│   ├── pack-abc123.pack    # 大量对象压缩在一起
│   └── pack-abc123.idx     # 索引
```

packfile 内做了两件优化：
1. **zlib 压缩**：每个对象单独压
2. **delta 压缩**：相似对象之间存 delta（如同一文件的多个版本）

何时触发打包：
- `git gc`（手动）
- `git push` / `git fetch` 后
- 散对象数超过阈值时自动触发（`gc.auto`，默认 6700）

GC 还做一件事：**清理"不可达对象"**。如果一个 commit 没有任何 branch / tag / reflog 能到达它，且超过 `gc.pruneExpire`（默认 2 周）就会被真删除。

**这就是为什么 `git reset --hard` 之后 reflog 能救命** —— reset 只是移动了 branch 指针，commit 对象还在 objects 里，reflog 里还有指向它的"足迹"，可达。

但如果你 `git reflog expire --expire=now` + `git gc --prune=now --aggressive`，**对象真的被物理删掉**，神仙难救。

### 1.5 索引（.git/index）：暂存区的实体

很多人以为 "stage" 是个抽象概念。不是 —— 它是个二进制文件：`.git/index`。

```bash
git ls-files --stage
# 100644 8d0e41234f... 0       README.md
# 100644 7c1faaa... 0          src/main.go
```

每行 = (mode, sha, stage, path)。`git add` 做的事：
1. 把工作区文件用 `git hash-object -w` 写成 blob
2. 在 index 里插入/更新一行

`git commit` 做的事：
1. 把当前 index 直接序列化成 tree 对象
2. 创建 commit 对象，parent = HEAD
3. 把 HEAD 指向新 commit

**反直觉的点**：你 `git add file.txt` 之后再修改 `file.txt`，commit 时**提交的是 add 那一刻的内容**，不是当前工作区的。因为 commit 用的是 index，index 里那个 SHA 是 add 时存的。

### 1.6 三种工作区

```
+---------------+   git add   +-------+   git commit   +------------+
|  working dir  |  --------> | index | -------------> | repository |
+---------------+             +-------+                +------------+
        ^                         |                          |
        |       git checkout      |                          |
        +-------------------------+                          |
        ^                                                    |
        |                  git checkout / reset              |
        +----------------------------------------------------+
```

| 区域 | 物理位置 | 意义 |
| --- | --- | --- |
| working directory | 文件系统 | 你正在编辑的文件 |
| index / staging | `.git/index` | 下次 commit 的预备快照 |
| repository | `.git/objects/` | 已提交的所有对象 |

`git status` 实际是同时对比：
- working ↔ index：未 staged 的修改
- index ↔ HEAD：已 staged 的修改

`git diff` 默认 working ↔ index；`git diff --cached` 看 index ↔ HEAD；`git diff HEAD` 看 working ↔ HEAD。

> 口诀：**add 改 index，commit 改 repo，checkout 改 working**。reset 三模式恰好是这三档（见 §3.3）。

---

## 2. 工作流模式对比

模型清楚了，看上层。**没有"最好的工作流"**，只有"匹配团队规模 / 发布节奏 / 多版本并行需求"的工作流。

### 2.1 Git Flow（develop + feature + release + hotfix）

2010 年 Vincent Driessen 提出，曾经的"标准答案"。

```
                            tag v1.0.0
                                |
main         o-------o----------o-----------o------>
              \       \                    /
               \       \  release/1.0.0   /
                \       o------o---------o
                 \             ^         ^
                  \   develop  |         | merge back
                   o----o------o---o-----o-------->
                        |          |
                        | feature/A
                        o----o-----o
```

5 种长期/短期分支：
- `main` —— 永远是发布版本，每次合入打 tag
- `develop` —— 集成分支，开发都在这汇合
- `feature/*` —— 从 develop 拉出，做完合回 develop
- `release/*` —— 准备发布的稳定化分支，bug 修在这里
- `hotfix/*` —— 从 main 拉，修完同时合 main 和 develop

**适用**：有明确版本号、需要并行维护多版本（如 v1.x / v2.x）、发布周期长（双周/月级）的产品型软件。

**致命缺点**：
- 长 feature 分支 → 大冲突 → 合并地狱
- develop / main 双轨 → 心智负担重
- release 分支阶段冻结新需求 → 与持续交付理念冲突

> Driessen 本人 2020 年加注：**如果你做 SaaS / Web 应用，别用 Git Flow，用 GitHub Flow**。

### 2.2 GitHub Flow（main + feature PR）

GitHub 自己用的极简模式。

```
main    o-------o-------o-------o------->
         \       \               /
          \       o------o------o   (feature/X，PR 合回)
           \             /
            o-----o-----o   (feature/Y)
```

规则就 5 条：
1. main 任何时刻都是可部署的
2. 想做事就从 main 拉一个有描述性名字的分支
3. 持续 commit，push 到远端
4. 想合并就开 PR，讨论 + Review + CI
5. 合入 main 即部署

**适用**：SaaS / 持续部署、单一生产版本、小到中等团队。

**前提**：CI 必须强、main 保护必须严、回滚要快。

### 2.3 GitLab Flow（环境分支）

GitHub Flow 不够用的场景：你有 staging / pre-prod / production 三套环境，每套环境对应不同的"已部署版本"。

```
main         o-------o-------o-------o-------o------>
                              \
                               \  cherry-pick / merge
                                v
pre-production    o-------o----o-------o------------->
                                        \
                                         v
production              o-------o--------o----------->
```

每个环境对应一个长期分支。代码从 main 流向 pre-production 流向 production，**只能往下游流，不能反向**。修 bug 在 main，cherry-pick / merge 到下游环境。

也有"按发布版本"变体：`release/2.4` `release/2.5` 长期分支并行维护，类似 Git Flow 但去掉了 develop。

**适用**：受监管行业（金融 / 医疗）、需要明确审计线、环境差异大的场景。

### 2.4 Trunk-based Development（短分支 + Feature Flag）

Google / Facebook / Netflix 等大型工程团队的事实标准。

```
trunk    o--o--o--o--o--o--o--o--o--o--o--o--o--o-->
           \     \     \     \     \     \
            o-o   o-o   o     o-o   o     o-o
            <2d   <1d   1h    <2d   1h    <1d     (短分支，几小时到 2 天)
```

核心原则：
1. **所有人直接在 trunk 上协作**（或极短命的分支，< 2 天）
2. **每天至少合一次回 trunk**
3. **未完成的功能用 Feature Flag 关掉**，代码可以合，但用户看不到
4. **CI 必须毫秒级反馈、强制门禁**
5. **发布从 trunk 切 release 分支，只 cherry-pick 紧急修复**

为什么大厂选它：
- 没有"长期分支"= 没有合并地狱
- 持续集成 = 持续验证，问题暴露最早
- Feature Flag 把"代码合并"和"功能发布"解耦 —— 上线节奏不再被开发节奏绑架

**前提**很高：
- 强 CI（提交 → 反馈 < 10min）
- 完善的 Feature Flag 平台（LaunchDarkly / Unleash / 自研）
- 代码评审在分支被合前完成（pre-merge review，不是 post-merge）
- 团队自律（不能合半成品到 trunk，哪怕有 flag 包着）

### 2.5 各模式的适用场景对照

| 模式 | 团队规模 | 发布频率 | 多版本并行 | CI 要求 | 心智负担 |
| --- | --- | --- | --- | --- | --- |
| Git Flow | 中大 | 周-月 | 强（v1/v2 并存） | 中 | 高 |
| GitHub Flow | 小-中 | 日-周（SaaS） | 弱（单一生产版本） | 中-高 | 低 |
| GitLab Flow | 中-大 | 日-周 | 中（环境分支） | 中-高 | 中 |
| Trunk-based | 任意（尤其大） | 多次/日 | 弱（短期 cherry-pick） | 极高 | 中（前期高，稳定后低） |

**选型口诀**：
- 卖软件包（IDE / 数据库 / 操作系统）→ Git Flow / GitLab Flow
- SaaS、APP 后端 → GitHub Flow（小团队）/ Trunk-based（大团队）
- 移动端 APP → GitHub Flow + 发版分支
- 强监管（银行核心）→ GitLab Flow + 强审计

> 口诀：**分支寿命越短，工作流越简单，前提是 CI 越强**。三者必须匹配。

### 2.6 Monorepo 工作流（Bazel / Nx / Lerna）

Monorepo 不是 Git 配置问题，是**工程哲学**问题。

什么场景"必然"走 Monorepo：
- 多个组件强耦合演进（前端组件库 + 上层应用）
- 跨服务的 API 合约要原子修改（gRPC proto + 客户端 + 服务端 同 PR 改完）
- 共享代码改一行希望立刻波及所有依赖者

Monorepo **必然要解决** 的问题：
1. **构建必须是"按依赖图增量"**：改一个工具库不能跑全仓库测试。Bazel / Nx 的核心价值就在这里。
2. **CI 必须做 "affected analysis"**：识别本次变更影响了哪些 target，只跑这部分
3. **代码所有权必须显式**：CODEOWNERS 文件按目录划分 reviewer
4. **大仓库性能**：Git 在 100 万文件级别会卡。Facebook 的 EdenFS、微软的 GVFS、Sapling 都是为此

Monorepo **反而引入** 的问题：
- 仓库 clone 时间变长（部分克隆 partial clone / 浅克隆 shallow clone 必备）
- 单点故障：仓库挂了所有人停工
- "权限隔离弱"：原生 Git 没法按子目录授权
- IDE 和工具链对超大仓库性能差

工具谱：
- **Bazel / Buck** —— Google / Meta 派，hermetic build，学习曲线陡，但万级目标可扩展
- **Nx / Turborepo** —— 前端 / Node 生态，开箱体验好
- **Lerna** —— 老牌，主要做 npm 包发布编排，构建本身能力弱
- **Pants v2** —— Python 友好的 Bazel 替代

> 口诀：**Monorepo 不是把代码堆一起，是配套构建系统 + CI + 代码所有权的整套工程实践**。否则就是个臃肿的多仓。

---

## 3. 高级操作切片（每段命令完整可复制）

> 下面所有命令在 Git 2.40+ 上验证过。

### 3.1 rebase 的几种用法（变基 / interactive / onto）

#### 3.1.1 普通 rebase：把分支"挪到"新基点

```bash
# 你在 feature 分支，main 又往前走了几个 commit
git checkout feature
git rebase main

# 等价于：
# 1. 把 feature 上 main 之后的 commit 暂存（按时间顺序）
# 2. 把 feature 重置到 main 的 HEAD
# 3. 依次 cherry-pick 暂存的每个 commit 到 feature
```

效果：feature 的"基点"从原来的某个 commit 变成 main 当前 HEAD。**commit SHA 全变了**（因为 parent 变了）。

#### 3.1.2 interactive rebase：编辑历史

```bash
git rebase -i HEAD~5
# 弹出编辑器，可以做：
# pick   a1b2c3 commit 1     ← 保留
# squash d4e5f6 commit 2     ← 合并到上一个
# fixup  g7h8i9 commit 3     ← 合并到上一个，丢弃 message
# reword j1k2l3 commit 4     ← 改 message
# edit   m4n5o6 commit 5     ← 暂停，让你修改文件
# drop   p7q8r9 commit 6     ← 删除
```

**生产用法**：在 PR 合并前清理自己的 commit 历史。把"fix typo / fix again / 终于修好了"挤成一个干净 commit。

#### 3.1.3 `--onto`：把分支移植到任意基点

最强大也最绕的 rebase 用法。

场景：你从 `feature/A` 拉了个 `feature/B`，做了几个 commit。结果 `feature/A` 被废弃了，你想把 `feature/B` 直接接到 `main` 上。

```bash
# 当前：main ---o
#                \
#         feature/A o---o
#                       \
#                feature/B o---o---o   ← 你在这

git rebase --onto main feature/A feature/B
# 含义：把 feature/A..feature/B 之间的 commit，"嫁接"到 main 上
```

效果：

```
main ---o---o'---o'---o'   (B 的 3 个 commit 被重放到 main 之上)
         \
   feature/A o---o   (孤立了)
```

**逐参数解释**：
- `--onto <newbase>`：新基点是 main
- `<upstream>` = feature/A：剔除这之前的所有 commit
- `<branch>` = feature/B：要操作的分支

#### 3.1.4 rebase 的本质代价

每次 rebase 都会**重写 commit**（产生新 SHA）。如果 rebase 已经 push 到共享分支的提交：

```bash
# ❌ 灾难现场
git checkout main
git rebase -i HEAD~10        # 整理了 main 的历史
git push --force origin main # 同事的本地 main 全断
```

**铁律**：**rebase 只对你自己未推送的本地 commit 用**。一旦推送到共享分支，只能 merge，不能 rebase。

> 口诀：**Rebase before push, merge after push**。

### 3.2 cherry-pick 跨分支挑选

```bash
# 单个 commit
git cherry-pick a1b2c3

# 一段连续 commit
git cherry-pick a1b2c3..d4e5f6   # 不含 a1b2c3，含 d4e5f6
git cherry-pick a1b2c3^..d4e5f6  # 含 a1b2c3

# 多个不连续
git cherry-pick a1b2c3 e5f6g7 h8i9j0

# 只把 commit 内容应用到工作区，不创建 commit
git cherry-pick -n a1b2c3
```

**关键选项**：
- `-x`：在 message 后追加 `(cherry picked from commit ...)` 注释，方便追溯
- `--mainline 1`：cherry-pick 一个 merge commit，告诉 Git 取哪边的"主线"

#### 与 rebase 的关系

`git rebase --onto X Y..Z` 的实现 **就是连续 cherry-pick**。所以二者本质同源，区别在：

| 维度 | cherry-pick | rebase |
| --- | --- | --- |
| 目标位置 | 当前分支末尾 | 新指定的基点 |
| 范围 | 灵活（任意 commit 列表） | 一段连续区间 |
| 是否移动 ref | 否（只追加） | 是（重写整个分支） |

cherry-pick 适合"跨分支挑修复"（hotfix → develop / release），rebase 适合"整体迁移"。

### 3.3 reset 三种模式（soft / mixed / hard）

记住前面的图：working / index / repo 三个区。reset 就是把 HEAD 移到目标 commit，然后**有选择地同步另外两个区**。

```bash
git reset --soft  <commit>    # 只移 HEAD，index 和 working 不动
git reset --mixed <commit>    # 移 HEAD，重置 index，working 不动（默认）
git reset --hard  <commit>    # 移 HEAD，重置 index 和 working（危险！）
```

| 模式 | HEAD | index | working | 典型用途 |
| --- | --- | --- | --- | --- |
| `--soft` | 改 | 不动 | 不动 | 想合并最近几个 commit 重 commit |
| `--mixed` | 改 | 改 | 不动 | 想重新 add 一遍 |
| `--hard` | 改 | 改 | 改 | 彻底回到某个版本，**不可逆** |

举例：你想把最近 3 个 commit 合成一个新 commit。

```bash
git reset --soft HEAD~3      # HEAD 退 3 步，但所有改动还在 index 里
git commit -m "新的合并 commit"
```

`--hard` 的恐怖：working 区里**没 commit 的修改全没了**。Git 不进 trash，**直接删**。

### 3.4 reflog：救命神器

reflog 记录了 **HEAD 和每个 branch** 的"指针移动历史"。任何"看似破坏性"的操作（reset / rebase / branch -D / checkout）都会在 reflog 留下足迹。

```bash
git reflog
# a1b2c3 HEAD@{0}: reset: moving to HEAD~3
# d4e5f6 HEAD@{1}: commit: my last commit before disaster
# g7h8i9 HEAD@{2}: commit: another commit
# ...
```

救命套路：

```bash
# 看 reflog 找到"灾难前"的那个 commit
git reflog
# 找到 d4e5f6

# 恢复
git reset --hard d4e5f6
# 或新建一个分支保住它
git branch rescue d4e5f6
```

#### reflog 的边界

reflog 救命**有有效期**：
- 默认 90 天后过期（`gc.reflogExpire`）
- 不可达的 reflog 条目 30 天过期（`gc.reflogExpireUnreachable`）
- 过期后 + 下次 GC 时被清理

**真正不可救** 的情况：
1. 操作发生在另一台机器（reflog 是本地的，不上传不下载）
2. `git reflog expire --expire=now --all` + `git gc --prune=now` 同时执行
3. 仓库被重新 clone（reflog 全丢，因为 reflog 本地存在 `.git/logs/`）

### 3.5 worktree：多分支并行工作

传统玩法：一个仓库，切分支 = 切工作区。如果你正在 `feature/A` 写代码写一半，需要紧急修 `main` 上的 bug —— `git stash` 然后切 main，修完再切回来 stash pop。

`worktree` 让你**同一仓库挂载多个工作目录**。

```bash
# 假设你在 ~/repo
git worktree add ../repo-hotfix main
# 在 ../repo-hotfix 出现一个完整的工作目录，HEAD 是 main

cd ../repo-hotfix
# 在这里改代码、commit、push，完全独立

# 完了清理
git worktree remove ../repo-hotfix
```

**底层**：`.git/worktrees/<name>/` 存这个 worktree 的 HEAD / index / 等元数据。所有 worktree 共享同一个 objects 库 —— 不重复占磁盘。

**生产场景**：
- 紧急切 main 修 bug，不打断当前 feature
- 同时跑两个分支的 CI / 编译
- review 别人 PR 时新建一个 worktree，不影响自己的工作目录

**铁律**：**同一个 branch 不能在两个 worktree 同时 check out**（避免 ref 冲突）。

### 3.6 bisect：二分查找 Bug

某次发版后某功能挂了，你知道 v1.5 还好，v1.6 已经坏。但 v1.5..v1.6 之间有 200 个 commit。怎么找出引入 bug 的那个？

```bash
git bisect start
git bisect bad           # 当前 HEAD 是坏的
git bisect good v1.5     # v1.5 是好的

# Git 自动 checkout 中间一个 commit，让你测
# 测完告诉它结果：
git bisect good          # 当前这个 commit 是好的
# 或
git bisect bad           # 当前这个 commit 是坏的

# Git 继续二分，直到找到第一个坏的 commit
# log2(200) ≈ 8 次就能定位

git bisect reset         # 用完恢复 HEAD
```

#### 自动化 bisect

写一个测试脚本，让 git 自己跑：

```bash
# test.sh：返回 0 = good，非 0 = bad，125 = skip
#!/bin/bash
make build && ./run-test.sh
exit $?

git bisect start HEAD v1.5
git bisect run ./test.sh
# Git 自动跑完，输出 "first bad commit: a1b2c3..."
```

**坑**：
- `bisect skip` 用于"这个 commit 编不过、跳过它"
- 二分前必须确保 good 真的好、bad 真的坏（人脑都会错）
- 测试必须确定性。flaky test 会让 bisect 完全失效

### 3.7 filter-branch / git-filter-repo：历史改写

需要：
- 删除某个被误 commit 的敏感文件 / 大文件
- 把整个仓库的 author email 批量改了
- 提取某个子目录作为新仓库

`filter-branch` 是 Git 自带的，**官方已弃用**（性能差、坑多）。新的标准是 `git-filter-repo`（独立工具，pip / brew 安装）。

#### 删大文件

```bash
# ❌ 老办法（慢、不安全、官方不推荐）
git filter-branch --tree-filter 'rm -f huge.psd' HEAD

# ✅ 新办法（filter-repo，快几十倍）
pip install git-filter-repo
git filter-repo --path huge.psd --invert-paths
```

`--invert-paths` 含义：保留所有文件，**除了** 指定的。

#### 提取子目录为新仓库

```bash
# 把 monorepo/services/foo 单独抽出来
git filter-repo --subdirectory-filter services/foo
```

#### 批量改作者

```bash
git filter-repo --mailmap mailmap.txt
# mailmap.txt 内容：
# New Name <new@example.com> <old@example.com>
```

#### 重要警告

历史改写会**让所有 commit SHA 改变**。任何引用旧 SHA 的：
- 已发出去的 PR 链接
- 其他仓库 cherry-pick 过来的 commit
- CI / 监控系统记录的 SHA

全部失效。**只对私有 / 未协作仓库做历史改写**，或所有协作者必须重新 clone。

### 3.8 submodule vs subtree

两种"在仓库里嵌入另一个仓库"的方案。

#### submodule

```bash
git submodule add https://github.com/foo/bar libs/bar
# 父仓库里多了一个 .gitmodules 和 libs/bar/ 目录
# libs/bar/ 实际是个独立 git 仓库
```

父仓库**只记录子仓库的 SHA 指针**，不存子仓库的内容。clone 时要：

```bash
git clone --recurse-submodules <repo>
# 或
git clone <repo> && git submodule update --init --recursive
```

**痛点**：
- 子模块 SHA 不更新，父仓库可能拉到老版本
- 多人协作时，"我提交了，你 pull 了，但子模块没更"是日常
- detached HEAD 状态默认（子模块 checkout 出来不是分支）

#### subtree

```bash
git subtree add --prefix=libs/bar https://github.com/foo/bar main --squash
# 把 bar 仓库的内容"嵌"到 libs/bar/，作为父仓库的一部分
```

父仓库**包含子仓库的全部历史 / 内容**（或 squash 后的快照）。

```bash
# 拉子仓库的更新
git subtree pull --prefix=libs/bar https://github.com/foo/bar main --squash

# 把 libs/bar/ 的修改推回子仓库
git subtree push --prefix=libs/bar https://github.com/foo/bar main
```

**痛点**：
- 父仓库会变大（含子仓库历史）
- subtree push / pull 命令冗长
- 推回上游需要权限

#### 选型

| 场景 | 选 |
| --- | --- |
| 子项目独立演进、版本明确 | submodule |
| 子项目主要在父仓库改、上游偶尔同步 | subtree |
| 大型项目想要 monorepo 体验 | 其实都不该用，直接 monorepo |

**新趋势**：用 Bazel / Buck 的外部依赖系统，或 Go modules / npm workspaces 这类语言级方案，绕开 git submodule/subtree 的坑。

> 口诀：**submodule 是指针、subtree 是拷贝**。前者轻但易乱，后者重但自洽。

---

## 4. 生产案例（9 则）

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 误 push 强制覆盖了同事的工作

#### 现象

你做完自己的 feature，rebase 到最新 main 后 `git push --force origin main`。
半小时后同事在群里：**"我上午的 3 个 commit 怎么没了？"**

去 GitHub 看 main 分支历史，确实只有你的 commit，同事的 3 个 commit 消失。

#### 根因

```bash
git push --force
# 等于：用本地分支当前状态，覆盖远端分支当前状态
# 远端原来指向的 commit 如果没有任何 ref 还指着它，就成"不可达"
```

你 rebase 时基于的 main 是 1 小时前的 main，没拉同事的提交。force push 把同事的 3 个 commit "孤儿化"了。

#### 修复

**第一步：远端救人**。GitHub / GitLab 有 reflog，但用户访问不到。需要：

```bash
# 同事在自己机器上还有那 3 个 commit（除非他也 reset 了）
# 让他从自己本地恢复
git push --force-with-lease origin missing-commits:main
```

或**通过远端 reflog 救**（需要服务器管理员权限）：

```bash
# 在服务器上的 bare 仓库
ssh git@server
cd /path/to/repo.git
git reflog refs/heads/main
# 找到被覆盖前的 SHA，比如 a1b2c3
git update-ref refs/heads/main a1b2c3
```

GitHub 不开放服务器 reflog 直接访问，但可以用 API 恢复：

```bash
# GitHub API：events 接口里能查到被推之前的 commit SHA
curl https://api.github.com/repos/<owner>/<repo>/events | jq '.[] | select(.type=="PushEvent")'
# 拿到 SHA 后用 update-ref 恢复
```

**第二步：永久预防**。把 `--force` 从大脑里删掉，换成 `--force-with-lease`：

```bash
git push --force-with-lease origin main
# 含义：只有当远端 main 还在我"上次拉到"的位置时，才覆盖
# 如果别人已经推了新内容，这个命令会拒绝，不会覆盖
```

**第三步：服务器侧加保护**。GitHub 设置 → Branches → Branch protection：

- ✅ Restrict pushes that create files
- ✅ Require pull request reviews before merging
- ✅ Disallow force pushes
- ✅ Disallow deletions

main / master 永远不允许直接 push，必须 PR。

#### 验证

写个 git alias 把 `push --force` 永远 alias 成 `push --force-with-lease`：

```bash
git config --global alias.pushf "push --force-with-lease"
# 以后只敲 git pushf
```

或更狠：

```bash
git config --global alias.push '!echo "use git pushf instead" && false'
```

> 口诀：**force-with-lease 永远代替 force**。Branch protection 是最后一道防线。

---

### 案例 ② rebase 灾难：解决冲突时丢代码

#### 现象

你的 feature 分支落后 main 30 个 commit，你 `git rebase main`。中间 8 个 commit 都有冲突。你一路 `git add . && git rebase --continue`。

合到 main 后跑 CI，**一半测试挂了**。一查 diff，发现你自己实现的某个关键函数没了。

#### 根因

rebase 是**逐个 commit 重放**。每次解决冲突只解了"这一个 commit 引入的冲突"。如果同一段代码被你的 commit X 和后续 commit Y 都改了，X 解冲突时若选错（比如 `--theirs` 取了 main 的版本），**你的修改就被冲掉了**。Y 解冲突时基于的"已解决版本"已经是错的。

更糟：rebase 中间的"中间状态" commit 可能根本编不过，但你没跑测试，错误一直累积到最后。

#### 修复

**方案 A：用 `git rebase --merge` 风格的工具辅助**

```bash
# 用 ours/theirs 之前先看清是哪边
git checkout --conflict=diff3 path/to/file
# 文件里会出现：
# <<<<<<< HEAD
# main 的版本
# ||||||| common ancestor
# 共同祖先
# =======
# 你的版本
# >>>>>>> commit-being-rebased
```

`diff3` 风格能看到祖先，最容易判断"应该保留谁"。

**方案 B：对长冲突 rebase，改用 merge**

```bash
git checkout feature
git merge main
# 一次性解决所有冲突，不是逐 commit 解
```

代价：feature 分支多一个 merge commit，历史不那么"直"。但**对大量冲突场景，merge 是更安全的选择**。

**方案 C：使用 rerere（reuse recorded resolution）**

```bash
git config --global rerere.enabled true
```

第一次解决某个冲突后，Git 记录"冲突 hash → 解决方案"。下次遇到同样的冲突自动应用。对于反复 rebase 同一分支的场景救命。

**方案 D：每解决一个 commit 的冲突，跑一遍编译**

```bash
git rebase --exec 'make build && make test' main
# 任何一步编译/测试挂了，rebase 立刻停下，让你检查
```

#### 验证

rebase 完成后，**强制对比**：

```bash
# 对比 rebase 前后的"实际代码差异"，应该和 rebase 前 feature vs main 一致
git diff feature@{1} feature
# feature@{1} 是 reflog 里 rebase 前的位置

# 或保险起见，rebase 前打 backup tag
git tag backup-before-rebase
git rebase main
# 确认无误后再删 tag
git tag -d backup-before-rebase
```

> 口诀：**长冲突 rebase 是事故源头**。要么 `--exec` 每步验证，要么改用 merge。**rerere 必开**。

---

### 案例 ③ merge 冲突解决错误导致回归

#### 现象

PR 合并后线上某个支付路径直接 500。代码 review 时大家都看了，但合并后的代码里**少了一个 if 判断**。这个 if 是同事 A 上周加的修复。

#### 根因

PR 作者的分支与 main 同时改了同一段代码：
- A 在 main 上加了 `if (amount < 0) throw` 校验
- 你在自己分支重构了这段 method

merge 时冲突展示：

```
<<<<<<< HEAD (main)
if (amount < 0) throw new IllegalArgumentException();
processPayment(amount);
=======
processPaymentV2(normalize(amount));
>>>>>>> your-branch
```

你"重构后没必要那个 if 了"心想，直接选了自己的版本：

```
processPaymentV2(normalize(amount));
```

但 `normalize` 不会拒绝负数。**A 的修复消失**。Code reviewer 不知道 main 上那个 if 是上周加的，看 diff 看不出"原来这里有"。

#### 修复

**方案 A：semantic conflict 必须人脑参与，工具救不了**。流程上：
- merge 解冲突时**必须看 main 的 git log -p**，理解被冲掉的那段代码"为什么存在"
- 解完冲突跑全量测试，不只是相关测试
- 大改动前先沟通

**方案 B：保留 A 的修复，重新写**

```java
public void processPayment(BigDecimal amount) {
    if (amount.signum() < 0) {
        throw new IllegalArgumentException("amount must be non-negative");
    }
    processPaymentV2(normalize(amount));
}
```

**方案 C：增加 regression test**

```java
@Test
void shouldRejectNegativeAmount() {
    assertThrows(IllegalArgumentException.class,
        () -> service.processPayment(new BigDecimal("-1")));
}
```

测试一旦写了，**未来任何 merge 重新引入这个 bug 都会被 CI 拦下**。

#### 验证

```bash
# 解冲突后，必须做的对比：
git log --oneline main..HEAD     # 我合进来的 commit
git log --oneline HEAD..main     # main 上我没有的 commit

# 重点看 HEAD..main 这边，确认每个修复都还在
git diff main HEAD -- path/that/changed.java
```

CI 必须有：
- 完整的回归测试套件
- 关键路径覆盖率 > 80%
- 强制门禁，红灯不让合

> 口诀：**冲突解决错的代价远大于冲突本身**。每次冲突解决都是一次"小型 code review"，必须看 git log 理解被冲的代码"为何存在"。

---

### 案例 ④ 大文件被 commit，仓库膨胀

#### 现象

仓库 clone 时间从 30 秒涨到 5 分钟。du -sh .git 看到 4GB。但工作目录只有 80MB。

`git rev-list --objects --all | sort -k 2 | uniq -f 1 -d`，一查发现两年前某次提交带进来一个 1.2GB 的训练数据 zip，后来被删除了，但还在历史里。

#### 根因

Git 的"内容寻址 + 对象库"特性意味着：**任何一次出现过的内容，永远在 objects 库里**（除非显式删 + GC）。`git rm` 只是从当前 commit 的 tree 里去掉了引用，blob 对象本身还在 packfile 里。

每次 clone = 下载完整 objects 库 = 包括两年前那 1.2GB。

#### 修复

**第一步：找出大文件**

```bash
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '/^blob/ {print substr($0, index($0, $3))}' \
  | sort -rn | head -20
# 输出按大小排序的前 20 个 blob
```

或用 [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/)（专门做这个）：

```bash
java -jar bfg.jar --strip-blobs-bigger-than 50M repo.git
```

**第二步：用 git-filter-repo 清理**

```bash
# 先备份完整仓库！
cp -r repo repo.backup

# 清掉指定文件
cd repo
git filter-repo --path data/training.zip --invert-paths

# 清掉所有大于 100MB 的 blob
git filter-repo --strip-blobs-bigger-than 100M
```

**第三步：force push（团队所有人必须重新 clone）**

```bash
git push --force --all origin
git push --force --tags origin
```

**第四步：远端清理**

```bash
# 在服务器侧（GitHub / GitLab Admin）
# 联系运维做 server-side gc
# 否则 GitHub 仓库实际大小不变，只是用户 clone 时不再下载这些对象
```

GitHub 自助：Settings → Storage → "Run garbage collection"（仅企业版可见，一般要联系 support）。

**第五步：未来防护 —— Git LFS**

大文件改用 LFS（Large File Storage）：

```bash
git lfs install
git lfs track "*.psd" "*.zip" "*.pdf"
# 自动生成 .gitattributes：
# *.psd filter=lfs diff=lfs merge=lfs -text
```

LFS 的工作原理：
- 文件**不进 git 对象库**，进 LFS 服务器
- Git 里只存一个指针文件：`oid sha256:abc... size 1234567`
- clone 时按需下载（或浅下载）

#### 配 pre-commit hook 拦大文件

```bash
# .git/hooks/pre-commit
#!/bin/bash
max_size=10485760  # 10 MB

added=$(git diff --cached --name-only --diff-filter=A)
for file in $added; do
    size=$(stat -f %z "$file" 2>/dev/null || stat -c %s "$file")
    if [ "$size" -gt "$max_size" ]; then
        echo "❌ $file ($size bytes) exceeds $max_size"
        exit 1
    fi
done
```

或用 [pre-commit](https://pre-commit.com/) 框架的官方 hook：

```yaml
# .pre-commit-config.yaml
- repo: https://github.com/pre-commit/pre-commit-hooks
  hooks:
    - id: check-added-large-files
      args: ['--maxkb=10240']
```

#### 验证

```bash
# 清理后看仓库大小
du -sh .git

# 看是否还有大对象
git rev-list --objects --all | git cat-file --batch-check='%(objectsize) %(rest)' | sort -rn | head

# 重新 clone 看时间
time git clone <new-url> /tmp/test-clone
```

> 口诀：**大文件进 LFS、配 pre-commit 拦截、history 用 filter-repo 清理**。三件套缺一不可。GitHub 仓库实际大小不会自动随历史改写减少，必须联系运维。

---

### 案例 ⑤ 分支历史发散无法合并

#### 现象

Fork 出去的 fork 仓库做了大量修改后想合回原仓库，PR 一开 GitHub 显示 "this branch has conflicts that must be resolved"，看了一下涉及 200 个文件的冲突。

更糟的是，原仓库和 fork 仓库的初始 commit 都被改写过（早期都做过 filter-branch），**历史完全没有公共祖先**。

#### 根因

Git merge 算法 (recursive / ort) 依赖找到**最近公共祖先**（LCA）。如果两个分支历史没有任何公共祖先，merge 算出来的"差异"就是两边所有文件的全量 diff。

`git log --oneline --graph fork-main upstream-main` 完全是两条平行线，没有交点。

#### 修复

**方案 A：当成两个独立项目对待**，手动挑要合的内容

```bash
# 在原仓库新开一个分支
git checkout -b adopt-fork upstream-main

# 把 fork 的整个工作树拷贝过来（不带历史）
rsync -av --exclude='.git' /path/to/fork/ ./

# 手动 review 差异，分批 commit
git add -p   # 交互式选择要 stage 的内容
```

**方案 B：强制 allow-unrelated-histories**

```bash
git remote add fork https://github.com/foo/bar
git fetch fork
git merge fork/main --allow-unrelated-histories
# 接下来要解决 200 个冲突...
```

不推荐。冲突解决工作量等同方案 A，但历史更难看。

**方案 C：subtree merge**

```bash
git remote add fork https://github.com/foo/bar
git fetch fork
git read-tree --prefix=fork-snapshot/ -u fork/main
git commit -m "import fork snapshot as fork-snapshot/"
# 然后逐文件 diff，决定哪些应用到主代码
```

#### 长期预防

- **维护 fork 时定期 sync upstream**：

```bash
git remote add upstream <original-url>
git fetch upstream
git rebase upstream/main   # 或 merge upstream/main
git push origin main       # 推到自己 fork
```

- 周期：至少周级，最好日级。
- 不要在 fork 上做 history rewrite（除非彻底放弃合回）。

#### 验证

```bash
# 看两个分支是否有公共祖先
git merge-base fork-main upstream-main
# 输出 SHA = 有公共祖先；输出空 = 没有

# 看分支差异规模
git rev-list --count fork-main ^upstream-main
git rev-list --count upstream-main ^fork-main
```

> 口诀：**长期 fork 必须定期同步 upstream**。一旦历史发散到没公共祖先，就只能当两个独立项目对待。

---

### 案例 ⑥ 误删本地分支

#### 现象

清理本地分支：

```bash
git branch | grep feature/ | xargs git branch -D
```

跑完发现有个 `feature/important-but-not-yet-pushed` 也被删了，里面是你昨天写到一半的 200 行代码，**还没 push**。

#### 根因

`git branch -D` 只是删了 ref（`.git/refs/heads/feature/important-but-not-yet-pushed` 这个文件）。commit 对象本身还在 objects 里，但没有任何 ref / HEAD 指着它，成"不可达"。

90 天 GC 之前都还在，能救。

#### 修复

```bash
# 找回 commit
git reflog
# 在 reflog 里应该能看到你那个分支最后的 commit
# a1b2c3 HEAD@{15}: commit: WIP: 分支死前的最后一个 commit

# 重建分支
git branch feature/important-but-not-yet-pushed a1b2c3
# 或直接切过去
git checkout -b feature/important-but-not-yet-pushed a1b2c3
```

如果 reflog 找不到（比如分支是从 detached HEAD 创建的，再切走过）：

```bash
# 列出所有"不可达"的 commit
git fsck --unreachable --no-reflogs
# unreachable commit a1b2c3 ...
# unreachable blob ...

# 但这种是真孤儿，得逐个 git show 看是不是你想要的
```

#### 长期预防

**预防 1：永远先 push WIP 分支**。哪怕代码没写完：

```bash
git commit -m "WIP: in progress, do not review"
git push -u origin feature/important
```

push 上去 = 远端有备份 = 删了能从远端拉回来。

**预防 2：批量删分支用 `-d` 不用 `-D`**

```bash
git branch -d feature/foo
# -d (小写)：拒绝删除"未合并"的分支，给你提示
# -D (大写)：强制删，不检查
```

`xargs git branch -d` 会让"未推送 / 未合并"的分支保留下来。

**预防 3：用脚本而不是 oneliner**

```bash
#!/bin/bash
# clean-merged-branches.sh
git fetch --prune
for branch in $(git branch --merged main | grep -v "main\|master\|develop"); do
    echo "Delete $branch? [y/N]"
    read confirm
    [ "$confirm" = "y" ] && git branch -d "$branch"
done
```

#### 验证

```bash
# 删除前先看有哪些分支没 push
git for-each-ref --format='%(refname:short) %(upstream:short)' refs/heads/ \
  | awk '$2 == "" {print $1}'
# 输出所有"没有跟踪远端"的分支
```

> 口诀：**未 push 的分支永远先 push**。`-d` 拒绝未合并分支是 git 内置的安全网，**永远先用 `-d` 试一下，再用 `-D`**。

---

### 案例 ⑦ git pull 自动 merge 污染历史

#### 现象

团队的 main 分支历史看起来像意大利面：

```
*   Merge branch 'main' of github.com/foo/bar into main
|\
* | fix: typo
| * fix: typo (一模一样)
|/
*   Merge branch 'main' of github.com/foo/bar into main
|\
* | feat: A
| * feat: B
|/
```

每次 pull 都创建一个 merge commit。10 个开发者并行工作，main 历史完全不可读。bisect 时看到一堆 "Merge branch 'main' of ..." 找不到真正的代码变更。

#### 根因

`git pull` 默认行为 = `git fetch + git merge FETCH_HEAD`。如果你和远端各自有提交，merge 会创建一个 merge commit。

```
你本地：    main → A → B (你的提交)
远端：      main → A → C (同事的提交)

git pull 后：
            main → A → B ─┐
                     \    \
                      C ── M (merge commit)
```

每次有人 pull = 一个 merge commit。10 人协作 = 历史完全爆炸。

#### 修复

**方案 A：把默认改成 rebase**

```bash
git config --global pull.rebase true
# 等价于 git pull --rebase
# fetch 后用 rebase 把本地 commit 重放到远端 HEAD 上，无 merge commit
```

```
你本地：    main → A → B
远端：      main → A → C

git pull --rebase 后：
            main → A → C → B'   (B 被 rebase，SHA 变成 B')
```

历史保持线性。

**方案 B：禁止默认行为**

```bash
git config --global pull.ff only
# 只允许 fast-forward 的 pull，否则报错
# 强制你显式选 rebase 或 merge
```

跑 `git pull` 报错时，你必须想清楚要用哪个：

```bash
git pull --rebase   # 我有本地 commit，想把它接到远端 HEAD 上
git pull --no-rebase # 我想保留 merge commit（少见）
```

**方案 C：commit 前先 pull --rebase**

```bash
# 习惯：写代码前先同步
git pull --rebase
# 写代码、commit
# push
git push
```

#### 服务器侧强制

GitHub 的 PR 合并方式：

| 方式 | 效果 | 推荐场景 |
| --- | --- | --- |
| Create a merge commit | 保留所有 PR commit + merge commit | 想保留完整开发过程 |
| Squash and merge | 把 PR 所有 commit 压成 1 个 commit | **trunk-based 标配**，main 历史最干净 |
| Rebase and merge | 把 PR commit 一个个 rebase 到 main | 想保留单 commit 但要线性历史 |

团队约定后，在 Repository Settings 里**只勾选一种**，强制执行。

#### 验证

```bash
# 看历史是否线性
git log --graph --oneline main | head -30
# 健康的 main 应该是一条直线

# 找出无意义的 merge commit
git log --merges --grep="^Merge branch" --oneline | head
```

> 口诀：**`pull.rebase=true` 和 `pull.ff=only` 是 git 配置的"基础卫生"**。GitHub PR 强制 squash 或 rebase merge，禁掉默认 merge commit。

---

### 案例 ⑧ submodule 引发的依赖灾难

#### 现象

CI 突然全挂，错误：

```
Cloning into 'libs/common'...
fatal: unable to access 'https://github.com/old-org/common.git/': repository not found
```

查了一下，common 库 owner 把仓库迁移到新组织，旧 URL 失效。但你父仓库 `.gitmodules` 里 hard-code 了旧 URL。

更糟：父仓库里 50 个微服务都通过 submodule 引用这个 common，**全部 CI 红**。

#### 根因

submodule 设计假设：**子仓库 URL 永远稳定**。一旦上游改 URL / 改 owner / 私有化，所有依赖方全挂。

`.gitmodules` 里：

```ini
[submodule "libs/common"]
    path = libs/common
    url = https://github.com/old-org/common.git
```

这个 URL 在每个 fork、每个开发者本地、每个 CI 节点都是固定的。

#### 修复

**应急：批量改 URL**

```bash
# 在每个仓库
git config --file .gitmodules submodule.libs/common.url https://github.com/new-org/common.git
git submodule sync --recursive
git submodule update --init --recursive
git add .gitmodules
git commit -m "fix: update common submodule URL"
git push
```

50 个仓库要写脚本：

```bash
#!/bin/bash
for repo in service-*; do
    cd $repo
    git checkout main && git pull
    git config --file .gitmodules submodule.libs/common.url https://github.com/new-org/common.git
    git submodule sync --recursive
    git add .gitmodules
    git commit -m "fix: common URL"
    git push
    cd ..
done
```

**长期：用相对 URL**

```ini
[submodule "libs/common"]
    path = libs/common
    url = ../../common.git    ; 相对当前父仓库的位置
```

这样 fork 时子模块自动跟着 fork 走，不再 hard-code。但只在"父子仓在同一组织"时有效。

**更长期：替代 submodule**

| 替代方案 | 适用 |
| --- | --- |
| 包管理器（npm / Maven / Go modules） | 子项目能发包的语言 |
| Monorepo | 父子强耦合、希望原子修改 |
| Bazel external repository | 需要可重现 hermetic build |
| `git subtree`（带历史拷贝） | 子项目变化少 |

submodule 在现代工程实践里**几乎只剩"嵌入第三方代码"**这一个场景，且必须 fork 一份保命。

#### 验证

```bash
# 检查所有 submodule 状态
git submodule status
# 输出形如：
# +abc123 libs/common (热点：+号表示 submodule 不在父仓库记录的 commit 上)
# -def456 libs/utils  (-号表示 submodule 还没初始化)
#  789abc libs/models (空格表示 OK)

# 强制 reset 所有 submodule 到父仓库记录的版本
git submodule update --init --recursive --force
```

CI 必须有：

```yaml
# .github/workflows/ci.yml
- uses: actions/checkout@v4
  with:
    submodules: recursive
    token: ${{ secrets.SUBMODULE_PAT }}   # 私有 submodule 必须配 PAT
```

> 口诀：**submodule 是"指针游戏"**，URL / 权限 / 同步时机任何一个问题都会让团队停工。能不用尽量不用，必须用就配相对 URL + 监控 + 文档。

---

### 案例 ⑨ Trunk-based 下的"主干被破坏"应急

#### 现象

团队走 trunk-based。下午 3 点某个开发的 commit 合到 main，触发 CI 全红。但他没在群里说就下班了。

后续半小时内，10 个开发者 pull 了最新 main，本地立刻报错。CI 队列堵满。所有人都来问"main 怎么了"。

#### 根因

trunk-based 的核心假设：**main 任何时刻都是绿的**。一旦 main 被破坏，所有以 main 为基的 PR / 本地分支全部受影响。

破坏可能来自：
- 测试在本地通过但 CI 环境失败（环境差异）
- 某个被合并的 PR 与同一时刻另一个 PR 的语义冲突（merge 时 git 没识别为冲突，但语义上冲突）
- 被合并的 PR 引入了一个 flaky test，CI 偶尔失败

#### 修复

**短期：revert 优先于 forward fix**

trunk-based 团队必须有一个文化共识：**main 红了，先 revert 再调查**。

```bash
git revert <bad-commit-sha>
# revert 创建一个反向 commit，恢复到 bad commit 之前的状态
git push origin main
```

为什么 revert 优先于"我马上修一下再合"？
- revert 是 1 分钟操作
- "马上修一下" 可能 30 分钟，期间所有人被堵
- revert 后原作者依然可以从被 revert 的 commit 上重新做修复 PR

**更狠：自动 revert**

CI 自动检测 main 红，就 revert 最后合入的 PR：

```yaml
# .github/workflows/auto-revert.yml
on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]

jobs:
  auto-revert:
    if: ${{ github.event.workflow_run.conclusion == 'failure' && github.event.workflow_run.head_branch == 'main' }}
    steps:
      - run: |
          git revert --no-edit ${{ github.event.workflow_run.head_sha }}
          git push origin main
```

**中期：merge queue**

GitHub Merge Queue / GitLab Merge Train：PR 不直接合并到 main，而是排队，CI 在"假设合并后"的状态上跑，绿了才真合。

```
PR1 想合 main:
  CI: rebase PR1 onto main, run tests → 绿 → 合
PR2 想合 main:
  CI: rebase PR2 onto (main + PR1), run tests → 绿 → 合
PR3 想合 main:
  CI: rebase PR3 onto (main + PR1 + PR2), run tests → 红 → 拒绝合
```

代价：CI 资源消耗大、合并延迟变高（要排队）。但 main 极少被破坏。

**长期：feature flag 保护**

任何"高风险修改"先 ship 到 main 时用 flag 关掉：

```java
if (FeatureFlags.isEnabled("new-payment-flow")) {
    return newPaymentFlow.process(req);
} else {
    return oldPaymentFlow.process(req);
}
```

线上效果可控（flag 默认 off），即使 main 上有问题也不影响生产。问题修好后慢慢打开 flag 给 1% / 10% / 100% 用户。

#### 团队约定

trunk-based 团队必须明文：
1. Push 完不准下班，必须等 CI 绿
2. main 红了所有人停手，先 revert
3. flaky test 24 小时内必须 quarantine（标 `@Flaky` 暂时跳过）或 fix
4. PR 合并前必须 rebase 到最新 main + CI 绿

#### 验证

```bash
# 看 main 最近 7 天的红/绿率
gh run list --workflow=CI --branch=main --limit=100 --json conclusion \
  | jq '[.[] | .conclusion] | group_by(.) | map({status: .[0], count: length})'
```

健康指标：
- main CI 通过率 > 95%
- 平均 main 红到绿恢复时间 < 10 分钟
- 单 PR 合并到 main 的端到端时间 < 1 小时

> 口诀：**trunk-based 的命门是 main 绿率**。**revert 优先于 forward fix**、merge queue + feature flag 是配套设施。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 配置层
- [ ] `.gitignore` 完备：覆盖 IDE / OS / 构建产物 / 密钥
- [ ] `.gitattributes` 配置：`*.png binary` / `text eol=lf` / LFS 规则
- [ ] `pull.rebase=true` 和 `pull.ff=only` 在团队级文档要求
- [ ] `rerere.enabled=true` 推荐开
- [ ] 仓库级 `core.hooksPath` 指向 `.githooks/` 目录，hook 入版本控制

### 分支保护
- [ ] main / master 禁止直接 push
- [ ] PR 必须至少 1 个 reviewer approve
- [ ] PR 必须 CI 绿才能合
- [ ] 禁止 force push 到保护分支
- [ ] 禁止删除保护分支
- [ ] PR 合并方式只允许一种（团队选 squash 或 rebase）

### Commit 质量
- [ ] 遵循 Conventional Commits（`feat:` / `fix:` / `chore:` 前缀）
- [ ] commit message 标题 < 72 字符，body 解释 "why" 不只是 "what"
- [ ] 每个 commit 是"原子"的：一个完整的小变更，能独立 revert
- [ ] 不在 commit message 里堆 issue / PR 链接，用 `Refs:` `Closes:` 字段

### Hook 与自动化
- [ ] pre-commit 拦大文件（> 10 MB）
- [ ] pre-commit 跑 lint / format / 类型检查
- [ ] commit-msg 校验 message 格式（用 commitlint）
- [ ] pre-push 跑相关单元测试
- [ ] CI 上重新跑全部 hook，本地 hook 不可信赖（开发者可 `--no-verify` 跳过）

### 安全
- [ ] 密钥扫描（git-secrets / trufflehog / gitleaks）入 CI
- [ ] 敏感目录（.env / credentials/）必须在 .gitignore
- [ ] 公开仓库前用 git-filter-repo 清理历史中可能的密钥
- [ ] GPG / SSH 签名 commit（`commit.gpgsign=true`）
- [ ] CODEOWNERS 文件维护，关键路径强制 review

### 大仓库性能
- [ ] 100 万文件以上启用 `partial clone` (`--filter=blob:none`)
- [ ] sparse-checkout 让开发只 checkout 自己负责的部分
- [ ] LFS 处理所有大文件
- [ ] 定期 `git gc --aggressive` 优化 packfile

### Trunk-based 专项
- [ ] PR 寿命 < 2 天
- [ ] CI 流水线 < 10 分钟
- [ ] feature flag 平台就位
- [ ] merge queue 启用
- [ ] main 红后自动 revert 或自动告警

### 灾备
- [ ] 远端仓库有镜像备份（不只依赖 GitHub / GitLab）
- [ ] 关键分支有定期快照导出
- [ ] 团队文档：误 push / 误 reset / 误删分支的恢复 SOP
- [ ] reflog 保留期延长（`gc.reflogExpire=180.days`）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Git Internals
- **Pro Git 第二版** 第 10 章 "Git Internals"：Scott Chacon 写的官方深度，从 plumbing / porcelain 到 packfile 的格式细节
- **Git from the Bottom Up** by John Wiegley：从对象模型自底向上理解，比 Pro Git 更"本质"
- packfile 的 v2 vs v3 格式、reachability bitmap、commit-graph 文件 —— 这些是 GitHub 处理 Linux kernel 级仓库的性能秘诀

### Conventional Commits & 自动化
- [Conventional Commits](https://www.conventionalcommits.org/) 规范：`type(scope): message` 的标准
- 配套工具：commitlint（校验）、commitizen（交互式 commit）、semantic-release（自动语义化版本）
- Changeset / changesets/cli：monorepo 下的"变更声明"机制

### GPG / SSH 签名
- `git commit -S` 用 GPG 签名 commit
- GitHub 2022 起支持 SSH 签名，比 GPG 易用
- **不签名的隐患**：仓库管理员能伪造任何 author 提交（`git commit --author="..."`）
- supply chain 攻击防护：sigstore / cosign 把签名做到容器 / 二进制层面

### Pre-commit hooks 体系
- [pre-commit.com](https://pre-commit.com/)：Python 写的 hook 管理框架，支持多语言
- husky（Node 生态）/ lefthook（Go 写的，跨语言更快）
- Trunk Check：把 lint / format / security 工具统一编排
- 服务端 hook：GitHub 的 push rules、GitLab 的 push rules

### CI / CD 集成
- GitHub Actions / GitLab CI / Jenkins 的 trigger 模型差异
- 增量 CI：只跑本次变更影响的测试（Bazel / Nx affected）
- merge queue / merge train 的实现原理
- 缓存策略：node_modules / Maven 依赖 / Docker layer

### Atlassian Git Tutorials
- Atlassian 出的 [Git Tutorials](https://www.atlassian.com/git/tutorials) 是公认最系统的入门-进阶资源
- "Comparing Workflows" 那一节对四种工作流的对比图至今最清晰
- 推荐看完 `Advanced Tips` 整个系列

### 下一代 VCS
- **Jujutsu (jj)**：Google 主导，"Git 兼容但模型更优"。每个修改自动是新 commit，无 staging area，conflict 不阻塞操作
- **Sapling**：Facebook (Meta) 出，与 Mercurial 同源，针对超大 monorepo 优化
- **Pijul**：基于补丁理论的 VCS，merge 是数学上无歧义的操作
- **Fossil**：SQLite 作者自用，自带 wiki / issue / forum，all-in-one

### 大型工程实践
- Google 的 "Why Google Stores Billions of Lines of Code in a Single Repository"（Communications of the ACM 2016）
- Microsoft 的 GVFS / Scalar，把 Git 改造支持 Windows 仓库（300GB +）
- Linux 内核的 patch-by-email 工作流（git format-patch + git send-email）—— 完全不用 PR
- Chromium 的 Gerrit + chromite 工作流：每个 commit 一个 review，与 PR 模型完全不同

---

## 7. 引用

- Pro Git, 2nd Edition, by Scott Chacon and Ben Straub. [git-scm.com/book](https://git-scm.com/book)
- Linus Torvalds, *Tech Talk: Linus Torvalds on Git* (Google Talks, 2007). YouTube 经典
- Vincent Driessen, *A successful Git branching model* (2010, addendum 2020)
- Trunk-Based Development. [trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/)
- Atlassian Git Tutorials: Comparing Workflows. [atlassian.com/git/tutorials/comparing-workflows](https://www.atlassian.com/git/tutorials/comparing-workflows)
- Google Engineering, *Why Google Stores Billions of Lines of Code in a Single Repository* (CACM, 2016)
- Aditya Mukerjee, *Why Sapling? Why now?* (Meta Tech Blog, 2022)
- Martin Fowler, *Patterns for Managing Source Code Branches* (martinfowler.com, 2020)
- `git-filter-repo` README & manual: [github.com/newren/git-filter-repo](https://github.com/newren/git-filter-repo)
- `pre-commit` documentation: [pre-commit.com](https://pre-commit.com/)
- Conventional Commits 1.0.0: [conventionalcommits.org](https://www.conventionalcommits.org/)
- Jujutsu (jj) tutorial: [github.com/martinvonz/jj](https://github.com/martinvonz/jj)

---

> 配套：[DevOps CI/CD](/article/devops/cicd) | [单元测试与 Mock](/article/testing/unit-testing) | [Trunk-based 与发布管理]()
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
