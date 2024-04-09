# hai-platform notes

## Install

1. [reference](https://hfailab.github.io/hai-platform/start/install.html)
2. 运行命令
    ```
    node1@node1:~/nfs_data/hai-platform/one$ sudo bash hai-up.sh dryrun --provider k8s
    [sudo] password for node1:
    DRYRUN.
    Using provider: k8s
    ################################
    #   WELCOME TO HAI PLATFORM!   # 
    ################################
    STEP: precheck
    STEP: parse user/host info
    training nodes: cn-hangzhou.172.23.183.227
    jupyter nodes: cn-hangzhou.172.23.183.226
    EXTRA_K8S_VOLUMEMOUNTS:
    
    EXTRA_K8S_HOSTPATH:
    
    EXTRA_K8S_ENVIRONMENTS:
    
    STEP: generate config
    generated rendered config in /nfs-shared/hai-platform: init.sql override.toml k8s_configs/hai-platform-deploy.yaml k8s_configs/hai-platform-service.yaml k8s_configs/hai-platform-ingress-studio.yaml
    ```

是的，您使用的是terway-eni的方式 节点pod最大数量=（ECS支持的弹性网卡数-1）×单个ENI支持的私有IP数
另外就是您的cpu也需要扩容，资源不足了

## 准备工作

1. 使用kubeasz部署双节点集群
    - ref: https://github.com/easzlab/kubeasz/blob/master/docs/setup/00-planning_and_overall_intro.md
    - 直接规划两个节点，一个master，一个worker
2. 部署loadbalancer：MetalLB
    - ref: https://www.lixueduan.com/posts/cloudnative/01-metallb/
    - 节点池的配置需要理解MetalLB的工作原理，不能与集群节点IP重复，不然会有冲突（GPT-4）
3. 部署ingress controller：ingress-nginx
    - ref: https://www.cnblogs.com/linuxk/p/18105831
    - ref: https://www.cnblogs.com/syushin/p/15271304.html
    - 使用helm安装，注意修改配置文件，不要直接安装，k8s域名下的镜像拉取不到
    - 下载链接: https://github.com/kubernetes/ingress-nginx
4. 在所有节点上配置nfs
5. 配置ingress_host：获取metallb为ingress提供的外部IP，在/etc/hosts文件中绑定自定义域名

## 部署命令

