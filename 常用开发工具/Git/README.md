# 简介
代码版本管理工具。

# 原理图
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/常用开发工具/imgs/1.png">

# 常用命令
## stash
stash 命令能够将还未 commit 的代码存起来，让你的工作目录变得干净。

```
# 保存当前未commit的代码
git stash

# 保存当前未commit的代码并添加备注
git stash save "备注的内容"

# 列出stash的所有记录
git stash list

# 删除stash的所有记录
git stash clear

# 应用最近一次的stash
git stash apply
// 应用第二条记录
// git stash apply stash@{1}

# 应用最近一次的stash，随后删除该记录
git stash pop

# 删除最近的一次stash
git stash drop
```

## reset --soft
回退你已提交的 commit，并将 commit 的修改内容放回到暂存区。

一般我们在使用 reset 命令时，<code>git reset --hard</code> 会被提及的比较多，它能让 commit 记录强制回溯到某一个节点。而 <code>git reset --soft</code> 的作用正如其名，--soft (柔软的) 除了回溯节点外，还会保留节点的修改内容。

```
# 恢复最近一次 commit
git reset --soft HEAD^

# 恢复到特定的commit
git reset --soft [commitHash]
```
对于已经 push 的 commit，也可以使用该命令，不过再次 push 时，由于远程分支和本地分支有差异，需要强制推送 <code>git push -f</code> 来覆盖被 reset 的 commit。

## revert
将现有的提交还原，恢复提交的内容，并生成一条还原记录。

```
git revert [commitHash]
```
因为 revert 会生成一条新的提交记录，这时会让你编辑提交信息，编辑完后 :wq 保存退出就好了。

# 常见场景
1、拉取远程分支并创建本地分支
> git checkout -b 本地分支名x origin/远程分支名x

2、添加暂存区
> git add --all   将目录内所有内容提交到暂存区<br>
git add 文件夹/   提交整个目录<br>
git add xxxx.js xxxx.js （手动添加特定的文件）

3、将暂存区的内容提交
> git commit -m "（这里写注释）"		 

4、将master推送到远程库
> git push -u origin master

5、本地分支重命名
> Git branch -m oldbranchname newbranchname 本地分支重命名

# 参考资料
[Git不要只会pull和push，试试这5条提高效率的命令](https://juejin.cn/post/7071780876501123085)