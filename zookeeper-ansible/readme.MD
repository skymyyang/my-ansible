## 使用ansible自动化部署zookeeper

### 角色定义

- base 定义修改sysctl参数以及ulimit限制
- chrony 时间同步机制
- openjdk 基于yum安装openjdk 11,可自行定义oracle jdk替换
- zookeeper 安装3个节点的zookeeper



### 变量

所有的变量都写在组变量中，all的文件当中

### HOSTS

hosts文件中定义了主机，可自行修改

### 入口

site文件为入口文件

```bash
ansible-playbook site.yml -i hosts
```



