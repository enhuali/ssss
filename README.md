# Kernel 版本升级

## 为什么要升级?

  当前运行Kubernetes集群的宿主机kernel版本大多为3.10,此版本已知存在Cgroup泄漏问题;随着Pod数量销毁重建等频率增加,最终可能导致节点处于空闲且Ready状态,但无法接受调度(**具体错误内容呈现为mkdir /sys/fs/cgroup/memory/kubepods/besteffort/podc6eeec88-8664-11e9-9524-5254007057ba: no space left on device**);最坏的情况会引起雪崩效应导致集群瘫痪;

- `Cgroup泄漏`: memory cgroup 开启 kmem后accounting 中的 `memory.kmem.limit_in_bytes` 就可能会导致不能彻底删除 memcg 和对应的 cssid，也就是说应用即使已经删除了 cgroup (`/sys/fs/cgroup/memory` 下对应的 cgroup 目录已经删除), 但在内核中没有释放 cssid，导致内核认为的 cgroup 的数量实际数量不一致，我们也无法得知内核认为的 cgroup 数量是多少



## 为什么未出现过?

  Cgroup最大为65535,通常一个pod能泄漏两个memory cgroup数量配额,也就是说当pod销毁级别在3W+/node时会有可能触发内核bug;

  当集群内cronjob类型服务越多时(尤其频率高)+业务突增时必须重视此类问题;



## 升级方案

  使用ansible-playbook完成相关工作;

```
.
|____roles
| |____kernel_upgrade
| | |____tasks
| | | |____kernel_upgrade.yaml
| | | |____main.yaml
| | |____defaults
| | | |____main.yaml
| | |____files
|____playbooks
| |____kernel_upgrade.yml
|____README.md
|____.gitignore
|____inventory
| |____group_vars
| | |____all.yaml.example
| |____inventory.example
|____scripts
| |____init.sh
| |____kernel_upgrade.sh
| |____ansible.cfg
```

- Process:

  1. 节点内核版本判断——驱逐Pod——安装高版本内核——重启宿主机——验证内核——验证节点是否正常—— 将节点恢复调度
  2. 节点内核版本判断——高于预期版本,无需升级

  **过程中如遇异常或不符合预期均抛出pannic**

- Step:

  1. `inventory/inventory` 中配置hosts信息
  2. `inventory/group_vars/all.yaml` 中配置K8s二进制文件路径
  3. `roles/kernel_upgrade/defaults/main.yaml` 中包含内核下载地址
     - 配置需要升级的内核版本遵循x.y.z格式
     - 配置/boot磁盘阈值(为避免使用率过大而安装不完整)
  4. `roles/kernel_upgrade/files/`中下载对应kernel.rpm包,并以kernel-x.y.z.rpm命名
  5. `scripts/` 中执行kernel_upgrade.sh

- Topic:
  为确保K8s集群正常提供服务且防止意外情况对集群的影响,全流程采取串行工作(可根据实际情况调整并发数量);



## 大家需要做什么?

  以上方案理论已将可能影响范围降至最低,但仍有可能出现异常情况,为规避对于业务的影响,请大家进行如下自检:

- 请用户A定义`resources` 中`limit`、`request` 合理的资源,以避免驱逐Pod后无法申请到对应资源而导致Pending;(另外注意,如果用户B没有设置resources字段,如果集群存在资源问题,调度时会根据[服务质量Qos](https://blog.csdn.net/horsefoot/article/details/52091077)级别进行驱逐从而影响到用户B业务)
- 请用户定义`affinity` 字段,根据[Pod反亲和或者node亲和](https://blog.csdn.net/horsefoot/article/details/52091077)进行服务部署,以避免所有Pod均调度在同一Node上,在驱逐时导致业务中断
- 请用户定义`readinessProbe`、`livenessProbe` [探针](https://blog.csdn.net/qq_32641153/article/details/100614499),以避免驱逐Pod后流量丢失



## 特别注意

1. 内核升级是否会影响到现有内核优化参数;
2. 所有节点除用作k8s节点是否为其它所用;