# git 命令学习

# 基础操作

```bash
# 初始化仓库
git init 

# 查看文件状态
git status 

# 添加到暂存区
git add .

# 提交到本地储存区
git commit -m 'init'

# 查看提交记录
git log 
# 简化查看提交记录
git log --pretty=oneline

# 回退到某一提交版本
git reset --hard commit_id
# 回退到上一版本 ^一个指一次
git reset --hard HEAD^

# 所有的日志查询(版本回退导致部分提交消失)
git reflog 

# 暂存区回滚到提交区最新状态 
git reset HEAD 文件名

```



# 分支操作

```bash
# 创建分支并切换到最新分支
git checkout -b dev 

# 查看分支
git branch

# 删除分支
git branch -d dev

# 强制删除分支
git branch -D dev

# 分支合并
git merge dev

# 冲突解决
手动解决完冲突之后在进行提交


```



# 标签

```bash
# 在最新版本上的 commit 打标签
git tag v1_tag

# 在历史版本是打标签
git tag v1_tag commit_id

# 在tag上加上说明
git tag v2.0 -m "这里打上了一个标签"

# 查看标签
git tag

# 删除标签
git tag -d v1_tag
```

