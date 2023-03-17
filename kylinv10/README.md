# 麒麟v10 系统适配

## `No such device or address`

在使用麒麟v10 操作系统安装 k8s 的情况下，如果满足以下条件，就会遇到在有资源限制的条件下，容器启动提示 `No such device or address` 的问题:

- `docker` 使用 `systemd` 做 `Cgroup` 驱动
- `kubelet` 使用默认配置，通过 `systemd` 管理 `Cgroup`
- `docker` 使用二进制安装，没有安装到 `/usr/local/bin` 目录下

此问题的根本原因在于，在麒麟 v10 操作系统中，默认存在一个 `/usr/local/bin/runc` 。如果 `docker` 安装到了 `/usr/bin` 等目录下，受麒麟 v10 操作系统中内置的 `runc` 影响，就会导致这个问题。一般可以通过直接删除 `/usr/local/bin/runc` 解决