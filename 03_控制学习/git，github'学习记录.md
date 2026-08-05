# 钟元 Git 学习记录——2026 年 8 月 6 日

## 8.6 Git 基础操作

### 一、今日完成内容

- 初始化本地 Git 仓库并完成第一次提交；
- 修改 `Core/Src/main.c` 中的 LED 闪烁次数并提交；
- 创建 `feature-blink` 分支，在分支中修改代码并完成合并；
- 使用 Hard Reset 将 `main` 分支回退到上一版本；
- 创建个人 GitHub 公开仓库，将本地工程推送到远程仓库；
- 在 GitHub 网页修改文件，并通过 Pull 将远程更新拉取到本地；
- 学习 Markdown 的标题、列表、表格、代码块和超链接语法；
- 使用 VS Code 的源代码管理和 Git Graph 插件查看提交历史。

### 二、Git 操作顺序

1. 使用 `git init` 初始化本地仓库；
2. 使用 `git add` 将文件加入暂存区；
3. 使用 `git commit` 创建本地提交；
4. 修改 `main.c` 并再次提交；
5. 创建并切换到 `feature-blink` 分支；
6. 在功能分支中修改代码并提交；
7. 切换回 `main`，比较并合并功能分支；
8. 使用 `git reset --hard` 体验版本回退；
9. 创建 GitHub 公开仓库并添加远程地址；
10. 将本地 `main` 分支推送到 GitHub；
11. 在 GitHub 网页修改文件并提交；
12. 使用 `git pull` 将远程更新同步到本地；
13. 编写本学习记录并提交、推送。

---

### 三、作业 1：第一次提交

#### 1. 完成内容

首先删除原工程中已有的 `.git` 文件夹，然后重新初始化仓库，将当前工程作为一个新的 Git 项目管理。

#### 2. 终端命令操作

```bash
git init
git add .
git commit -m "first commit"
```

其中：

- `git init`：在当前文件夹中创建 Git 仓库；
- `git add .`：将当前目录中未被 `.gitignore` 忽略的文件加入暂存区；
- `git commit`：把暂存区中的文件保存为一次提交。

#### 3. VS Code 图形化操作

1. 使用 VS Code 打开工程根目录；
2. 点击左侧“源代码管理”图标；
3. 如果仓库还未初始化，点击“初始化仓库”；
4. 在“更改”列表中点击“暂存所有更改”；
5. 在提交消息框中填写：

```text
first commit
```

6. 点击“提交”。

---

### 四、作业 2：修改并提交

#### 1. 完成内容

打开：

```text
Core/Src/main.c
```

将：

```c
#define BLINK_TIMES 3U
```

修改为：

```c
#define BLINK_TIMES 5U
```

修改后，LED 的闪烁次数由 3 次变为 5 次。

#### 2. 终端命令操作

```bash
git add Core/Src/main.c
git commit -m "将闪烁次数调整为5"
```

#### 3. VS Code 图形化操作

1. 修改并保存 `main.c`；
2. 打开“源代码管理”；
3. 在 `main.c` 右侧点击 `+`，将文件加入暂存区；
4. 输入提交消息：

```text
将闪烁次数调整为5
```

5. 点击“提交”。

---

### 五、作业 3：分支与合并

#### 1. 完成内容

创建 `feature-blink` 功能分支，在该分支中修改 LED 闪烁延时，并将修改合并回 `main`。

分支的作用是让新功能的开发与主分支暂时分离，功能完成后再合并。

#### 2. 终端命令操作

创建并切换分支：

```bash
git checkout -b feature-blink
```

在分支中修改代码并提交：

```bash
git add Core/Src/main.c
git commit -m "fb1"
```

切换回主分支：

```bash
git checkout main
```

查看提交历史和分支差异：

```bash
git log --oneline
git diff main feature-blink
```

将功能分支合并到主分支：

```bash
git merge feature-blink
```

#### 3. VS Code 图形化操作

1. 点击 VS Code 左下角的当前分支名 `main`；
2. 选择“创建新分支”；
3. 输入：

```text
feature-blink
```

