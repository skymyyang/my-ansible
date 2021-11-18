## 基于Ubuntu 20.04以及官方的gz包部署elasticsearch集群

1. 使用ansible自动化部署
2. 主要变量存放在groups_vars/all 文件中
3. hosts中也有自定义的变量
4. elasticsearch roles中 main.yml出的hosts 文件修改，也需要自己定义
5. elasticsearch没有开启跨域，需自行修改配置文件
6. 需要手动关闭防火墙
7. 时区也需要手动设置
8. 系统参数的调整在base role中





### 基于Ubuntu 20.04 的系统准备工作


| IP地址          | NodeName | 系统        | 配置 | 节点        |
| --------------- | -------- | ----------- | ---- | ----------- |
| 192.168.227.200 | node-e1  | ubuntu20.04 | 2C2G | master data |
| 192.168.227.201 | node-e2  | ubuntu20.04 | 2C2G | master data |
| 192.168.227.202 | node-e3  | ubuntu20.04 | 2C2G | master data |



gz的包名称为：elasticsearch-7.14.0-linux-x86_64.tar.gz，如有格式错误，可能会出错。

该目录下的包为空，需自行下载，然后覆盖。[下载地址](https://www.elastic.co/cn/downloads/elasticsearch#ga-release)

下载该目录，然后执行:

```ansible
ansible-playbook -i hosts site.yml
```

验证：

```json
curl http://192.168.227.201:9200/_cluster/health
{
"cluster_name": "myapp-es-cluster",
"status": "green",
"timed_out": false,
"number_of_nodes": 3,
"number_of_data_nodes": 3,
"active_primary_shards": 1,
"active_shards": 2,
"relocating_shards": 0,
"initializing_shards": 0,
"unassigned_shards": 0,
"delayed_unassigned_shards": 0,
"number_of_pending_tasks": 0,
"number_of_in_flight_fetch": 0,
"task_max_waiting_in_queue_millis": 0,
"active_shards_percent_as_number": 100
}
```

附赠：

## 基于centos7 的手动安装步骤

## 基于官方的gz包安装elasticsearch集群7.14.0


### 基于centos7 的系统准备工作


| IP地址          | NodeName | 系统    | 配置 | 节点        |
| --------------- | -------- | ------- | ---- | ----------- |
| 192.168.227.130 | node-c1  | centos7 | 2C2G | master data |
| 192.168.227.131 | node-c2  | centos7 | 2C2G | master data |
| 192.168.227.132 | node-c3  | centos7 | 2C2G | master data |



###### 创建用户 （所有节点均需执行）

```bash
$ useradd elasticsearch -s /sbin/nologin -M -U
#-U 创建一个组与此用户的名字相同 -M 不创建家目录
```
##### 操作系统的优化配置 (所有节点均需执行)
```
$ cat /etc/security/limits.conf

# End of file
root soft nofile 65535
root hard nofile 65535
* soft nofile 65535
* hard nofile 65535
elasticsearch soft memlock unlimited
elasticsearch hard memlock unlimited
```

```
$ cat /etc/sysctl.conf
#追加如下两行
vm.max_map_count=655360
fs.file-max=65536
```

PS: 这里修改完配置后，为了生效，建议重启服务器

##### 创建相关目录(所有节点均需执行)

```bash
$ mkdir -p /data/elasticsearch
$ mkdir -p /data/elasticsearch/logs -p
$ chown -R elasticsearch:elasticsearch /data/elasticsearch/
```

##### 下载安装包 (所有节点均需执行)

[下载地址](https://www.elastic.co/cn/downloads/elasticsearch#ga-release)

```bash
$ cd /usr/local/src/
$ wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.14.0-linux-x86_64.tar.gz
$ tar zxvf elasticsearch-7.14.0-linux-x86_64.tar.gz
mv elasticsearch-7.14.0 /usr/local/elasticsearch
chown elasticsearch:elasticsearch -R /usr/local/elasticsearch/
```

##### 配置主机名

```bash
$ cat /etc/hosts
192.168.227.130 node-c1
192.168.227.131 node-c2
192.168.227.132 node-c3
```

##### 修改配置文件(其他节点只需要修改`node.name`即可)

```bash
cluster.name: myapp-es-cluster
node.name: node-c1
node.master: true
node.data: true
path.data: /data/elasticsearch
path.logs: /data/elasticsearch/logs
network.host: 0.0.0.0
http.port: 9200
transport.tcp.port: 9300
transport.tcp.compress: true
discovery.seed_hosts: ["192.168.227.130:9300", "192.168.227.131:9300", "192.168.227.132:9300"]
cluster.initial_master_nodes: ["node-c1", "node-c2", "node-c3"]
gateway.recover_after_nodes: 3
```

- `cluster.name: myapp-es-cluster` 
  
    配置elasticsearch集群名称，默认是elasticsearch。这里定义为myapp-es-cluster，elasticsearch会自动发现在同一网段下的集群名为myapp-es-cluster的主机。如果在同一网段下有多个集群，可以通过此属性来区分。

- `node.name: node-c1`

    节点名。PS注意这里要与`cluster.initial_master_nodes`里作为master节点的名称保持一致。
    
- `node.master: true`

    指定该节点是否有资格被选举为master。 默认为true， elasticsearch集群中默认第一台启动的机器为master角色，如果这台服务器宕机就会重新选举master。此集群中其他的节点也设置为了master。所以也要把此处设置为true.
    
- `node.data: true`

    指定该节点是否存储索引数据，默认为true，表示数据存储节点，如果节点配置`node.master: false`，并且`node.data: false`，则该节点就是client node. 这个`client node`类似于一个路由器，负责将集群层面的请求转发到主节点，将数据相关的请求转发到数据节点。我们此集群中3个节点均为数据节点。
    
- `path.data: /data/elasticsearch`

    设置索引数据的存储路径，默认是elasticsearch根目录下的data文件夹，可以定义多个存储路径，用逗号隔开。
    
- `path.logs: /data/elasticsearch/logs`

    设置日志文件的存储路径，默认是elasticsearch根目录下的logs文件夹。
    
- `network.host: 0.0.0.0`

    此配置项是`network.publish_host`和`network.bind_host`两个配置项的集合，`network.bind_host`用来设置elasticsearch提供服务的ip地址，默认值为`0.0.0.0`，如果服务器有多块网卡，可以进行对应的设置。
    
    `network.publish_host`用来设置elasticsearch集群中该节点和其他节点交互通信的IP地址，一般为内网地址。
    
- `http.port: 9200`

    设置elasticsearch对外提供服务的http端口，默认为9200.
    
- `transport.tcp.port: 9300`

    此配置项用来设置节点间交互通信的TCP端口。 默认是9300
    
- `transport.tcp.compress: true`

    节点之间通信启用压缩。
    
- `discovery.seed_hosts: ["192.168.227.130:9300", "192.168.227.131:9300", "192.168.227.132:9300"]`

    集群中 node 节点发现列表
    
- `cluster.initial_master_nodes: ["node-c1", "node-c2", "node-c3"]`

    集群初始化那些节点可以被选举为 master,这里的填写不能直接下IP地址，要与node.name保持一致。
    
- `gateway.recover_after_nodes: 3`

    一个集群中的 N 个节点启动后,才允许进行数据恢复处理
    
- `action.destructive_requires_name: true`

    设置是否可以通过正则或者_all 删除或者关闭索引库，默认 true 表示必须需要显式指定索引库名称， 生产环境建议设置为 true，删除索引库的时候必须指定，否则可能会误删索引库中的索引库。
    
    

PS: 这里备注一下，之前版本无法生效的参数；添加后会启动报错。这里的版本是7.14.0

```
discovery.zn.minimum_master_nodes: 1
#配置当前集中最少的master节点数，默认为1.如果master节点低于此值，elasticsearch集群将停止运行。
bootstrap.mlockall：true
#设置为 True 时可锁住内存。因为当 JVM 开始 Swap 时，ES 的效率会降低，所以要保证它不 Swap。
```


##### 配置开机启动项

```
[root@c1 ~]# cat /usr/lib/systemd/system/elasticsearch.service 
[Unit]
Description=Elasticsearch
Documentation=http://www.elastic.co
Wants=network-online.target
After=network-online.target

[Service]
RuntimeDirectory=elasticsearch
PrivateTmp=true
Environment=ES_HOME=/usr/local/elasticsearch
Environment=ES_PATH_CONF=/usr/local/elasticsearch/config
Environment=PID_DIR=/usr/local/elasticsearch
EnvironmentFile=-/etc/sysconfig/elasticsearch

WorkingDirectory=/usr/local/elasticsearch

User=elasticsearch
Group=elasticsearch

ExecStart=/usr/local/elasticsearch/bin/elasticsearch -p ${PID_DIR}/elasticsearch.pid --quiet

# StandardOutput is configured to redirect to journalctl since
# some error messages may be logged in standard output before
# elasticsearch logging system is initialized. Elasticsearch
# stores its logs in /var/log/elasticsearch and does not use
# journalctl by default. If you also want to enable journalctl
# logging, you can simply remove the "quiet" option from ExecStart.
StandardOutput=journal
StandardError=inherit

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65535

# Specifies the maximum number of processes
LimitNPROC=4096

# Specifies the maximum size of virtual memory
LimitAS=infinity

# Specifies the maximum file size
LimitFSIZE=infinity

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=0

# SIGTERM signal is used to stop the Java process
KillSignal=SIGTERM

# Send the signal only to the JVM rather than its control group
KillMode=process

# Java process is never killed
SendSIGKILL=no

# When a JVM receives a SIGTERM signal it exits with code 143
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

##### 开机启动

```bash
$ systemctl daemon-reload
$ systemctl start elasticsearch
$ systemctl enable elasticsearch
```