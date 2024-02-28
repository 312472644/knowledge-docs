# git常用命令

## git config

Git配置相关的指令，像配置user、email等。

```powershell
# 检查一下用户名和邮箱是否配置
git config --global --list

# 设置全局用户 
git config --global user.name 'your name' 
git config --global user.email 'xxxxx@example.com'

# 设置局部用户（在指定项目目录下，可配置不同项目有不同的git用户）
git config user.name 'ccc' 
git config user.email 'xxxxx@example.com' 

# 设置全局的一次性修改配置 
git config --global --edit
```

## git 生成密钥

```powershell
ssh-keygen -t rsa -c 'xxxxx@example.com'
```

## git remote

```powershell
# 初始化仓库
git init

# 查看已关联的仓库 
git remote -v

# 关联远程仓库 
git remote add origin https://github.com/xxxx/xxxx.git

# 删除远程仓库 
git remote remove origin

# 直接修改远程仓库 
git remote set-url origin https://github.com/xxxx/xxxx.git

# 查看远程仓库的详细信息 
git remote show 
git remote show origin
```

## git clone

```powershell
# git clone <远程仓库地址>
git clone http://github.com/xxx/xxx.git

# git clone <远程仓库地址> <本地目录名> 
git clone http://github.com/xxx/xxx.git <project_name>
```

## git branch

```powershell
# 查看本地仓库所有分支
git branch
# 查看远程仓库所有分支
git branch -r
# 查看本地和远程仓库的所有分支
git branch -a

# 基于当前分支，新建一个分支
git branch <new_branch_name>

# 删除分支：被删除分支 --是-- 基于当前分支新建出来的，可用-d 或 -D
git branch -d <local_branch_name>

# 删除分支：被删除分支 --不是-- 基于当前分支新建出来的，只能使用 -D
git branch -D <local_branch_name>

# 删除远程分支
git push origin --delete <remote_branch_name>

# 重命名当前分支
git branch -m <new_name>
```

## git checkout

```powershell
# 切换到本地分支 
git checkout <local_branch_name> 

# 切换到指定某次提交的commit_id 
git checkout <commit id> 

# 从现有分支中新建新的分支，并切换到新分支 
git checkout -b <new_branch_name> 

# 从远程分支中新建新的分支，并切换到新分支 
git checkout -b <new_branch_name> origin/<remote_branch> 

# 放弃工作区的修改，只影响工作区 
git checkout . 
# 放弃工作区和暂存区的修改，影响工作区和暂存区 
git checkout -f
```

## git status

查看工作区和暂存区的状态。

```powershell
git status
```

## git add

将工作区的修改添加到暂存区。

```powershell
# 将工作区的所有修改提交到暂存区 
git add .

# 将指定目录添加到暂存区，包括子目录所有修改
git add [dir]

# 将src目录下的所有js文件添加到暂存区
git add src/**/**.js
```

## git commit

将暂存区的修改，提交到本地仓库

`git commit -m` 与 `git commit -am` 的区别

- `-m`: 是将暂存区的修改，提交到本地仓库
- `-am`: 将暂存区的修改提交到本地仓库之前，多了一步操作：先把本地的变动提交到暂存区，所以它是将暂存区和工作区的修改，提交到本地仓库。

**注意：** 这里需要注意一点，工作区提交到暂存区的变动文件，是已经与远程版本库有了关联的（即之前已经提交到远程的），该命令对新增的文件，是不起作用的，所以提交新增的文件，需要执行`git add .`  指令。

```powershell
# 将暂存区的修改，提交到本地仓库 
git commit -m '提交信息'

# 将暂存区和工作区的修改，提交到本地仓库 
git commit -am '提交信息'

# 避开钩子函数的检查，强制提交 
git commit -m '提交信息' --no-verify 
git commit -m '提交信息' -n

# 将暂存区的修改，加到上一次的commit中，进入commit编辑，输入:wq 退出 
git commit --amend

# 修改上一次提交的commit信息（未push到远程仓库） 
git commit --amend --only -m '新的提交信息'
```

## git pull/push 拉取/提交

拉去远程仓库分支到本地仓库，或者推送本地仓库分支到远程分支。

```powershell
# 拉取代码，将远程仓库分支同步到本地 
git pull

# 将本地仓库的分支推送到远程分支（建立在本地分支追踪远程分支基础上） 
git push

# 推送到远程分支，并设置本地分支跟踪的远程分支 
git push --set-upstream origin <remote_branch> 
git push -u origin <remote_branch>
```

