---
title: link in docker
date: 2016-05-12 00:00:00
tags: tech
---

#  --link in docker

容器A通过--link选项使用容器B的某个服务，`docker-compose.yml` 配置如下：

    A:
      image: image_a
      links:
        - B:B
      ...
    
    B:
      image: image_b
      ...

当容器B被重启后，A的link不会被自动更新，要一并重启A才行：

    $ docker-compose restart B # A doesn't work
    $ docker-compose restart A # it works well
