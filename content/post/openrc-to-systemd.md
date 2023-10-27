---
title: "Gentoo 迁移 OpenRC 到 SystemD"
date: 2023-10-05T12:26:03+08:00
tags:
    - Linux
    - Init System
    - Systemd
    - OpenRC
---

从 Arch Linux 迁移到 Gentoo 是因为当时 `小米笔记本 Air 13.3 2017` 使用 [`Systemd`](https://wiki.archlinux.org/title/Systemd) 作为 `init system` 时，会出现无法正常关机的情况，后续也切了几个发行版 （`Ubuntu` `Debian` `Void Linux` 等等），依旧存在某个设备持续在关闭，但迟迟关不上。当时怀疑的是驱动问题，但了解的太少，就跳过了排查阶段，直接上手了 `Gentoo`。

> 目前看来是 `SystemD` 会等待进程关闭，和桌面相关的，`SystemD` 会将一部分进程强制关联起来，这个操作让我很迷惑，比如 `Gnome` 使用 `Alacritty` 作为默认 `Terminal`，启动之后将会等待 `Alacritty` 关闭，而 `Alacritty` 并没有遵循指令关闭，直到 `SystemD` 强制关闭。
>
> `OpenRC` 在使用过程中没有遇到相关问题，可能其直接忽略了这个过程，没有细究。

`OpenRC` 只是一个 `init system`，想要实现一个完整的操作系统，依赖 `sys-auth/elogind` 和 `sys-fs/eudev` 实现登录控制和设备管理，完整流程和 `Systemd + udev` 几乎一致，在设计上，`OpenRC` 更坚持 `KISS` 原则。

非要找个理由替换到 `OpenRC` 的话，最直接的就是 `OpenRC` 只有 `Gentoo` 在维护，虽然 `Artix Linux` 等其他反 `Systemd` 的系统再用，生态上是比 `Systemd` 差很多。很多程序例如 [`Anbox`](https://github.com/anbox/anbox)，还有最近在测试的 [`玲珑`](https://linglong.dev/) （`Deepin` 的包管理工具），在源码里面硬依赖了 `SystemD` 或者是编译结果只附带了 `SystemD` 的启动方式，如果想适配 `OpenRC`，就得自己打补丁。

虽然 `Gentoo` 在适配 `OpenRC` 上做了很多努力，包仓库里面的程序基本打上了补丁以适配 `OpenRC`，且体验上基本无差异，但如果要使用一些强依赖的 `SystemD` 的软件，将其摘除就是件十分困难的事情。

随着 `SystemD` 入侵几乎所有主流版本，我几乎找不到继续对抗它的原因和动力，毕竟统一管理在我看来是一个比较合适的行为，工具太过分散使用还是不太舒服的。即使这使用 `Gentoo` 近五年时间里没有太多管理 `init system` 的需要，但遇到时，解决起来还是有点炸毛。

实行起来还是存在成本的，一是 `SystemD` 依赖爆炸，编译时间很长；二是完成后是否还存在之前的问题，又该怎么回退。

所以还是得打好快照做好备份，对步骤进行记录，本文会详细记录操作的过程。

主要参考以下文档：

1. [OpenRC](https://wiki.gentoo.org/wiki/OpenRC)
2. [Systemd(Gentoo)](https://wiki.gentoo.org/wiki/Systemd)
4. [Systemd(Arch Linux)](https://wiki.archlinux.org/title/Systemd)
3. [Hard dependencies on systemd](https://wiki.gentoo.org/wiki/Hard_dependencies_on_systemd)
5. [Snapper](https://wiki.gentoo.org/wiki/Snapper)

## 备份系统

一直在使用 `Btrfs` 作为磁盘格式，使用 `Snapper` 做定时快照，已被不时之需，此处直接使用 `Snapper` 创建使用前快照。

> 如果没有的 `Btrfs`，可以使用 [Timeshift](https://github.com/linuxmint/timeshift) 尝试用 `Rsync` 备份到外置磁盘。

```shell
snapper -c root create --description openrc-to-systemd-pre -c number
# or pre post
NUMBER=`snapper -c root create -t pre --description openrc-to-systemd-pre -c number`
# 更新后
snapper -c root create -t post --pre-number $NUMBER --description openrc-to-systemd-post -c number
```

完成快照后，可以安心变更 `USE` 了。

## Kernel 配置

[Systemd#Installation](https://wiki.gentoo.org/wiki/Systemd#Installation) 给出了详细的变更流程，此处按步骤操作。

### gentoo-source

可以使用 `sys-kernel/gentoo-sources` 直接选择 `systemd`，依赖选项比较多。其他源的话后续小节写了一个脚本。

```kernel
Gentoo Linux --->
   Support for init systems, system and service managers --->
      [*] systemd
```

### 其他 kernel 分支

根据[文档](https://github.com/systemd/systemd/blob/main/README)描述使用以下脚本，直接在 `/usr/src/linux` 执行即可。

> 实际上你只要为内核配置过 `docker` 和 `kvm` 支持，基本上只需要添加 `CONFIG_CMDLINE "init=/lib/systemd/systemd"` 即可。
>
> 在内核上 `OpenRC` 和 `SystemD` 依赖差别几乎不大。

```shell
# see https://github.com/systemd/systemd/blob/main/README
# /tmp/kernel.sh
# REQUIREMENTS
./scripts/config -e CONFIG_DEVTMPFS
./scripts/config -e CONFIG_CGROUPS
./scripts/config -e CONFIG_INOTIFY_USER
./scripts/config -e CONFIG_SIGNALFD
./scripts/config -e CONFIG_TIMERFD
./scripts/config -e CONFIG_EPOLL
./scripts/config -e CONFIG_UNIX
./scripts/config -e CONFIG_SYSFS
./scripts/config -e CONFIG_PROC_FS
./scripts/config -e CONFIG_FHANDLE

./scripts/config -d CONFIG_SYSFS_DEPRECATED

./scripts/config --set-val CONFIG_UEVENT_HELPER_PATH ""

./scripts/config -d CONFIG_FW_LOADER_USER_HELPER

# udev
./scripts/config -e CONFIG_DMIID
# SCSI
./scripts/config -e CONFIG_BLK_DEV_BSG
# Private Network
./scripts/config -e CONFIG_NET_NS
# Private User
./scripts/config -e CONFIG_USER_NS

# Optional but strongly recommended
./scripts/config -e CONFIG_IPV6
./scripts/config -e CONFIG_AUTOFS_FS
./scripts/config -e CONFIG_TMPFS_XATTR
./scripts/config -e CONFIG_{TMPFS,EXT4_FS,XFS,BTRFS_FS,F2FS_FS}_POSIX_ACL
./scripts/config -e CONFIG_SECCOMP
./scripts/config -e CONFIG_SECCOMP_FILTER
./scripts/config -e CONFIG_KCMP

# CPUShares
./scripts/config -e CONFIG_CGROUP_SCHED
./scripts/config -e CONFIG_FAIR_GROUP_SCHED

# CPUQuota
./scripts/config -e CONFIG_CFS_BANDWIDTH

# Required for IPAddressDeny=, IPAddressAllow=, IPIngressFilterPath=,IPEgressFilterPath= in resource control unit settings unit settings.
# Required for SocketBind{Allow|Deny}=, RestrictNetworkInterfaces= in resource control unit settings
./scripts/config -e CONFIG_BPF
./scripts/config -e CONFIG_BPF_SYSCALL
./scripts/config -e CONFIG_BPF_JIT
./scripts/config -e CONFIG_HAVE_EBPF_JIT
./scripts/config -e CONFIG_CGROUP_BPF

# For UEFI systems
./scripts/config -e CONFIG_EFIVAR_FS
./scripts/config -e CONFIG_EFI_PARTITION

# Required for signed Verity images support
# Required to verify signed Verity images using keys enrolled in the MoK
# (Machine-Owner Key) keyring
./scripts/config -m CONFIG_DM_VERITY
./scripts/config -e CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG
./scripts/config -e CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG_SECONDARY_KEYRING
./scripts/config -e CONFIG_IMA_ARCH_POLICY
./scripts/config -e CONFIG_INTEGRITY_MACHINE_KEYRING

# Required for RestrictFileSystems= in service units
./scripts/config -e CONFIG_BPF
./scripts/config -e CONFIG_BPF_SYSCALL
./scripts/config -e CONFIG_BPF_LSM
./scripts/config -e CONFIG_DEBUG_INFO_BTF
./scripts/config --set-val CONFIG_LSM "landlock,lockdown,yama,integrity,apparmor,bpf"

# We recommend to turn off Real-Time group scheduling in the kernel when using systemd. RT group scheduling effectively makes RT scheduling unavailable for most userspace, since it requires explicit assignment of RT budgets to each unit whose processes making use of RT. As there's no sensible way to assign these budgets automatically this cannot really be fixed, and it's best to disable group scheduling hence.
./scripts/config -d CONFIG_RT_GROUP_SCHED

# Required for systemd-oomd.
./scripts/config -e CONFIG_PSI
./scripts/config -d CONFIG_PSI_DEFAULT_DISABLED

# Note that kernel auditing is broken when used with systemd's container code. When using systemd in conjunction with containers, please make sure to either turn off auditing at runtime using the kernel command line option "audit=0", or turn it off at kernel compile time using.
./scripts/config -d CONFIG_AUDI

# 配置 systemd
# 注意未完成 systemd 配置，不要重启系统
./scripts/config -e CONFIG_CMDLINE_BOOL
./scripts/config --set-val CONFIG_CMDLINE "init=/lib/systemd/systemd"
```

完整配置可以参考 [gemone/kernel-config#tb-ara-systemd](https://gitee.com/gemone/kernel-config/tree/tb-ara-systemd)

### genkernel

检查以下配置，确保 `initramfs` 写入正确。

```conf
# /etc/initramfs.mounts
/usr
```

参考上一小节提供的 `shell`，写入 `kernel.sh`，随后执行，完整命令如下：

```shell
# /usr/src/linux
# 保留原有配置，只打 PATCH
chmod +x /tmp/kernel.sh
/tmp/kernel.sh

cp .config .config-gen
genkernel --kernel-config=.config-gen all
```

编译即可，暂时不需要重启。我们直接修改配置。

## 配置系统

备份 `OpenRC` 服务，用以后续重新启用：

```shell
rc-update -v show >> services
```

### merge-user

`SystemD` 今年(2023)开始已经不再提供 `unmerged-usr` 支持，所以需要开启 `merge-usr` 支持。

> 具体查看 [Merge-usr](https://wiki.gentoo.org/wiki/Merge-usr)。

```shell
emerge --ask sys-apps/merge-usr
merge-usr
```

### profile

需要清理 `elogind` 标记：

```shell
euse -R systemd
euse -D elogind
```

```shell
eselect profile list
# 目前迁移到了 使用 gnome
eselect profile set default/linux/amd64/17.1/desktop/gnome/systemd/merged-usr
emerge -avDN @world
```

如果出现问题的话，收集 `block` 重构：

```shell
# 收集写入 block 文件
# 后续查找已安装，调整之后再安装回去
cat /tmp/block | xargs -n 1 eix -I --world | grep "\[I\]" > block-select
# 记得调整下 block-select 里面的内容，保留包名即可
sed -i 's/\[.\]//g' block-select

# 然后 deselect 所有 block 的软件
cat /tmp/block | xargs emerge --deselect
# 重新安装
emerge -avDN @world
# 重新添加
cat block-select | xargs emerge --select=y --noreplace
```

**下面冲突是文档提供示例，收集 `block` 重构的过程已经处理，可以跳过**

清理 `openrc`：

```shell
euse -D netifrc -p sys-apps/openrc
emerge --oneshot sys-apps/openrc

emerge --deselect sys-apps/openrc sys-apps/sysvinit
emerge --deselect sys-fs/udev
emerge --oneshot virtual/udev virtual/libudev

# 一定要将 deselect 的依赖重新 select 一遍，不然会清理掉
emerge --ask --depclean
```

确认所有依赖完好，就可以重启了，接下来步骤需要使用 `systemd`。

## 升级到 SystemD

迁移之前我的笔记本安装了 `Gnome`，所以相关的依赖基本是完善的，这个过程并不是长，半分钟左右整个系统就重新编译好了。

后续就可以直接初始化 `SystemD` 了：

```shell
systemctl daemon-reexec

systemd-firstboot --prompt --setup-machine-id
systemctl preset-all
systemd-machine-id-setup
```

完成初始化后，重启。

升级完成后根据之前保存的 `services` 文件使用 `systemctl enable --now` 添加到开机启动。

清理一些不需要的软件，例如日志：

```shell
emerge --deselect app-admin/sysklogd
emerge --depclean
```

此时 `Gnome` 的亮度保存恢复正常了，之前使用 `OpenRC` 无法保存当前桌面的亮度，我都是使用脚本保存亮度处理的。应该是 `Gnome` 依赖了相关组件。

> `Gnome` 在定制上太复杂了，赶紧找时间切回 `wm` 上。

到此，`SystemD` 的切换就基本完成了。

## systemd-homed

**这只是一个尝试，提供的文档极有可能将 `home` 目录打乱！**

`homed` 支持加密 `home` 目录，还在输入密码是自动解密。

> 但很遗憾，这个功能只能在 `Luks` 磁盘上实现 ，我在 `btrfs` 上怎么尝试都不成功。最终驱动切换成 `btrfs`，使用子卷放置用户目录，毫无意义的切换。

```conf
# /etc/portage/package.use/systemd
sys-auth/pambase homed
sys-apps/systemd cryptsetup homed
```

```shell
emerge --ask --update --changed-use --deep @world
systemctl daemon-reexec
```

打好快照：

```shell
snapper -c root create --description root-homed-pre -c number
snapper -c home create --description home-homed-pre -c number
```

备份用户文件：

```shell
cp /etc/passwd /etc/passwd.bak
cp /etc/shadow /etc/shadow.bak
cp /etc/gshadow /etc/gshadow.bak
```

创建 `home` 目录：

**指定 uid 并不生效，系统会分配新的 uid，这会导致之前的大部分目录权限无法再使用！ **

```shell
U=xxx
cp --archive --recursive /home/$U /home/$U.saved
# group 可以使用 groups $U 查看
homectl create $U --uid=1000 --real-name="xx xx" --member-of=wheel,audio,docker,kvm,video,plugdev,portage,users,vboxusers,libvirt
```

> 如果存储空间不足，可以完全退出用户，使用 `mv /home/$U /home/$U.saved` 重命名目录。

### 问题

`homectl` 挂载选项有 `nosuid`，无法使用 `rootless` 的 `docker/podman`，需要将 `graphroot` 迁移到其他盘里。

## 小结

在 `Gentoo` 上修改 `init system` 并不困难，你可以从 `OpenRC` 迁移到 `SystemD`，如果你觉得不合适，也随时可以将 `SystemD` 迁移到 `OpenRC`，毕竟两者做的事情几乎相同，只是理念不一致。

大体的流程如下：

1. 配置 `Kernel`，添加 `init system` 必要的配置；
2. 切换 `profile`，处理冲突依赖；
3. 重构系统，启动 `init system`;

这个步骤适合迁向 `SystemD`，也适合迁向 `OpenRC`。

### 体验

对上层的软件并没有太大影响，体验上大体一致。

1. `Gnome` 的使用相对更稳定了，之前总是会有小概率的故障，而且还没法排除原因，目前没有相关影响；
2. `journalctl` 在默认的情况下使用很舒服，集中管理的暴力美学，一览无余；
3. 用户登录的自启动也可以使用 `systemctl --user` 管理 （管的真多，`pipewire` 终于不需要手动关闭了）;
4. 服务的状态获取也很容易，借助日志很容易排查原因，相较于 `OpenRC`，默认做这些事情会省心很多；（`OpenRC` 只会说 `crashed`）；

其它的都差不多，暂时就留在这个状态了，可以删除快照了。