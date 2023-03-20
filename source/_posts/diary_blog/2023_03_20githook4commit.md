---
title: 利用 git hooks 自动追加 commit 前缀
toc: true
date: 2023-03-20 15:30:16
tags:
  - other
  - blog
categories:
  - other

---

关于实现 git commit 自动添加分支名为前缀。（全局）

<!--more-->


```
git config --global -l
git config --global core.hooksPath "~/.git/hooks/"
```


```bash
#!/bin/sh
#
# An example hook script to check the commit log message.
# Called by "git commit" with one argument, the name of the file
# that has the commit message.  The hook should exit with non-zero
# status after issuing an appropriate message if it wants to stop the
# commit.  The hook is allowed to edit the commit message file.
#
# To enable this hook, rename this file to "commit-msg".

# Uncomment the below to add a Signed-off-by line to the message.
# Doing this in a hook is a bad idea in general, but the prepare-commit-msg
# hook is more suited to it.
#
# SOB=$(git var GIT_AUTHOR_IDENT | sed -n 's/^\(.*>\).*$/Signed-off-by: \1/p')
# grep -qs "^$SOB" "$1" || echo "$SOB" >> "$1"

# This example catches duplicate Signed-off-by lines.

# 获取当前分支
line=$(head -n +1 .git/HEAD)
branch=${line##*/}
if [[ "$branch" =~ [0-9]+\.[0-9]+\.[0-9]+(.*) ]];  then
	branch=$(echo "$branch" | sed -E 's/^[0-9]+\.[0-9]+\.[0-9]+_//g')
	echo "$branch"
fi
# 设置新的commit
commit=${branch}:$(cat $1)
echo "$commit" > "$1"
```



- [git pre-receive钩子 实现全局hook的问题? - 知乎 (zhihu.com)](https://www.zhihu.com/question/65604891/answer/232935144)
- [记一次Git hooks不生效的问题 | Yukino的杂记 (yukinoyukino.com)](https://yukinoyukino.com/archives/ji-yi-ci-githooks-bu-sheng-xiao-de-wen-ti)
- [使用git hook 为commit message添加前缀 - 简书 (jianshu.com)](https://www.jianshu.com/p/8f0549d70caf)

