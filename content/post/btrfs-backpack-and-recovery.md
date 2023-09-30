---
title: "Btrfs 的备份与恢复"
date: 2023-05-28T16:50:57+08:00
categories:
    - 技术
    - IT 小记
tags:
  - Linux
  - Btrfs
  - Snapper
---

目前笔记本的磁盘格式是 `Btrfs`，已经用了三年多，配合 `snapper` 实现本地快照。

发行版本是 `Gentoo`，配置 `portage` `bash` 实现快照，可以很方便地置换快照内容。

## 转储快照

将快照保留在原磁盘内起到的作用并不大，万一磁盘发生损坏，而你的备份又是在本地，会导致数据丢失。

因此引入 ”异地备份“。

可以使用 [`pv`](https://www.ivarch.com/programs/pv.shtml) 获取查看传输的进度。

```shell
# 使用 pv 查看
btrfs send /.snapshots/2/snapshot/ | pv | btrfs receive /mnt/snapper/backpack/2022-05-08/

# 无 pv
btrfs send /.snapshots/2/snapshot/ | btrfs receive /mnt/snapper/backpack/2022-05-08/
```

## 自动快照

[`snapper`](https://zh.opensuse.org/SDB:Snapper_Tutorial) 是 `openSUSE` 提供的创建和管理快照的工具，可以实现定时快照。区分快照对(`pre` `post`)和单一快照(`single`)，其中快照对主要应用于应用安装前后备份，单一快照主要应用于定时快照。

> 只讨论 `Btrfs` 文件系统。

### `portage` 的快照

通过 `snapper` 可以很方便管理每个安装包编译前后的快照，如果编译之后出现问题，可以直接依据快照名称执行回退。

这对 `Gentoo` 编译操作来说是一个很实用的功能。

```shell
# /etc/portage/bashrc
case "${EBUILD_PHASE}" in
        preinst)
                if [ -z "${REPLACING_VERSIONS}" ]; then
                        DESC="Installing ${CATEGORY}/${PF}"
                else
                        DESC="Upgrading to ${CATEGORY}/${PF} replacing version(s) ${REPLACING_VERSIONS}"
                fi
                NUMBER=`snapper create -t pre -p -d "${DESC}" -c number`
                ;;
        postinst) 
                snapper create -t post --pre-number $NUMBER -d "${DESC}" -c number
                ;;
esac
```

但缺点是产生快照会特别多，管理起来相对复杂。

### 时间快照

[`Timeshift`](https://github.com/linuxmint/timeshift) 做的更直观些，恢复备份也更好操作。不过 `snapper` 功能也足够，我的需求主要是定时快照一下就行（`crontab + btrfs` 也可以？）。

```shell
# 创建两个盘对应的配置
snapper create-config /
snapper -c home create-config /home
```

无需特殊配置，将自动创建快照。

快照虽然简单，但恢复快照还是比较困难的，`snapper` 在 `openSUSE` 之外的系统操作回滚还是很麻烦的，毕竟 `yast2` 还是很强大的。

## 回滚操作

回滚操作是一个比较复杂的变更，使用 `snapper` 也是一个比较繁杂的操作。

需要注意尽量确认自己修改了哪个文件，需要回滚，尽量使用 `undochange` 命令进行回滚，`rollback` 需要配合额外的工具实现具体回滚（ `openSUSE` 正常回滚，会创建 `grub2` 条目，但其他系统修改 `/etc/fstab` 无法成功 ）。

因此文件数量较少，可以使用 `undochange` 撤销更改：

```shell
snapper undochange <修改前的快照编号>..<修改后的快照编号> <文件名>

# 如果需要比对快照，可以使用 status
snapper status <第一个快照编号>..<第二个快照编号> //第一个快照的创建时间要早于第二个
```

如果确实需要 `rollback` 到对应的快照，完全回滚到对应的快照号，则可以通过 `subvol` 特性实现。

> 手上需要一个支持 `btrfs` 格式挂载的系统维护盘，`gentoo` 的安装盘即可。

假设快照所在的磁盘标签为 `DISK`（磁盘格式为 `btrfs`）。

```shell
# subvol=/ 可以直接挂在显示子卷
mount /dev/disk/by-label/DISK /mnt/gentoo -o subvol=/
```

### 迁移 snapshot 目录

如果你创建的 `snapper` 快照目录存储在默认的 `/` 路径下，并没有单独分盘，可以考虑将其迁移出单独的盘，更好管理。若原先已分成单独子卷可以，可以跳过该步骤。

假设当前子卷架构如下：

| ID  |  gen   | top level | path  |
| --- | ------ | --------- | ----- |
| 2   | 876941 | 5         | @home |
| 1   | 876939 | 5         | @     |

而 `snapper` 原来存储快照在 `@/.snapshots` 目录下，即默认路径，未挂载单独目录。

`btrfs` 的子卷可以当作普通文件夹取管理：

```shell
mv @/.snapshots @snapshots
```

将 `@/.snapshots` 移动到根目录，其 `top level` 就会变化和 `@` 一致。

| ID  |  gen   | top level |    path    |
| --- | ------ | --------- | ---------- |
| 3   | 876913 | 5         | @snapshots |
| 2   | 876941 | 5         | @home      |
| 1   | 876939 | 5         | @          |

后续直接添加 `/etc/fstab` 条目即可：

```conf
LABEL=DISK     /.snapshots     btrfs   defaults,subvol=@snapshots,compress=zstd:8,ssd,space_cache=v2,discard=async         0 0
```

### 通过子卷快照回滚

不需要明确的回滚操作，通过子卷移动或者“快照的快照”，也可以方便地实现快照的回滚。

假设 `snapper` 快照号为 `326` ，则其子卷路径为 `@snapshots/326/snapshot`，可以通过 `btrfs subvolume list /mnt/gentoo | grep 326` 搜索路径。此处回滚 `root` 的快照

```shell
# 备份原有 root
btrfs subvolume snapshot @ @bak
btrfs subvolume delete @
# or
mv @ @bak

# 迁移快照
btrfs subvolume snapshot @snapshots/326/snapshot @
# or
mv @snapshots/326/snapshot @
```

迁移完成后重启即可。

## 小结

`btrfs` 目前已经很稳定，没有比较大的问题，快照管理也十分简单，所有快照都是子卷，子卷当文件管理也可以。

不过恢复比较麻烦，就需要重启设备，而且异地存储还需要目标机器是 `btrfs` 格式的磁盘。

> `kernel 5.x` 的时候出现过断电文件丢失，现在没遇到过这种情况了，可能是充电意识打满了？