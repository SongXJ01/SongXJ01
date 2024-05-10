---
title: BUG日志：宝塔CentOS安装Docker失败（2022.9.13）
date: 2022-09-14 10:55:01
copyright: true
tags: [BUG, CentOS, Docker]
categories:
- BUG日志
---



报错内容

```bash
Traceback (most recent call last):
  File "class/panelPlugin.py", line 2788, in a
    return p.exec_fun(get)
  File "class/pluginAuth.py", line 67, in exec_fun
    raise public.PanelError(res['msg'])
public.PanelError: 面板运行时发生错误: Traceback (most recent call last):
  File "/www/server/panel/plugin/docker/docker_main.py", line 57, in GetConList
    for con in self.__docker.containers.list(all=True):
AttributeError: 'NoneType' object has no attribute 'containers'
```
<!--more-->

---

## 解决方式
### 1. 卸载已经安装的 Docker

如果已经安装了未运行成功的 Docker，错误如下图所示，那么请将这个 Docker 卸载。

![错误页面](/SongXJ01/images/BUG日志_宝塔CentOS安装Docker失败/bug_docker_1.png)

### 2. 在 `/etc/docker` 路径下创建 daemon 配置文件 

![daemon 配置文件](/SongXJ01/images/BUG日志_宝塔CentOS安装Docker失败/bug_docker_2.png)

在 `daemon.json` 文件中提前配置好 Docker 的镜像源，即将下面这段代码粘贴到 `daemon.json` 文件中。 `daemon.conf` 文件此时保持空即可。
```bash
{"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]}
```

### 3. 重新安装 Docker
我安装的是宝塔软件商店中的 3.9.1 的版本。


---

## BUG原因分析
可能是因为宝塔提供的镜像源和CentOS的版本不匹配，因为CentOS基于Python2.7运行的，Docker 3.9.1 的运行环境好像是Python3，所以要更新一下镜像源。

<br/><br/><br/><br/>