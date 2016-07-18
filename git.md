#### git 列出log的所有信息在一行显示

`git log --pretty=oneline`

#### 你可以通过下面的命令, 让工作目录回到上次提交时的状态(last committed state):
`git reset --hard HEAD`

#### 将本地tags推送到远程仓库

`git push origin [tagname]`  //推送指定tag到远程

`git push origin --tags`    //推送所有的tag到远程

### git 重命名一个分支

`git branch -m feature/fixBug`
