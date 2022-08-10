## Git 命令

[Git-Book](http://git-scm.com/book/zh/v2)
[Pro Git（中文版）](http://git.oschina.net/progit/)
[猴子都能懂的Git入门](https://backlog.com/git-tutorial/cn/)

```java

git pull // 拉取代码
git push // 上传代码
git push review // 往 Gerrit 推送code review

git status

git reset --hard HEAD~ // 取消操作

git rebase master // 将master的内容拿到当前分支
git add myfile.txt // 接触冲突，并添加改动文件
git rebase --continue // 继续执行 rebase
git rebase --abort // 取消 rebase

```

- 文件增删

`mkdir testFile` 创建目录
`touch 123.txt` 创建文件
`echo "写入测试内容" >> 123.txt` 创建文件并追加写入内容

`git add 123.txt` 添加指定文件
`git add .` 添加所有
`git commit -m "提交信息"` 提交


- 查看历史提交记录

`git log` // 查看历史提交记录
`git log --oneline` // 查看历史记录的简洁版本

`git log --oneline --graph` // 开启分支合并拓扑图
   `--reverse` // 逆向显示所有日志
   `--author` // 只查找指定用户的提交日志

`git log --decorate` // 查看标签
`git blame [file]` // 查看指定文件的修改记录


- 分支创建合并

`git branch` // 查看本地分支
`git branch <branch_name>` // 创建分支
`git branch -d <branch_name>` // 删除分支
`git checkout <branch_name>` // 切换分支
`git checkout -b <branch_name>` // 创建并切换分支

`git checkout master` 切换到 master 分支
`git merge br_function` 将 br_function 合并到 master


- 查看远程库

`git remote` // 查看当前配置有那些远程仓库
`git remote -v` // 每个别名的实际链接地址
`git remote add [别名] [url]` // 添加远程库
`git remote rm [别名]` // 删除远程库


- 标签

`git tag` // 查看所有标签
`git tag v1.0` // 打标签
`git tag -a v1.0` // 打带注解的标签（推荐）
`git tag -a v1.0 85fc7e7` // 指定节点打标签
`git tag -a v1.0.1 -m "自定义信息"` // 指定标签信息
`git tag -s v1.0.1 -m "自定义信息"` // PGP签名标签




---