4. 在新分支中修改并保存 `main.c`；
5. 在源代码管理中暂存并提交；
6. 点击左下角分支名，切换回 `main`；
7. 按 `Ctrl+Shift+P`；
8. 搜索并执行：

```text
Git: Merge Branch
```

9. 选择 `feature-blink`。

本次合并属于 Fast-forward 快进合并，因此没有额外生成 Merge Commit，而是让 `main` 直接移动到功能分支的最新提交。

#### 4. Git Graph 图形化查看

使用 Git Graph 插件可以看到：

- `main` 和 `feature-blink` 分支的位置；
- 每次提交的提交消息；
- 分支创建和合并后的提交关系；
- 两个分支是否指向同一个提交。

---

### 六、作业 4：体验回退

#### 1. 完成内容

将 `main` 分支从最新提交回退到上一条提交，体验版本恢复操作。

#### 2. 终端命令操作

查看提交记录：

```bash
git log --oneline
```

回退一条提交：

```bash
git reset --hard HEAD~1
```

其中：

- `HEAD` 表示当前提交；
- `HEAD~1` 表示当前提交的上一条提交；
- `--hard` 表示同时恢复分支指针、暂存区和工作区文件。

#### 3. Git Graph 图形化操作

1. 确认当前分支是 `main`；
2. 打开 Git Graph；
3. 找到希望回到的旧提交；
4. 右键该提交；
5. 选择：

```text
Reset current branch to this commit
```

6. 重置方式选择：

```text
Hard
```

7. 确认操作。

#### 4. Soft、Mixed、Hard 的区别

| 重置模式 | 提交记录 | 暂存区 | 工作区文件 | 主要用途 |
|---|---|---|---|---|
| Soft | 回退 | 保留修改并保持暂存 | 保留修改 | 修改提交信息或重新提交 |
| Mixed | 回退 | 取消暂存 | 保留修改 | 重新选择需要提交的文件 |
| Hard | 回退 | 恢复到目标提交 | 恢复到目标提交 | 完全放弃后续修改 |

本次作业选择的是 `Hard`。

---

### 七、作业 5：远程仓库

#### 1. 完成内容

创建个人 GitHub 公开仓库，将本地工程上传到远程仓库，并练习 Push 和 Pull。

#### 2. 添加远程仓库的终端命令

```bash
git remote add origin https://github.com/2h34/git-github-.git
```

其中：

- `origin` 是远程仓库的本地名称；
- 后面的地址是 GitHub 仓库地址。

#### 3. VS Code 图形化添加远程仓库

1. 打开“源代码管理”；
2. 点击右上角 `...`；
3. 选择：

```text
远程 → 添加远程
```

4. 粘贴仓库地址：

```text
https://github.com/2h34/git-github-.git
```

5. 远程名称填写：

```text
origin
```

#### 4. Push 推送

Push 用于把本地新提交上传到 GitHub。

终端命令：

```bash
git push -u origin main
```

图形化操作：

1. 确认当前分支为 `main`；
2. 在 VS Code Git 图表中点击向上箭头“推送”；
3. 首次推送时选择 `origin`；
4. 根据提示发布当前分支。

#### 5. Pull 拉取

Pull 用于将 GitHub 远程仓库中的新提交下载并合并到本地。

终端命令：

```bash
git pull
```

本次操作先在 GitHub 网页修改 `README.md` 并提交，再回到 VS Code 执行 Pull。

图形化操作：

1. 确认当前分支为 `main`；
2. 打开“源代码管理”；
3. 点击右上角 `...`；
4. 选择“拉取”；
5. 拉取后，本地 `README.md` 中出现了远程新增内容。

---

### 八、作业 6：Markdown 学习记录

#### 1. Markdown 基本语法

一级标题：

```markdown
# 一级标题
```

二级标题：

```markdown
## 二级标题
```

无序列表：

```markdown
- 内容一
- 内容二
```

有序列表：

```markdown
1. 第一步
2. 第二步
```

超链接：

```markdown
[链接名称](链接地址)
```

代码块：

````markdown
```bash
git status
```