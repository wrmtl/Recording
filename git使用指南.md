#Git使用指南

## Git的认知

### Git的代码存储区域的划分

- 工作区
- 暂存区
- 本地版本库
- 远程版本库

## 工作区

- 用暂存区全部或指定的文件替换工作区的文件
  - git checkout .
  - git checkout -- [ filename ]

##提交到暂存区

- 添加一个或多个文件到暂存区
  - `git add [ file1 ] [ file2 ] ...`
- 添加指定目录到暂存区
  - `git add [ dir ]`
- 添加当前目录下的所有文件到暂存区
  - `git add .`
- 查看暂存区文件提交信息
  - `git status`
  - `git status -s`
- 重置暂存区的文件与当前的`commit`保持一致
  - `git reset --mixed HEAD [ filename]`
    - `--mixed`默认参数
    - `HEAD`：当前版本（本地版本库的最新一次的`commit`）
    - `HEAD^`：上一次的`commit`
    - `HEAD~3`：上上上一次的`commit`
    - `commit_id`：制定某次`commit`
- 将文件从暂存区删除
  - `git rm --cached [ filename ]`
    - 仅从暂存区删除
  - `git rm [ filename ]`
    - 从暂存区和工作区删除

##提交到本地版本库

- 提交暂存区到本地版本库
  - `git commit -m [ message ]`
- 提交指定文件到本地版本库
  - `git commit [ file1 ] [ file2 ] ... -m [ message ]`
-  撤销工作区中所有未提交的修改内容，将暂存区与工作区都回到上一次版本，并删除之前的所有信息提交
  - `git reset --hard HEAD`
  - 这个操作很危险

##提交到远程版本库

- `git push [远程主机名] [本地分支名]:[远程分支名]`
  - `git origin master:master`
  - 如本地与远程名称相同，可以省略远程分支名
- 如果本地版本与远程版本有差异，但又要强制推送可以使用`--force`参数
  - `git push --force origin master`
- 删除主机的分支可以使用`--delete`参数
  - `git push origin --delete master`
- 撤销远程版本库的提交
  - 采用通过本地`reset`的方式
    - `git reset --hard [ commit_id ]`
    - `git push origin HEAD --force`
- 从远程版本库获取代码并自动合并本地的版本
  - `git pull [远程主机名] [远程分支名]:[本地分支名]`
- 从远程版本库获取代码，手动合并
  - `git fetch origin`
  - `git merge origin/master`

##远程仓库管理

### 查看远程仓库信息

- 显示所有远程仓库
  - `git remote -v`
- 显示某个远程仓库的信息
  - `git remote show [ remote ]`
- 添加远程版本库
  - `git remote add [ shortname ] [ url ]`
- 其他相关命令
  - `git remote rm name`
  - `git remote rename old_name new_name`

## 分支管理

- 注意：切换分支时，当前分支未提交信息会被全被舍弃掉
  - 提交修改
  - 暂存修改
    - `git stash`
      - 暂存修改现场
    - `git stash pop`
      - 恢复修改现场，并删除暂存信息
    - `git stash applay@{0}`
      - 恢复修改现场`0`，但不删除暂存信息
- 分支管理操作
  - 查看所有分支
    - `git branch`
  - 创建分支
    - `git branch [ branchname ]`
  - 重命名分支
    - `git branch -m [ branchname ]`
  - 删除分支
    - `git branch -d [ branchname ]`
  - 切换分支
    - `git checkout [ branchname ]`
  - 分支合并
    - `merge`合并
      - `git checkout [ branchname ]`
      - `git merge master`
        - 将`master`分支合并到当前分支
      - 分支冲突
        - `git merge master`后出现冲突后，手动修改好冲突点
        - `git add .`添加手动合并的冲突点（这告诉了`git`已经解决好冲突点）
        - `git commit`
          - 自动合并分支，提示Merge brach 'master'
    - 变基操作：将自己的分支的分叉点重新校准到远程版本库的最新提交节点上
      - [参考教程：很形象](https://www.yiibai.com/git/git_rebase.html)
      - git checkout mywork
      - git rebase
        - 解决冲突
          - 手动合并冲突
          - `git add.`提交冲突
          - `git rebase --continue` `git`会自动应用余下的修改
          - `git rebase --abort`回到`rebase`前的状态

## 常见命令

- `git log`
  - 查看提交历史
- `git diff`
  - `git diff [ filename ]` 比较文件工作区和暂存区文件差异
  - `git diff [ id1 ] [ id2 ]` 比较两次提交间的差异
  - `git diff [ branch1 ] [ branch2 ]` 在两分支间进行比较
  - `git diff --cached` 比价暂存区和版本库差异
  - `git diff --staged` 比较暂存区和版本库差异
  - `git diff HEAD` 显示工作区和版本库差异
- `git blame [ filename ]`
  - 以列表形式查看文件的历史修改

##Git配置问题

###`Windows`系统下`git`中文乱码

- 支持中文字符集
  - `git`自带的`git bash`软件中，右键菜单`-> Options -> Text`修改：`Local`为`zh_CN` 、`Character set`为`UTF-8`

- 中文路径自动转义显示导致中文乱码：修改`quotepath`
  - `git config core.quotepath false`
  - 或者修改`.gitconfig`配置文件对应项