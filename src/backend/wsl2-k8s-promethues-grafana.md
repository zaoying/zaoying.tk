# 在WSL2上部署标准k8s集群并使用Prometheus监控spring cloud服务

> 2019年Windows Build大会上，微软发布 WSL2，意味着开发者们终于可以在Windows上开发和运行原汁原味 Linux 应用。

本篇文章将教会大家如何在 WSL2 部署一个标准版的 k8s 集群以及使用 Prometheus 和 grafana 监控 spring cloud 服务。

和第一代 WSL不同，WSL2 是运行在 hyper-v 上的一个 VM，因此可以运行原汁原味的 Linux 发行版，例如 Ubuntu，同样对 Linux ABI 接口兼容也更好，支持更多 Linux 应用，例如 docker 。

经过将近一年时间的开发与优化，微软团队终于发布了正式版 WSL2，用户只需将操作系统升级到 2004 以后的 Windows 10 即可。