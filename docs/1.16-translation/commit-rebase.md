# 多次 commit rebase 为一次进行提交

翻译过程中，难免会对同一个`PR` 进行多次提交，在最终的`PR` 里面就会出现多次提交，其中大部分提交的内容都是`fix translate type` 、`fix translation issue` 。如果合并到翻译分支，且`merge` 到上游仓库（有可能，这也是我们的目标），就会显得`commit` 信息杂乱。虽然我们`review` 团队在最终的`merge` 时候会选择`squash` ，但是良好的`commit` 习惯还是由开发人员自己养成。下面就些一个将多个`commit` 进行`rebase` ，将其变成一个节点进行提交。

### 第一步：查看 git log  确定要 rebase 的节点数

```text
$ git log

commit f09ff8904d95f68d6d50049ed559543e6ca108e9 (HEAD -> cloud-security-policy)
Author: majinghe <devops008@sina.com>
Date:   Fri Dec 18 17:13:15 2020 +0800

    fix translation issue

commit 4b59058477738dd9f12ac0d45a868acc87e0c16c (origin/cloud-security-policy)
Author: majinghe <devops008@sina.com>
Date:   Fri Dec 18 17:06:36 2020 +0800

    fix translation issue

commit 1f6ee9587df47922e304822244935746ebbce500
Author: majinghe <devops008@sina.com>
Date:   Fri Dec 18 16:42:55 2020 +0800

    add cloud security policy translation
```

### 第二步：执行 rebase 命令

执行 `git rebase -i HEAD~n` ，其中`n` 代表想要合并的`commit` 节点数量，本例子为`3` 。执行前述命令后会出现如下内容：

```text
$ git rebase -i HEAD~3

pick 1f6ee95 add cloud security policy translation
pick 4b59058 fix translation issue
pick f09ff89 fix translation issue

# Rebase d05c0ff..f09ff89 onto d05c0ff (3 commands)
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
```

具体意思大家可以详读上面的具体内容。接下来是比较重要的一步，将后面两个`commit` 的信息进行压缩，也就是将`pick` 改成`s` \(代表`squash` ）。

```text
pick 1f6ee95 add cloud security policy translation
s 4b59058 fix translation issue
s f09ff89 fix translation issue
```

然后根据提示，进行保存退出。在接下来出现的界面中，根据提示修改`commit` 信息：

```text
# This is a combination of 3 commits.
# This is the 1st commit message:

add cloud security policy translation

# This is the commit message #2:

fix translation issue

# This is the commit message #3:

fix translation issue

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
#
```

保留想要的`commit` 信息，不想保留的可以以`#` 进行注释即可。比如本例注释后面的两行，只保留第一行：

```text
# This is a combination of 3 commits.
# This is the 1st commit message:

add cloud security policy translation

# This is the commit message #2:

#fix translation issue

# This is the commit message #3:

#fix translation issue
```

保存并推出。

### 第三步：再次查看 git log

```text
$ git log

commit ce1339fa6862c440140b9f91ccfd67c4f17d04fe (HEAD -> cloud-security-policy)
Author: majinghe <devops008@sina.com>
Date:   Fri Dec 18 16:42:55 2020 +0800

    add cloud security policy translation

```

可以看到前三个`commit` 信息被整合成了一个。接下来只需要将修改推送至远端仓库即可。

### 第四步：推送至远端仓库

将`rebase` 结果推送至远端仓库。这儿需要注意的是，必须用`-f` 或者`--force` 来完成强制推送，否则会报如下错：

```text
! [rejected]        cloud-security-policy -> cloud-security-policy (non-fast-forward)
error: failed to push some refs to 'git@github.com:majinghe/cloudnative.to.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Integrate the remote changes (e.g.
hint: 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

使用`git push -f` 或者`git push --force` 来完成强制推送。（推送分支基于大家自己的选择）

随后即可看到，推送成功的信息

```text
Counting objects: 16, done.
Compressing objects: 100% (16/16), done.
Writing objects: 100% (16/16), 259.38 KiB | 891.00 KiB/s, done.
Total 16 (delta 6), reused 0 (delta 0)
remote: Resolving deltas: 100% (6/6), completed with 6 local objects.
To github.com:majinghe/cloudnative.to.git
 + 4b59058...ce1339f cloud-security-policy -> cloud-security-policy (forced update)
```

大家可以在`github` 对应的`PR` 上看到信息已经被合并成了一个。

