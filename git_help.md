# Git 入门手册

> 环境：Windows · Git 2.53.0 · 示例仓库 `C:\study_work\stu\RecW`
> 文中终端输出均为实际运行结果，可直接对照。

---

## 目录

1. [心智模型：一张图](#一心智模型一张图)
2. [日常 90% 的操作就五条](#二日常-90-的操作就五条)
3. [看改动：git diff 到底比的是什么](#三看改动git-diff-到底比的是什么)
4. [撤销：分清撤到哪一层](#四撤销分清撤到哪一层)
5. [分支](#五分支git-真正值钱的地方)
6. [冲突](#六冲突不是错误是-git-在等你裁决)
7. [在本仓库落地](#七在-recw-仓库上落地)
8. [速查表](#八速查表)
9. [常见坑](#九常见坑)
10. [练习路线](#十建议的练法)

---

## 一、心智模型：一张图

Git 只做一件事：**给整个项目拍快照**。一次 `commit` 就是一张快照，之后你能回到任何一张、也能从任何一张分叉出去。其他命令都是围绕"拍哪张 / 回哪张"。

```
    工作区                  暂存区                本地仓库
  你的文件夹    --add-->   (stage)   --commit-->  .git/ 里的历史
      ↑                                            │
      └──────────── restore / reset ←──────────────┘
```

- 改文件 = 只动了工作区，Git 还不关心
- `git add` = 告诉 Git"下次拍照要包含这个文件现在的样子"
- `git commit` = 真的拍一张
- `git status` = 方向盘。任何时候不确定，先敲它

**为什么中间非要夹一个"暂存区"？** 因为一次提交应该是**一件完整的事**，而不是"我顺手保存了一下"。你可能同时改了论文笔记、改了两个脚本，`add` 让你把"给论文笔记加一节"和"修 bug"拆成两条干净的记录，将来回看历史时才知道每次改动是为了什么。

---

## 二、日常 90% 的操作就五条

```powershell
git init               # 把一个文件夹变成仓库（只做一次）
git status             # 我现在处于什么状态
git add <文件>          # 把文件放进暂存区
git commit -m "说明"    # 拍一张快照
git log --oneline      # 看历史
```

真实跑一遍：

```
PS> git init -b main
Initialized empty Git repository in .../git-teach-6288/.git/

PS> git status --short        ← ?? 表示"未跟踪"，Git 还没管它
?? draft.txt
?? notes.txt

PS> git add notes.txt

PS> git status --short        ← A 表示已进暂存区
A  notes.txt
?? draft.txt

PS> git commit -m "docs: add first note"
[main (root-commit) cbe86f2] docs: add first note
 1 file changed, 1 insertion(+)
 create mode 100644 notes.txt

PS> git log --oneline         ← 每行 = 一张快照，前面是它的编号
cbe86f2 docs: add first note
```

`cbe86f2` 这种 7 位编号就是快照的身份证：

- `git show cbe86f2` —— 看这次改了什么
- `git reset cbe86f2` —— 回到那一步

不需要背它，允许只打前几位。

**提交说明怎么写？** 一句话说清"做了什么"，别写 "update"。推荐这个格式：

```
类型: 一句话描述

feat: 增加召回模块的对比实验
fix: 修正评测脚本里的分母错误
docs: 补充第二章实验设置
chore: 初始化项目
```

---

## 三、看改动：`git diff` 到底比的是什么

新手最容易绕晕的地方，记住两句话：

| 命令 | 比较对象 | 含义 |
|---|---|---|
| `git diff` | 工作区 ↔ 暂存区 | "我改了但还没 add 的东西" |
| `git diff --staged` | 暂存区 ↔ 上次提交 | "这次 commit 会真的写进去什么" |

```
PS> git diff
@@ -1 +1,2 @@
 hello git
+second line          ← + 是新增的行，- 是删掉的行
```

所以正确习惯是：

```
改完 → git diff 自己审一遍 → git add → git diff --staged 再确认 → commit
```

Git 不是自动保存，**你审过的内容才该进历史**。

---

## 四、撤销：分清"撤到哪一层"

这里搞错是新手最痛的，因为不同层要用不同命令，而且有一个会真删东西：

| 你想撤销什么 | 命令 | 后果 |
|---|---|---|
| 文件改了，还没 add | `git restore 文件名` | 改动直接消失，回到上次提交的样子 |
| 已经 add，想拿出暂存区 | `git restore --staged 文件名` | 改动还在，只是不进这次提交 |
| 刚 commit 完，发现漏文件/写错说明 | `git add 漏的` → `git commit --amend` | 重写上一次提交，不产生新记录 |
| 撤销最近一次提交，保留改动 | `git reset --soft HEAD~1` | 提交消失，改动退回暂存区 |
| 彻底丢掉最近一次提交和改动 | `git reset --hard HEAD~1` | ⚠️ 改动真的没了，未提交的部分救不回 |

实跑出来的两个关键动作：

```
PS> git restore --staged app.py     ← 先出暂存区，改动仍在
 M app.py

PS> git restore app.py              ← 再丢改动，回到 v1
v1
```

**关于 `--amend` 的副作用**：它会改写上一次提交，编号会变。只要那次提交还没推到远程、没别人拉过，就完全安全；推过之后再 amend 会导致本地和远程分叉，需要 `git push --force`（危险操作，协同时别用）。

---

## 五、分支：Git 真正值钱的地方

分支不是复制文件夹，它只是一个**指向某张快照的便签**，所以创建成本几乎为零。典型用法：想大改一个模块，又怕把能跑的版本弄坏，就开个分支折腾，成了再合回来，不成直接删掉。

```
PS> git switch -c feature/login      ← 新建并切过去
PS> git switch main                  ← 切回来
PS> git merge feature/login          ← 把那个分支的活并进来
Fast-forward
 notes.txt | 1 +
 1 file changed, 1 insertion(+)
```

`Fast-forward` 的意思是：你切走之后 main 没动过，所以 Git 直接把便签往前挪一格，不需要真的"合并"。如果两边都动过，才会产生一次真正的合并提交。

看历史结构最好用这张图：

```
PS> git log --oneline --graph --all
*   8ae5062 merge: resolve conflict
|\
| * 37fd04b fix: A version
* | d9f1441 fix: B version
|/
* 196920f feat: login draft
```

---

## 六、冲突：不是错误，是 Git 在等你裁决

你和分支都改了同一个文件的同一处，Git 不敢替你决定，就把两边都写进文件让你选。这是**正常流程**，不是报错。

```
PS> git merge fix-a
CONFLICT (content): Merge conflict in notes.txt
Automatic merge failed; fix conflicts and then commit the result.

PS> git status --short
UU notes.txt          ← UU = 双方都改了
```

文件里现在长这样：

```
hello git
<<<<<<< HEAD
from branch B          ← 你当前分支的版本
=======
from branch A          ← 要合进来的版本
>>>>>>> fix-a
```

处理只有三步：

1. **手工把文件改成你最终想要的样子** —— 把 `<<<<<<<`、`=======`、`>>>>>>>` 三行连同不要的那半边一起删掉
2. `git add notes.txt` —— 这一步是告诉 Git"我裁完了"
3. `git commit`

如果搞乱了想重来：`git merge --abort`，一切回到合并前。

---

## 七、在 RecW 仓库上落地

### 现状

- 分支是 `master`，**一次提交都没有** —— 所有文件都还是"未跟踪"
- `user.name` 是空的，`user.email` 已是 `2128076613@qq.com`。**不补名字，第一次提交会被拒绝**
- 没有 `.gitignore`
- 内容：`lit/`（3 篇 PDF，6.4 MB）、`recworld_handover-master/`（59 个文件）、`src/`（一个 zip）、`recworld_paper_o.pdf`、一份中文笔记

### 检查清单

- [ ] **第一步：补身份**（只需一次，`--global` 对整机生效）

  ```powershell
  git config --global user.name "你的名字"
  ```

  不设的话提交会直接失败：

  ```
  Author identity unknown
  *** Please tell me who you are.
  ```

- [ ] **第二步：默认分支改名**（可选，和 GitHub 现在的习惯一致）

  ```powershell
  git branch -m main
  ```

- [ ] **第三步：写 `.gitignore`**

  **这一步必须在第一次 add 之前做**，因为 Git 会一直记着你曾经提交过的东西，事后补 ignore 也删不掉历史。

  ```gitignore
  # Python
  __pycache__/
  *.py[cod]
  .venv/
  venv/
  .ipynb_checkpoints/

  # 密钥（研究代码最容易踩的坑）
  .env
  *.env
  config.local.*

  # 编辑器 / 系统
  .vscode/
  .idea/
  .DS_Store
  Thumbs.db
  *.tmp

  # 跑实验的产物
  *.log
  runs/
  out/
  ```

  至于那 6.4 MB 的论文 PDF 和 zip：总体积不大，提交进仓库完全没问题，好处是换台电脑就有。唯一代价是 **Git 不擅长存二进制** —— 以后每改一次 PDF，仓库就多存一整份。如果打算把论文和代码分开管，就在 `.gitignore` 里加两行：

  ```gitignore
  *.pdf
  *.zip
  ```

- [ ] **第四步：第一次提交**

  ```powershell
  git add .
  git status                    # ← 逐行看一遍，确认没有 .env、没有临时文件
  git commit -m "chore: 初始化 RecW 项目"
  git log --oneline
  ```

  `git add .` 之后**一定要用 `git status` 复核清单**，这是防止误提交密钥的最后一道门。

- [ ] **第五步（可选）：推到 GitHub**（等你要云端备份时再做）

  ```powershell
  git remote add origin https://github.com/你的用户名/RecW.git
  git push -u origin main       # 第一次要 -u，之后直接 git push
  ```

  如果这个仓库只在自己电脑上，**不做第五步也完全成立** —— 本地 Git 本身就是完整的版本控制，远程只是备份和协作。

---

## 八、速查表

| 我想… | 命令 |
|---|---|
| 看当前状态 | `git status` |
| 看历史（带图） | `git log --oneline --graph --all` |
| 看某次提交改了什么 | `git show <编号>` |
| 看工作区改动 | `git diff` |
| 看将要提交的内容 | `git diff --staged` |
| 撤销工作区改动 | `git restore <文件>` |
| 把文件移出暂存区 | `git restore --staged <文件>` |
| 改上一次提交 | `git commit --amend` |
| 开分支 / 切分支 | `git switch -c <名>` / `git switch <名>` |
| 合并 | `git merge <分支名>` |
| 合并搞砸了 | `git merge --abort` |
| 临时手头活放一边 | `git stash` → `git stash pop` |
| 看某行是谁改的 | `git blame <文件>` |
| 查某个文件的历史 | `git log --follow -p <文件>` |

---

## 九、常见坑

**1. `git status` 里中文文件名变成八进制转义**

在 RecW 仓库里会看到这样的输出：

```
"\351\235\242\345\220\221\346\216\250\350\215\220..."
```

那是 `面向推荐系统的大模型用户模拟.md`。修一下就好：

```powershell
git config --global core.quotepath false
```

**2. 换行符警告**

```
warning: in the working copy of 'notes.txt', LF will be replaced by CRLF
```

Windows 上 Git 默认要做换行符转换，单机用没影响；如果要和别人共享仓库，统一一下：

```powershell
git config --global core.autocrlf true
```

**3. 提交了不该提交的东西**

- 只是没 add 过：写进 `.gitignore` 就好
- 已经 commit 了：`git rm --cached <文件>` 再 commit，文件从历史消失但留在磁盘上
- 已经 push 了：历史里永远有它。如果里面是**密钥**，第一件事不是删历史，而是**立刻去服务商后台作废那个 key**

**4. 删除 `.git` 文件夹 ≠ 重置**

`rm -rf .git` 会连历史一起抹掉，项目退回成普通文件夹。这是"我彻底不想要版本控制了"才会用的操作，别拿它当撤销。

**5. Windows 下文件名大小写**

Windows 文件系统不区分大小写，`readme.md` 改成 `README.md`，Git 可能察觉不到。需要显式告诉它：

```powershell
git mv readme.md README.md
```

---

## 十、建议的练法

别背命令，按这个顺序练，每一步都能看到 `git status` 的变化，手感就来了：

1. 在这个仓库里完成第七节的第一到第四步，得到第一次提交
2. 改一行笔记 → `git diff` → `git add` → `git commit -m "docs: 补充..."`
3. 故意把某个文件改坏 → `git restore` 恢复，体会"改动真的消失"
4. 开一个分支改点东西，切回 main 看文件变回去，再 `merge` 回来
5. 故意制造一次冲突（两个分支改同一行），手工裁决一次

**做完第 5 步，Git 就不再神秘了。**

---

## 附：一句话记住

> `add` 是挑，`commit` 是拍，`log` 是翻，`switch` 是换桌，`merge` 是并桌，
> `restore` 是撤回未拍的，`reset` 是撕掉已拍的。撕之前想清楚。
