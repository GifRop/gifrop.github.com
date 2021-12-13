---
title: /dev/ptmx:no space left on device: unknown
date: 2021-12-13 22:29:18
tags: [docker]
---

在使用 `docker exec -it 容器id bash` 进入容器时，出现错误：

```shell
OCI runtime exec failed: exec failed: container_linux.go:370:starting container process caused:open /dev/ptmx: no space left on device: unknown
```

这个问题主要是由于伪终端数量限制了，在相应的机器上，执行下面命令即可：

```shell
sysctl -w kernel.pty.max=8192
```

