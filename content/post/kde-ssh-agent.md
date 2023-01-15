---
title: "KDE SSH Agent 配置"
date: 2023-01-15T15:40:13+08:00
description: "与 Gnome 不一样， KDE 配置 SSH Agent 需要一些额外的步骤。"
keyswords:
    - KDE
    - DE
    - ssh-agent
tags:
    - IT
    - SSH
    - DE
---

在 `Gnome` 及其他 `GTK` 环境中使用 `gnome-keyring` 进行密钥管理时，`gnome-keyring` 默认会启动 `ssh-agent`。

> `Gentoo` 中启用 `ssh-agent` 标志。

但 `KDE` 使用 `KDE Wallet` 并不会这么方便，需要自行配置。

1. 配置 `~/.ssh/config` 开启 `AddKeysToAgent yes`；
2. 配置 `ssh-agent` 自启动；（`systemd` 可以配置服务）；
3. 配置 `SSH_ASKPASS` 为 `ksshaskpass`；

具体步骤如下：

## 配置 `ssh-agent`

需要配置 `ssh-agent` 启停。

> 此处仅考虑在桌面环境下使用 `ssh-agent` 进行管理，因此使用 `plasma-workspace` 进行配置即可。

```shell
#!/usr/bin/env bash

# ~/.config/plasma-workspace/env/ssh-agent.sh

[ -z "$SSH_AGENT_PID" ] && eval "$(ssh-agent -a /tmp/$USER.agent -s -t 300)"
export SSH_ASKPASS='/usr/bin/ksshaskpass'
export SSH_ASKPASS_REQUIRE=prefer
```

其中 `ssh-agent` `-t` 参数用以配置密钥有效时间，此
处配置为 `5m`，若未配置，则为永久有效。

而 `SSH_ASKPASS_REQUIRE` 用于唤起 `ksshaskpass` 的机制，未设置的话会导致仅使用 `tty` 密钥输入。 

记得退出桌面时关闭 `ssh-agent`，避免登出后造成配置不生效。

```shell
#!/usr/bin/env bash

# ~/.config/plasma-workspace/shutdown/kill-ssh-agent.sh

[[ -n "$SSH_AUTH_SOCK" ]] && eval `ssh-agent -k`
```

到此，`ssh-agent` 正常启用。

## 配置 `ssh/config`

`ssh-agent` 虽然启用了，但 `ksshaskpass` 并不会自动调用 `ssh-agent` 保存认证信息。

因此需要使用 `ssh` 本身配置，自动添加密钥到 `ssh-agent` 执行。

需要在 `~/.ssh/config` 添加配置，（或全局配置 `/etc/ssh/ssh_config`）:

```ini
# auto run ssh-add
AddKeysToAgent yes
```

到此，重新登录会话，`ssh-agent` 功能应该正常得到发挥。

## 注意事项

在 `ssh-agent` 启动脚本中记得添加 `-t <second>`，否则密钥会在当前桌面永久生效。（除非必要，避免让密码认证永久生效。）

与 `Gnome Keyring` 不同， `KDE Wallet` 不会记录 `SSH Keys` 的信息，全权交给 `ssh-agent` 自身处理，可以使用 `ssh-add -l` 查看已添加的密钥。
