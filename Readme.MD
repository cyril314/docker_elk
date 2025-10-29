# 在Docker中快速搭建ELK环境

  本项目基于ES原版镜像转阿里云加速镜像实现的搭建过程，但如果你需要的版本在[elastic_booster](https://github.com/cyril314/elastic_booster)
中已经生成加速容器，不然不能提供加速服务。
情报功能在ES8.8.0开始有的。

注：现提供的加速版本
1. 8.0.0
2. 8.7.0
3. 8.8.1
4. 8.11.2
5. 8.17.0

## 使用方式
```
# 开通端口
$ firewall-cmd --zone=public --add-port=5601/tcp --add-port=9200/tcp --add-port=9300/tcp --permanent && firewall-cmd --reload

# 开启代理
$ export HTTP_PROXY=http://192.168.30.41:1081 && export HTTPS_PROXY=http://192.168.30.41:1081
$ unset HTTP_PROXY HTTPS_PROXY

# 将本项目放入搭建好docker环境的服务器中后执行如下操作
$ docker-compose up setup
$ docker-compose up -d

# 项目没有将ES的数据目录设置到指定位置，所以使用的是默认容器存储
$ cd /var/lib/docker/volumes && ls

# 删除存储
$ docker volume ls -q |grep elastic |xargs docker volume rm

# 安装其他控件
$ docker-compose -f docker-compose.yml -f extensions/fleet/fleet-compose.yml up -d

# 访问不到security可能需要代理支持
$ vi /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:20171"
Environment="HTTPS_PROXY=http://127.0.0.1:20171"
Environment="NO_PROXY=localhost,127.0.0.1"

$ systemctl daemon-reload && systemctl restart docker

```
## docker配置
```
创建文件：sudo vi /etc/docker/daemon.json
添加内容：如果版本低于 18.xx需要使用'graph'代替
{
  "data-root": "/home/data/docker",
  "registry-mirrors": [
    "https://hub-mirror.c.163.com",
    "https://dockerproxy.net",
    "https://i9h29342.mirror.aliyuncs.com",
    "https://ee14a5b07b11413ba6d4af63f0eaea5b.mirror.swr.myhuaweicloud.com",
    "http://hammal.staronearth.win",
    "http://hub.staronearth.win",
    "https://docker.xuanyuan.me"
  ],
  "dns": ["119.29.29.29", "114.114.114.114"]
}
如果需要移动镜像存放位置需要执行: mkdir -p /home/data && mv /var/lib/docker /home/data/docker
```

## 解决证书过期登录不了

```
# 设置成基础证书
curl -XPOST "http://localhost:9200/_license/start_basic?acknowledge=true" -H "Authorization: Basic $(echo -n elastic:changeme | base64)"
curl -XPOST "http://localhost:9200/_xpack/license/start_trial?acknowledge=true" -H "Authorization: Basic $(echo -n elastic:changeme | base64)"
# 查看许可证信息
curl -X GET "localhost:9200/_license?pretty" -H "Authorization: Basic $(echo -n elastic:changeme | base64)"
# 清理缓存
curl -X POST "localhost:9200/_cache/clear?pretty" -H "Authorization: Basic $(echo -n elastic:changeme | base64)"
# 强制合并段文件，减少内存占用
curl -X POST "localhost:9200/_forcemerge?only_expunge_deletes=true&max_num_segments=1&pretty"
# 刷新所有索引
curl -X POST "localhost:9200/_flush?pretty"  -H "Authorization: Basic $(echo -n elastic:changeme | base64)"

# 添加fleet-agent时如果遇到证书问题添加不上可以使用参数临时跳过验证，禁用SSL/TLS验证参数' --insecure'，强制安装参数' --force'
如果时windows上添加，需要在管理员的PowerShell中执行添加服务命令，例如
$ .\elastic-agent.exe install --url=http://192.168.31.84:8220 --enrollment-token=xxxx --insecure --force
以管理员权限运行服务,先停止服务，修改执行权限后再启动
$ sc.exe config "Elastic Agent" obj= "LocalSystem"
添加hosts配置，用来确保局域网中能正常访问到fleet-server服务,以管理员身份运行 PowerShell，然后执行：
$ Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "`r192.168.31.84`t`tfleet-server" -Force
```

## ES配置说明

```
thread_pool.write.queue_size: 10000  # 默认1000，增加队列容量
thread_pool.write.size: 16           # 默认CPU核心数，增加处理线程
﻿
# 调整批量队列设置
thread_pool.bulk.queue_size: 20000   # 批量操作队列
thread_pool.bulk.size: 12            # 批量处理线程数
﻿
# 增加熔断器限制，防止内存不足错误
indices.breaker.total.limit: 70%     # 默认70%，可适当提高
indices.breaker.fielddata.limit: 40% # 字段数据熔断器
﻿
# 增加索引缓冲区大小
indices.memory.index_buffer_size: 20%  # 默认10%
```

## 设置情报配置

1. 首先配置高级管理中的威胁索引和'Elasticsearch 索引'添加',tjym_*'
2. 在数据视图中创建视图
3. 导入数据会创建tjym_ecs索引

## 设置enterprise-search只读模式
```
# 设置只读
PUT /_cluster/settings
{
    "persistent": {
      "cluster.blocks.read_only": false,
      "cluster.blocks.read_only_allow_delete": false
    },
    "transient": {
      "cluster.blocks.read_only": false,
      "cluster.blocks.read_only_allow_delete": false
    }
}
# 查看状态
GET _cluster/settings?pretty
```