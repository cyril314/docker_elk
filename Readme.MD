# 在Docker中快速搭建ELK环境

  本项目基于ES原版镜像转阿里云加速镜像实现的搭建过程，但如果你需要的版本在[elastic_booster](https://github.com/cyril314/elastic_booster)
中已经生成加速容器，不然不能提供加速服务。

注：现提供的加速版本
1. 8.0.0
2. 8.7.0
3. 8.8.1
4. 8.11.2
5. 8.17.0

## 使用方式
        # 将本项目放入搭建好docker环境的服务器中后执行如下操作
        $ docker-compose up setup --rm
        $ docker-compose up -d

        # 项目没有将ES的数据目录设置到指定位置，所以使用的是默认容器存储
        $ cd /var/lib/docker/volumes && ls

        # 删除存储
        $ docker volume ls -q |grep elastic |xargs docker volume rm

        # 安装其他控件
        $ docker compose -f docker-compose.yml -f extensions/fleet/fleet-compose.yml up -d