## git merge 分支合并

合并本地仓库的其他分支某个分支到当前分支。

```powershell
# 把本地仓库的某分支合并到当前分支 
git merge <local_branch> 

# 取消合并 
git merge --abort
```

## git stash 本地存储

将工作区的改动（未commit），临时存储在本地。

```powershell
# 默认按stash的顺序命名: stash@{n} 
git stash 

#添加备注 
git stash save 'message' 

# 查看存储列表 
git stash list 

# 应用最近一次的stash 
git stash apply 

# 应用指定的那一条 
git stash apply stash@{n} 

# 应用最近一次的stash，随后删除该记录 
git stash pop 

# 删除stash的所有记录 
git stash clear
```

## git log 日志过滤

主要用于查看Git版本演变历史（也就是提交历史），同时根据追加的参数和选项不同，过滤出想要的结果。

**参数说明：**

```powershell
按数量过滤： 
    -n: 显示前 n 条记录 
    shortlog -n：按作者分类，过滤出前 n 条 
按时间过滤： 
    --after=: 如 --after='2023-08-30'，显示 2023-8-30 之后的提交记录（包含8-30当天） 
    --before=: 如：--before='2023-08-30', 显示 2023-8-30 之前的提交记录（不包含8-30当天）
    before/after 是个相对时间，可以这么写：--after='a week ago', --after='yesterday' 
按作者过滤： 
    --author=: 作者名不需要精确匹配，只需要包含就行了，可使用正则匹配 
按commit信息过滤： 
    --grep='关键字': 过滤出记录中(commit提交时的注释)与关键字有关的记录 
过滤merge commit:
    --no-merges: 过滤出不包含 merge 的记录 
    —merges: 只过滤出包含 merge 的记录 


-p：按补丁显示每个更新文件的差异，比下一条 --stat命令信息更全 
--stat：统计出每次更新的修改文件列表, 及改动文件中 添加/删除/变动 的行数 
--pretty=：使用其他格式显示统计信息，参数有点复杂，目前很少用到
```

**常用指令：**

```powershell
git log

# 查看所有的提交记录
git log -all

# 将记录一行一行的形式展示：简洁明
git log --oneline

# 记录以图形化的形式展示
git log --graph

# 显示每次更新的文件修改统计信息，会列出具体文件列表
git log --stat

# 展示前10条
git log -10 
# 按作者分类，过滤出前10条
git shrtlog -10 
# 过滤出 'xxx' 的前10条记录，不包括 merge的记录
git log --author='XXX' -10 --no-merges 
# 过滤出 commit 提交的注释中  包含'feat'关键字的前10条记录，不包括merge 的记录
git log --grep='feat' -10 --no-merges
```

## git revert 代码撤销

撤销指定的提交，并产生一个新的commit，但保留了原来的commit记录。

```powershell
# 撤销指定的提交版本 
git revert <commit_id>

# 撤销的版本范围 
git revert <commit_id1>..<commit_id2>

# 撤销上一次提交 
git revet HEAD

# 撤销上上次提交 
git revet HEAD^
```

## git reset 代码回滚

让代码回滚到指定的提交版本，并且不保留原来的commit记录。

```powershell
# 仅是撤销commit记录，所有改动都保留（工作区和暂存区） 
# HEAD^ 代表上个版本 
git reset --soft HEAD^ 
git reset --soft commit_id

# 撤销commit记录，不保留改动，直接回退到指定的提交版本 
git reset --hard HEAD^ 
git reset --hard commit_id

# 强推到远程 
git push origin dev --force
```

## git tag 版本号管理

```powershell
# 列出所有标签 
git tag

# 默认在 HEAD 上创建一个标签 
git tag v1.0.0

# 指定一个 commit id 创建一个标签 
git tag v1.0.0 <commit_id>

# 创建带有说明的标签，用 -a 指定标签名，-m 指定说明文字 
git tag -a v1.0.0 -m "说明文字"

# 查看单个标签具体信息 
git show <tagname>

# 推送指定的本地标签到远程仓库 
git push origin <tagname>

# 推送本地未推送的所有标签到远程仓库 
git push origin --tags

# 删除本地标签 
git tag -d v1.0.0

# 删除一个远程标签 
git push origin tag --delete <tagname>
```