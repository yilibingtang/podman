# immich-app podman + quadlet 部署

⚠️ **当前支持的 immich 版本：[v1.113.1]（https://github.com/immich-app/immich/releases/tag/v1.113.1）** ⚠️

这是一组通过 podman-quadlet systemd 生成器部署 immich 的单元文件

参见 [documentation]（https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html）
这是改编自 immich 提供的 docker-compose 文件，这将创建一个托管所有 thc 容器的 podman pod。

# 概述

此设置包括：
 - 'immich.pod' 文件定义将托管所有容器的 Pod。
 - 'immich-server.image' 定义 immich 版本。
 - 几个定义容器、卷等的 'immich-*.container' 文件。
 
“.container”文件被转换为创建容器的 systemd 单元。

注意 'immich-server.container' 在 'default.target' 上有一个安装目标，这使得它在启动时启动。

# 如何部署？

将 'env.example' 重命名为 'immich.env'。根据需要填充值。
## rootful podman

将这些文件复制到 '/etc/containers/systemd' 中，然后重新加载 systemd。

它可以是一个子目录。例如：
```
须藤 CP -R ./etc/containers/systemd/immich 中
sudo systemctl 守护进程-reload
```

## 无根 podman

创建将运行 immich containers 的用户。
容器将在主机上充当此用户：
- 在挂载的卷中强制执行文件权限时，将使用此用户。
- 当 immich 在挂载的卷中创建文件时，它们将归主机上的该用户所有。

```
useradd -r -m -d /var/lib/immich immich immich
```
容器和命名卷将存储在 '/var/lib/immich' 中。

为 podman 配置 'subuid' 和 'subgid' 以便能够以该用户身份运行：
```
cat >> /etc/subuid <<EOF immich：2000000：1000000 EOF cat >> /etc/subgid <<EOF
IMMICH：2000000：1000000
EOF
```

将这些文件复制到用户的 'containers/systemd' 目录中：
```
sudo -u immich mkdir -p ~immich/.config/containers/systemd/immich
sudo -u immich cp -v *.image *.container *.pod immich.env ~immich/.config/containers/systemd/immich/
```

启动用户会话，使其持久化并启动 pod：
```
systemctl start user@$（id -u immich）
loginctl enable-linger immich
systemctl --user -M immich@.host 启动 immich-pod.service
```

观察 'journalctl' 以查看容器是否成功启动 -
如果下载映像所需的时间超过默认启动超时时间，则首次启动可能会失败：
```
journalctl -f
```

容器应在下次启动时自动启动。

# 数据库备份

'database_backup' 文件夹建议了一种定期转储数据库的方法。确保将卷挂载添加到
数据库容器并将其绑定到 '/var/db_backup'

按原样，该单元将创建以创建日期“YYYYMMDD”命名的 gzip 压缩 SQL 转储。

要启用它，请将文件放在 '/etc/systemd/system' 中，然后启用计时器：'systemctl enable --now immich-database-backup.timer'。

# 待办事项
- 编写一个 makefile 或 justfile，将变量插入到单元文件中，也许？现在它需要一些复制和粘贴。
- 将其贡献给 immich ： 不再是一个目标，他们表示他们 [不感兴趣]（https://github.com/immich-app/immich/discussions/7977）。