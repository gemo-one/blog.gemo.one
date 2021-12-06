---
title: "Git 使用 --depth 之后"
date: 2021-11-05T10:53:15+08:00
description: "Git 关于 --depth 那些事，通过 --depth 之后再获取其他分支。"
keywords:
    - Git
    - depth
categories:
    - 技术
    - IT 小记
tags:
    - IT
    - Git
    - Version Control
---

在我们拉取 `Git repo` 时，如果不需要大量的日志记录，只想获取某个分支或者某次提交下的文件状态，可以借助 `depth=1` 参数实现。

## 使用

在使用 `git clone` 时直接携带 `depth`:

```shell
# 实现默认分支的拉取
git clone --depth 1 xxx.git

# 某个分支/标签
git clone --depth 1 -b "branch_name/tag_name" xxx.git
```

其中 `depth` 指定的是 `commit` 的深度，指定 `1` 则表示只拉取一次 `commit`，就是某次提交的完整状态。如果指定 `2`，就表示只想更深一层的 `commit`。

此处借助 [`vue`](https://github.com/vuejs/vue.git) 仓库作为参考:

### 默认状态

默认拉取主分支：

```shell
git clone --depth 1 https://github.com/vuejs/vue.git
```

进入 `vue`，查看日志:

```shell
git log --all
```

只有一次提交:

```
commit c1740aca77170d0bc5ff7972bd05933ed307321c (grafted, HEAD -> dev, origin/dev, origin/HEAD)
    ...
```

但实际上其有多次提交，我们还可以针对某次提交进行拉取。

### 拉取某次提交

> 实际上此操作与 `depth` 参数无关，只做案例。

`--branch` 接受 `Tag Name` 参数，需要对 `commit` 打标签才可以直接操作，如果需要直接使用 `commit`，可能需要全量拉取后在 `git checkout` 获取。

此处我们尝试使用拉取 [`v2.6.0`](https://github.com/vuejs/vue/commit/85548310f19975745c83ac173f700f8b2f1e8e3e) ，该分支的 `commit id` 为 `8554831`:

```shell
git clone --depth 1 -b v2.6.0 https://github.com/vuejs/vue.git
```

日志也仅有一条记录:

```shell
commit 85548310f19975745c83ac173f700f8b2f1e8e3e (grafted, HEAD, tag: v2.6.0)
    ...
```

如果指定 `depth=2`，则应有两条记录:

```shell
commit 85548310f19975745c83ac173f700f8b2f1e8e3e (HEAD, tag: v2.6.0)
    ...
commit 076dc8d84facb3a62700b73e69804eca9f47ae50 (grafted)
    ...
```

也可以使用 `git pull --depth 2` 获得以上的效果。

## 获取其他内容

在使用了 `git clone --depth 1` 拉取仓库的前提下，可以通过 `git pull --depth x` 或者 `git fetch --depth x` 可以获取当前提交之前的 `x` 层日志。

但如果使用 `git branch -a` 并不能查看到远程的分支，如果我们需要使用到远程的所有分支，或者说获取到某个远程的分支，应该怎么操作呢?

### 获取全部分支

需要从远程拉取当前分支或提交之前的所有记录，同时还会将远程的分支信息以及标签信息都同步到本地来，以便进行后续操作。

其实该需求就是将单一记录的仓库转换成完整的仓库，`git` 提供 `unshallow` 参数用以实现该功能。

```shell
git pull --unshallow
# or
git fetch  --unshallow
```

此时将会成远程拉取完整信息，`git pull --unshallow` 最终的结果与默认使用 `git clone` 仓库的效果一致。

> 当然这个过程也会十分漫长。

### 获取某个分支

获取某个分支也并不困难，但需要知晓该分支的名称，因为使用 `--depth` 参数之后将不会拉取远程的其他分支名称。

同样以 [`vue`](https://github.com/vuejs/vue.git) 为例，此时以我获取 [`v2.6.0`](https://github.com/vuejs/vue/commit/85548310f19975745c83ac173f700f8b2f1e8e3e) 后作为起点，去尝试获取拉取 `dev` 分支下的内容。

进入 `vue` 执行：

```shell
# 在本地声明远程有一个分支叫做 dev
git remote set-branches origin 'dev'

# 同步该分支
git fetch --depth 1 origin dev
# 迁出分支
git checkout -b dev origin/dev
```

> 其中 `origin` 是默认拉取的远程名称，你也可以设置其他远程名称。

效果如下:

```
❯ git fetch --depth 1 origin dev
remote: Enumerating objects: 375, done.
remote: Counting objects: 100% (375/375), done.
remote: Compressing objects: 100% (152/152), done.
remote: Total 198 (delta 148), reused 73 (delta 43), pack-reused 0
Receiving objects: 100% (198/198), 164.34 KiB | 139.00 KiB/s, done.
Resolving deltas: 100% (148/148), completed with 137 local objects.
From https://github.com/vuejs/vue
 * branch            dev        -> FETCH_HEAD
 * [new branch]      dev        -> origin/dev
```

```
❯ git checkout -b dev origin/dev
Previous HEAD position was 8554831 build: release 2.6.0
Branch 'dev' set up to track remote branch 'dev' from 'origin'.
Switched to a new branch 'dev'
```

此时查看日志:

```
❯ git log -a
commit c1740aca77170d0bc5ff7972bd05933ed307321c (grafted, HEAD -> dev, origin/dev)
    ...
```

与远程 `dev` 的最后一条提交记录同步。

## 小结

参数 `depth` 与 `branch` 可以用于 `git` 拉取代码包括但不限于 `clone` 与 `submodule add` 操作当中，有时候我们需要定制些其他操作，需要执行获取其他分支或者抓取其他记录，都需要借用 `git` 相关的参数实现。

然而这些指令不不存在某个文档当中，哪怕你用 `man git-clone` 都没看见相应的提示。(当然那是文档，不是教程。)

此处做了一部分记录:

```shell
# 转换为完整仓库
git pull --unshallow

# 获取某个分支
# 在本地声明远程有一个分支叫做 dev / 或其他名称
git remote set-branches origin 'dev'
# 同步该分支
git fetch --depth 1 origin dev
# 迁出分支
git checkout -b dev origin/dev
```

## 参考

1. [知乎: git clone --depth=1之后拉取其他分支](https://zhuanlan.zhihu.com/p/350155254)
2. [风轻云断: git clone --depth=1 后获取其他分支](https://www.cnblogs.com/zhangyiqiu/p/12295572.html)
3. [git-doc: git-clone](https://git-scm.com/docs/git-clone)
