## 基于阿里云的ECS安装docker-ce

yum仓库采用阿里云的镜像源。

配置文件已经优化

不包含系统的基础优化。

执行命令：

```bash
ansible-playbook site.yml -i hosts
```

