# hai-platform notes

<!-- TOC -->
* [hai-platform notes](#hai-platform-notes)
  * [本地部署实验环境说明](#本地部署实验环境说明)
  * [集群搭建](#集群搭建)
  * [部署命令](#部署命令)
  * [hai-up.sh中一些配置的修改](#hai-upsh中一些配置的修改)
  * [hai-cli提交任务](#hai-cli提交任务)
    * [一些有用的配置](#一些有用的配置)
    * [使用hai-cli](#使用hai-cli)
  * [hai-platform代码](#hai-platform代码)
  * [hai-cli使用报错问题排查](#hai-cli使用报错问题排查)
<!-- TOC -->

## 本地部署实验环境说明

1. 为什么要在本地部署
   - 阿里云太贵了
   - 无论是hai后续在裸金属上部署，还是后续perfxcloud部署，都是本地环境部署，都需要自己搭建集群，早晚都要走这条路
2. 本地部署环境
   - master节点：16核CPU，16G内存，512G硬盘，3070-8G显卡
   - node节点：12核CPU，32G内存，512G硬盘，3060-12G显卡
   
## 集群搭建

1. 使用kubeasz搭建双节点集群
    - 如果之前装过集群，需要先清理集群，ref：https://blog.51cto.com/u_12482901/5735949
    - ref: https://github.com/easzlab/kubeasz/blob/master/docs/setup/00-planning_and_overall_intro.md
    - 直接规划两个节点，一个master，一个worker
    - etcd配置一个，复用master节点
    - master：10.10.10.11
    - worker：10.10.10.21
2. 部署loadbalancer：MetalLB
    - ref: https://www.lixueduan.com/posts/cloudnative/01-metallb/
    - 节点池的配置需要理解MetalLB的工作原理，不能与集群节点IP重复，不然会有冲突（具体GPT）
    - 节点池配置为192.168.1.200-192.168.1.210
    - cmd
      ```bash
      kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
      kubectl apply -f IPAddressPool.yaml -f L2Advertisement.yaml
      ```
3. 部署ingress controller：ingress-nginx
    - ref: https://www.cnblogs.com/linuxk/p/18105831
    - ref: https://www.cnblogs.com/syushin/p/15271304.html
    - 使用helm安装，注意修改配置文件，不要直接安装，k8s域名下的镜像拉取不到
    - 下载链接: https://github.com/kubernetes/ingress-nginx
    - cmd
      ```bash
      kubectl create ns ingress-nginx
      helm install ingress-nginx -n ingress-nginx .
      ```
4. 在所有节点上配置nfs，过程略，**注意客户端要设置成每次开机自动挂载**
5. 配置ingress_host：获取metallb为ingress提供的外部IP，在/etc/hosts文件中绑定自定义域名
   - `192.168.1.200 hai.local`

## 部署命令

1. ref: https://hfailab.github.io/hai-platform/start/install.html
2. `bash hai-up.sh dryrun --provider k8s`，生成部署文件，存放在nfs目录下
3. `bash hai-up.sh up --provider k8s`，应用这些部署文件

## hai-up.sh中一些配置的修改

1. `: ${INGRESS_HOST:="hai.local"}`
   - 在阿里云等共有云上，这个域名配置的就是集群管理节点的域名（注意，不是ip）
   - 在裸金属上部署，没有公有云提供的负载均衡服务，所以就需要先在自己的集群中配置负载均衡服务，然后将其指定为汲取内部配置的负载均衡服务域名
   - 这也是在集群搭建时需要部署MetalLB和ingress-nginx的原因
2. `: ${TRAINING_NODES:="10.10.10.21"}` | `: ${JUPYTER_NODES:="10.10.10.11"}` | `: ${MANAGER_NODES:="10.10.10.11"}`
   - 这三个配置项，指定的不是ip，而是node的名称
   
   - 如下是我的集群节点
   ```bash
   (base) ➜  ~ k get node                           
   NAME     STATUS                     ROLES    AGE   VERSION
   master   Ready,SchedulingDisabled   master   23h   v1.29.0
   worker   Ready                      node     23h   v1.29.0
   ```
   
   - 按上述配置后，平台pod报错无法创建，log如下
   ```bash
   (base) ➜  ~ k logs hai-platform-0 -n hai-platform        
   ==== label nodes ====
   kubectl label nodes 10.10.10.21 hai_mars_group=training
   Error from server (NotFound): nodes "10.10.10.21" not found 
   ```
   
   - 解决方案
     1. `bash hai-up.sh down`
     2. 修改配置文件
     3. `bash hai-up.sh dryrun --provider k8s`
     4. `bash hai-up.sh up --provider k8s`
     
   - 效果
    ```bash
    (base) ➜  one k logs hai-platform-0 -n hai-platform        
    ==== label nodes ====
    kubectl label nodes worker hai_mars_group=training
    node/worker labeled
    kubectl label nodes master hai_mars_group=jupyter_cpu
    node/master labeled
    ==== start redis ====
    Starting redis-server: redis-server.
    ==== start postgres ====
    /tmp /high-flyer/code/multi_gpu_runner_server
    The files belonging to this database system will be owned by user "postgres".
    This user must also own the server process.
    
    The database cluster will be initialized with locale "C".
    The default text search configuration will be set to "english".
    
    Data page checksums are disabled.
    
    fixing permissions on existing directory /var/lib/postgresql/12/main ... ok
    creating subdirectories ... ok
    selecting dynamic shared memory implementation ... posix
    selecting default max_connections ... 100
    selecting default shared_buffers ... 128MB
    selecting default time zone ... Asia/Shanghai
    creating configuration files ... ok
    running bootstrap script ... ok
    performing post-bootstrap initialization ... ok
    initdb: warning: enabling "trust" authentication for local connections
    You can change this by editing pg_hba.conf or using the option -A, or
    syncing data to disk ... ok
    
    
    Success. You can now start the database server using:
    
        /usr/lib/postgresql/12/bin/pg_ctl -D /var/lib/postgresql/12/main -l logfile start
    
    --auth-local and --auth-host, the next time you run initdb.
    /high-flyer/code/multi_gpu_runner_server
    * Starting PostgreSQL 12 database server
      ...done.
      /tmp /high-flyer/code/multi_gpu_runner_server
      ALTER ROLE
      CREATE DATABASE
      /high-flyer/code/multi_gpu_runner_server
      合并所有sql到一个文件中：/tmp/fuse_sql.sql
      开始初始化，注意，若存在 task_ng 和 user 这两个表，将不会进一步初始化
      psql:/tmp/fuse_sql.sql:220: NOTICE:  moving and merging column "queue_status" with inherited definition
      DETAIL:  User-specified column moved to the position of the inherited column.
    ```

## hai-cli提交任务

### 一些有用的配置

1. `/nfsroot/hai-platform/override.toml`：存放了平台账号密码配置等信息
2. `/nfsroot/hai-platform/k8s_configs`：平台部署文件

### 使用hai-cli

1. refs
   1. https://github.com/HFAiLab/hai-platform
   2. https://hfailab.github.io/hai-platform/cli/user.html
   3. https://github.com/HFAiLab/hai-platform/issues/5
2. 初始化
    ```bash
    HAI_SERVER_ADDR=`kubectl -n hai-platform get svc hai-platform-svc -o jsonpath='{.status.loadBalancer.ingress[0].ip}'`

    # TOKEN 为 USER_INFO 设置的token，登录pgsql，查看user表，里面存了token
    hai-cli init ${TOKEN} --url http://${HAI_SERVER_ADDR}
    
    # python文件默认需放在用户工作目录 ${SHARED_FS_ROOT}/hai-platform/workspace/{user.user_name}
    # 如置于其他路径，需要在pg数据库storage表中添加相应挂载项
    hai-cli python ${SHARED_FS_ROOT}/hai-platform/workspace/$(whoami)/test.py -- -n 1
    ```
3. 本地初始化命令
   - `hai-cli init haiadmin --url 192.168.1.201`
   - `hai-cli init ACCESS-68516961646d696e2368616961646d696e-E0lGXwIswnn0HpbXAW_tVRjga1wRjD0u --url http://192.168.1.201`
   
## hai-platform代码

1. 使用python3.8版本，高版本python在requirements.txt中部分库不兼容
2. `supervisord.conf`
   - 这个配置文件是 `supervisord` 的配置文件，用于管理和监控 Linux 系统中的进程。`supervisord` 是一个用 Python 写的系统工具，能够管理和监控进程，确保指定的服务在服务器启动时自动运行，并在意外崩溃时自动重启。
   - 在这个配置文件中，定义了多个 `program` 部分，每个部分对应一个需要 `supervisord` 管理的进程。每个 `program` 部分都包含了一些配置项，如 `command`（启动进程的命令）、`directory`（进程的工作目录）、`autostart`（是否在 `supervisord` 启动时自动启动该进程）、`autorestart`（在进程意外退出时是否自动重启）等。 
   - 在你的项目中，这个配置文件被用于启动和管理如 `k8swatcher`、`launcher`、`scheduler`、`query_server`、`operating_server`、`ugc_server`、`monitor_server`、`haproxy` 和 `studio` 等进程。 
   - 在 `entrypoint.sh` 脚本中，如果 `MANUAL_START_SEVER` 环境变量不等于 "1"，则会使用 `supervisord -c one/supervisord.conf` 命令启动 `supervisord`，并使用这个配置文件来管理进程。

## hai-cli使用报错问题排查

1. 通过kubectl logs hai-platform...可以查看到初始化日志
    ```log
    2024-04-09 15:01:51,629 INFO exited: scheduler (exit status 1; not expected)
    2024-04-09 15:01:51,638 INFO spawned: 'scheduler' with pid 387
    2024-04-09 15:01:51,640 INFO exited: studio (exit status 1; not expected)
    2024-04-09 15:01:51,666 INFO spawned: 'studio' with pid 392
    2024-04-09 15:01:52,641 INFO success: scheduler entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2024-04-09 15:01:52,647 INFO exited: studio (exit status 1; not expected)
    2024-04-09 15:01:52,816 INFO exited: scheduler (exit status 1; not expected)
    ```
   - scheduler反复重启
   - 查看one/supervisord.conf:64
2. 查看log文件：/nfsroot/hai-platform/operating_0.log
    ```log
    ...
    sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) connection to server at "127.0.0.1", port 15432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
    ...
    ```
3. 查看svc
    ```bash
    (hai) ➜  one git:(main) ✗ k get svc -n hai-platform
    NAME               TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)                                                     AGE
    hai-platform-svc   LoadBalancer   10.68.30.2   192.168.1.201   5432:31171/TCP,6379:32370/TCP,80:30851/TCP,8080:31172/TCP   42m
    ``` 
4. 问题定位
   1. 基于上述log，猜测是因为端口号不对，导致连接失败，在代码中全局搜索15432，发现是在`one/one_etc/core.toml`文件中
   2. 进一步定位，发现该配置文件在`one/entrypoint.sh`中被使用，这个脚本把这个配置文件拷贝到了系统资源目录下
   3. 进一步定位，发现`one/supervisord.conf`配置文件中加载了该目录下的配置，用于创建守护进程`[program:scheduler]`
   4. 查看平台k8s部署文件，`entrypoint.sh`用于创建pod时执行初始化操作
   ```yaml
    containers:
    - command:
      - bash
      - -c
      - cd /high-flyer/code/multi_gpu_runner_server && bash one/entrypoint.sh
   
   ```
   5. `kubectl exec -it hai-platform-0 -n hai-platform -- bash`进入pod查看配置文件，发现`core.toml`文件中的端口号是`15432`、`16379`
   6. **至此，猜测大概率是端口问题**
4. 问题解决
   1. 重新构建镜像
      1. 参考readme
      ```txt
      # replace IMAGE_REPO with your own repo
      $ IMAGE_REPO=registry.cn-hangzhou.aliyuncs.com/hfai/hai-platform bash one/release.sh
      build hai success:
      hai-platform image: registry.cn-hangzhou.aliyuncs.com/hfai/hai-platform:fa07f13
      hai-cli whl:
      /home/hai-platform/build/hai-1.0.0+fa07f13-py3-none-any.whl
      /home/hai-platform/build/haienv-1.4.1+fa07f13-py3-none-any.whl
      /home/hai-platform/build/haiworkspace-1.0.0+fa07f13-py3-none-any.whl
      ```
      2. 修改Dockerfile，解决github无法连接问题，过程略，参考git log
      3. `IMAGE_REPO=10.10.10.11:1180/hfai/hai-platform bash one/release.sh`
   2. 本地搭建harbor，管理镜像
      1. 搭建过程略
      2. 使用http，禁用https，containerd运行时需要修改配置文件，集群所有节点都要改，参考：https://blog.csdn.net/lhf2112/article/details/117195731
   3. 修改hai-up.sh中的镜像地址为：`: ${BASE_IMAGE:="10.10.10.11:1180/hfai/hai-platform:c59aa45"}`，重新部署
      - `cd /root`
      - `python -m http.server 8000`
   4. 效果
   ```bash
   (hai) ➜  one git:(main) ✗ hai-cli python /nfsroot/hai-platform/workspace/haiadmin/test.py -- -n 1
    WARNING:  提交的任务将会继承当前环境 ，有可能造成环境不兼容，如不想继承当前环境请添加参数 --no_inherit 
   提交任务成功，定义如下
   --------------------------------------------------------------------------------
   name: test.py
   priority: 30
   resource:
     group: default
     image: default
     node_count: 1
   spec:
     entrypoint: test.py
     parameters: ''
     workspace: /nfsroot/hai-platform/workspace/haiadmin
   version: 2
   
   --------------------------------------------------------------------------------
   ==================== experiment ====================
   +----+---------+-------+--------------+-----------+---------------+----------------------------+
   | id | nb_name | nodes | chain_status | task_type | suspend_count | created_at                 |
   +====+=========+=======+==============+===========+===============+============================+
   | 1  | test.py | 1     |              | training  | 0             | 2024-04-11 16:19:35.356763 |
   +----+---------+-------+--------------+-----------+---------------+----------------------------+
   任务创建完成，请等待调度，可以使用以下接口查询
      hai-cli status test.py  # 查看任务状态
      hai-cli logs -f test.py # 查看任务日志
      hai-cli stop test.py # 关闭任务
   ```


