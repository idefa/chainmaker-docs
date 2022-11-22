# 启动支持旧版本Docker_VM的链
作者：长安链团队 李腾辉、卢光宏、魏宁州

## 通过管理台启动启用Docker虚拟机的链

<a id='3.1.1'></a>

### 登录
<img loading="lazy" src="../images/ManagementLogin.png" style="zoom:50%;" />

- 私有化部署本平台时会生成的对应的admin账号，默认密码为a123456

<a id='3.1.2'></a>

#### 新增组织证书
<img loading="lazy" src="../images/ManagementAddOrgCert.png" style="zoom:50%;" />

- 填写组织ID和组织名称
- 组织和和组织名称不能重复
- 支持申请国密和非国密两种证书。


#### 新增节点证书
<img loading="lazy" src="../images/ManagementAddNodeCert.png" style="zoom:50%;" />

- 目前节点证书角色分为共识节点和同步节点两种。
- 通过填写节点名称、组织信息，节点角色，申请节点证书。
- 支持申请国密和非国密两种证书。


#### 新增用户证书
<img loading="lazy" src="../images/ManagementAddUserCert.png" style="zoom:50%;" />

- 目前用户证书角色分为admin、client和light三种。
- 通过填写用户名称、组织信息，用户角色申请用户证书。
- 支持申请国密和非国密两种证书。
- 合约部署需要对应的管理员证书，所以需要申请对应的管理员用户

#### 新建区块链
<img loading="lazy" src="../images/ManagementAddChain2.png" style="zoom:50%;" />
<img loading="lazy" src="../images/ManagementAddChain3.png" style="zoom:50%;" />


- 选择证书模式
- 链配置文件参数设定
  - 此处用于新增链配置文件，目前支持自定义链的id、名称、区块最大容量，出块间隔、交易过期时长，以及共识配置。
  - 目前支持配置TBFT、RAFT、SOLO、MAXBFT共识。
  - 申请链配置文件前，请先确保，所需的组织和节点证书已经申请/导入本管理平台。
  - 此处需要勾选Docker_VM
  - 支持单机部署和多机部署，请正确填写所要之后要部署区块链节点的机器所在的ip，并确保端口不冲突。

#### 下载部署链
<img loading="lazy" src="../images/ManagementDownloadChain2.png" style="zoom:50%;" />

- 如图所示需要先下载安装Docker_VM的环境依赖，然后再部署区块链。
- 部署区块链
  - 下载链配置以zip包为准，zip包包含对应的链配置文件和部署脚本
  - 将下载的包移动的需要部署的机器上去（可以使用scp进行移动）
  - 执行`unzip`解压成`release`包，进入`release`包执行`start.sh`进行启动

#### 快速订阅链
<img loading="lazy" src="../images/ManagementSubscribe.png" style="zoom:50%;" />

- 链部署成功之后在管理台进行快速订阅

#### 部署/调用合约验证链是否正常
订阅成功后，可进行部署/调用示例合约，以检查链功能是否正常。部署合约的使用教程可详见：[部署示例合约](./部署示例合约.md)。



## 通过命令行启动启用Docker虚拟机的链

### 环境依赖

**操作系统**

DockerVM实现依赖于cgroup，目前仅支持在Linux系统下部署和运行DockerVM。

**软件依赖**

docker，7zip

依赖软件下载：

- docker：请参看[https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)
- 7zip：请参看[7zip官网](https://sparanoid.com/lab/7z/)

拉取官方Docker虚拟机景象：

```shell
docker pull chainmakerofficial/chainmaker-vm-engine:v2.3.0
```

### 生成证书并初始化配置

**启用DockerVM**

在chainmaker中启用Docker VM有两种方式。

**方式一:**

 [通过命令行工具启动链](../tutorial/通过命令行工具启动链.md)，在执行`prepare.sh`、`prepare_pk.sh`、`prepare_pwk.sh`时，`enable docker vm` 选择 YES

```shell
enable docker vm (YES|NO(default))
```

- 进入chainmaker-go/scripts目录，执行prepare.sh脚本生成单链4节点集群配置，存于路径chainmaker-go/build中

- prepare_pk.sh脚本支持生成4/7/10/13/16节点公私钥和配置

```shell
# 进入脚本目录
$ cd ../scripts
# 查看脚本帮助
$ ./prepare.sh -h
Usage:
    prepare.sh node_cnt(1/4/7/10/13/16) chain_cnt(1-4) p2p_port(default:11301) rpc_port(default:12301) vm_go_runtime_port(default:32351) vm_go_engine_port(default:22351)
    eg1: prepare.sh 4 1
    eg2: prepare.sh 4 1 11301 12301
    eg2: prepare.sh 4 1 11301 12301 32351 22351

# 生成单链4节点集群的证书和配置
./prepare.sh 4 1
begin check params...
begin generate certs, cnt: 4
input consensus type (0-SOLO,1-TBFT(default),3-MAXBFT,4-RAFT):
input log level (DEBUG|INFO(default)|WARN|ERROR):
enable vm go (YES|NO(default))YES
vm go transport protocol (uds|tcp(default))
input vm go log level (DEBUG|INFO(default)|WARN|ERROR):
config node total 4
begin generate node1 config...
begin node1 chain1 cert config...
begin node1 trust config...
begin generate node2 config...
begin node2 chain1 cert config...
begin node2 trust config...
begin generate node3 config...
begin node3 chain1 cert config...
begin node3 trust config...
begin generate node4 config...
begin node4 chain1 cert config...
begin node4 trust config...

# 查看生成好的节点证书和配置
$ tree -L 3 ../build/
../build/
├── backup
│   ├── backup_certs
│   │   ├── crypto-config_20220816164215
│   │   ├── crypto-config_20220816193731
│   │   ├── crypto-config_20220816193745
│   │   ├── crypto-config_20220817195010
│   │   ├── crypto-config_20220819164151
│   │   ├── crypto-config_20220822204421
│   │   ├── crypto-config_20220823193308
│   │   └── crypto-config_20220823193419
│   ├── backup_config
│   │   └── config_20220823193425
│   └── backup_release
│       ├── release_20220808210208
│       ├── release_20220809203122
│       ├── release_20220816193816
│       ├── release_20220817195117
│       └── release_20220819164434
├── config
│   ├── node1
│   │   ├── certs
│   │   ├── chainconfig
│   │   ├── chainmaker.yml
│   │   └── log.yml
│   ├── node2
│   │   ├── certs
│   │   ├── chainconfig
│   │   ├── chainmaker.yml
│   │   └── log.yml
│   ├── node3
│   │   ├── certs
│   │   ├── chainconfig
│   │   ├── chainmaker.yml
│   │   └── log.yml
│   └── node4
│       ├── certs
│       ├── chainconfig
│       ├── chainmaker.yml
│       └── log.yml
├── crypto-config
│   ├── wx-org1.chainmaker.org
│   │   ├── ca
│   │   ├── node
│   │   └── user
│   ├── wx-org2.chainmaker.org
│   │   ├── ca
│   │   ├── node
│   │   └── user
│   ├── wx-org3.chainmaker.org
│   │   ├── ca
│   │   ├── node
│   │   └── user
│   └── wx-org4.chainmaker.org
│       ├── ca
│       ├── node
│       └── user
├── crypto_config.yml
├── pkcs11_keys.yml
└── release
    ├── chainmaker-v2.3.0_alpha-wx-org1.chainmaker.org
    │   ├── bin
    │   ├── config
    │   ├── data
    │   ├── lib
    │   └── log
    ├── chainmaker-v2.3.0_alpha-wx-org1.chainmaker.org-20220819164205-x86_64.tar.gz
    ├── chainmaker-v2.3.0_alpha-wx-org2.chainmaker.org
    │   ├── bin
    │   ├── config
    │   ├── lib
    │   └── log
    ├── chainmaker-v2.3.0_alpha-wx-org2.chainmaker.org-20220819164205-x86_64.tar.gz
    ├── chainmaker-v2.3.0_alpha-wx-org3.chainmaker.org
    │   ├── bin
    │   ├── config
    │   ├── lib
    │   └── log
    ├── chainmaker-v2.3.0_alpha-wx-org3.chainmaker.org-20220819164205-x86_64.tar.gz
    ├── chainmaker-v2.3.0_alpha-wx-org4.chainmaker.org
    │   ├── bin
    │   ├── config
    │   ├── lib
    │   └── log
    ├── chainmaker-v2.3.0_alpha-wx-org4.chainmaker.org-20220819164205-x86_64.tar.gz
    └── crypto-config-20220819164205.tar.gz
```



**方式二**

 修改节点配置目录下的`chainmaker.yml`文件（`enable`设置为`true`）:

```yaml
vm:
  go:
    # 是否启用新版Golang容器
    enable: true
```

### DockerVM配置

**配置信息说明**

chainmaker.yml 文件相关配置如下：

```yaml
vm:
  go:
    # 是否启用新版Golang容器
    enable: true
    # 数据挂载路径, 包括合约、sock文件（uds）
    data_mount_path: ../data/wx-org1.chainmaker.org/go
    # 日志挂载路径
    log_mount_path: ../log/wx-org1.chainmaker.org/go
    # chainmaker和合约引擎之间的通信协议（可选tcp/uds）
    protocol: tcp
    # 如果需要自定义高级配置，请将vm.yml文件放入dockervm_config_path中，优先级：chainmaker.yml > vm.yml > 默认配置
    # dockervm_config_path: /config_path/vm.yml
    # 是否在控制台打印日志
    log_in_console: false
    # docker合约引擎的日志级别
    log_level: DEBUG

    # 下面两个server的最大消息发送大小, 默认100MB
    max_send_msg_size: 100
    # 下面两个server的最大消息接收大小, 默认100MB
    max_recv_msg_size: 100
    # 下面两个server的最大连接超时时间, 默认10s
    dial_timeout: 10

    # 合约引擎最多启用的原始合约进程数，默认为20（跨合约调用会额外拉起新的进程）
    max_concurrency: 20

    # 运行时服务器配置 (与合约实例进程交互，进行信息交换)
    runtime_server:
      # 端口号，默认为 32351
      port: 32351

    # 合约引擎服务器配置 (与chainmaker交互，进行交易请求、合约请求等交互)
    contract_engine:
      # 合约引擎服务器ip, 默认为 127.0.0.1
      host: 127.0.0.1
      # 端口号，默认为 22351
      port: 22351
      # 与合约引擎服务器的最大连接数
      max_connection: 5
```

<span id="migration-docker_vm-configuration"></span>

⚠️注意：旧版本的合约不支持在新版DockerVM中运行，为了兼容历史合约，我们支持同时运行两个DockerVM，旧的DockerVM用于执行老版本的合约，所以在*chainmaker.yml*中有两套关于DockerVM的配置项`vm:go`、`vm:docker_go`分别对新、旧DockerVM进行配置。

新旧版本名称对照如下表：

|                            | 新版本                                         | 旧版本                                            |
| -------------------------- | ---------------------------------------------- | ------------------------------------------------- |
| 官方镜像名称               | chainmakerofficial/chainmaker-vm-engine:v2.3.0 | chainmakerofficial/chainmaker-vm-docker-go:v2.3.0 |
| chainmaker.yml中对应配置项 | vm:go                                          | vm:docker_go                                      |
| 合约版本                   | v2.3.0                                         | v2.2.1及更低版本                                  |

如果需要启动**旧合约引擎容器**，需要将之前的配置迁移到`vm:docker_go`：

```yaml
# Contract Virtual Machine(VM) configs
vm:
  # Docker go virtual machine configuration
  docker_go:
    # Enable docker go virtual machine
    enable_dockervm: true
    # Mount point in chainmaker
    dockervm_mount_path: ../data/wx-org1.chainmaker.org/docker-go
    # Specify log file path
    dockervm_log_path: ../log/wx-org1.chainmaker.org/docker-go
    # Whether to print log at terminal
    log_in_console: true
    # Log level
    log_level: DEBUG
    # Unix domain socket open, used for chainmaker and docker manager communication
    uds_open: true
    # docker vm contract service host, default 127.0.0.1
    docker_vm_host: 127.0.0.1
    # docker vm contract service port, default 22351
    docker_vm_port: 22451
    # Grpc max send message size, Default size is 4, Unit: MB
    max_send_msg_size: 20
    # Grpc max receive message size, Default size is 4, Unit: MB
    max_recv_msg_size: 20
    # max number of connection created to connect docker vm service
    max_connection: 5
```

启用旧版DockerVM时，配置迁移对照如下图：

<img loading="lazy" src="../images/DockerVM_Update_Config.png" style="width:1024px;" />

**通过配置废弃旧合约的安装或升级，只使用新合约**

如果希望废弃旧合约的安装或升级，只使用新合约，请添加或修改下面的配置：

```yaml
# Contract Virtual Machine(VM) configs
vm:
  # Docker go virtual machine configuration
  docker_go:
    # Grpc max receive message size, Default size is 4, Unit: MB
    disable_install: true
    # max number of connection created to connect docker vm service
    disable_upgrade: true
```

**高级配置**

如果希望使用合约引擎高级配置，需要配置`vm:go`下`dockervm_config_path`中配置文件，配置文件模板如下：

```yaml
########### RPC ###########
rpc:
  chain_rpc_protocol: 1 # chain rpc protocol, 0 for unix domain socket, 1 for tcp(default)
  chain_host: 127.0.0.1 # chain tcp host
  chain_rpc_port: 22351 # chain rpc port, valid when protocol is tcp
  sandbox_rpc_port: 32351 # sandbox rpc port, valid when protocol is tcp
  max_send_msg_size: 100 # max send msg size(MiB)
  max_recv_msg_size: 100 # max recv msg size(MiB)
  server_min_interval: 60s # server min interval
  connection_timeout: 5s # connection timeout time
  server_keep_alive_time: 60s # idle duration before server ping
  server_keep_alive_timeout: 20s # ping timeout

########### Process ###########
process:
  # max original process num,
  # max_call_contract_process_num = max_original_process_num * max_contract_depth (defined in protocol)
  # max_total_process_num = max_call_contract_process_num + max_original_process_num
  max_original_process_num: 20
  exec_tx_timeout: 8s # process timeout while busy
  waiting_tx_time: 200ms # process timeout while tx completed (busy -> idle)
  release_rate: 30 # percentage of idle processes released periodically in total processes (0-100)
  release_period: 10m # period of idle processes released periodically in total processes

########### Log ###########
log:
  contract_engine:
    level: "info"
    console: true
  sandbox:
    level: "info"
    console: true

########### Pprof ###########
pprof:
  contract_engine:
    enable: false
    port: 21215
  sandbox:
    enable: false
    port: 21522

########### Contract ###########
contract:
  max_file_size: 20480 # contract size(MiB)
```

**⚠️注意：**

在容器启动脚本中，`max_concurrency`（最大启用的进程数量）默认为`20`（跨合约调用会额外拉起新的进程）。

如果是在生产环境下，建议根据cpu核数配置这三个参数。

|                 | 8C CPU | 16C CPU | 32C CPU |
| --------------- | ------ | ------- | ------- |
| max_concurrency | 20     | 100     | 1500    |

如果有较多跨合约调用交易，请根据跨合约调用的深度按比例减少`max_concurrency`的值，例如32核系统下，如果有较多两层跨合约调用，则`max_concurrency`值建议设为`750（1500/2）`。

如果使用脚本启动，请按需修改脚本里的参数配置。

### 编译和安装包制作

- 生成证书（prepare.sh脚本）后执行build_release.sh脚本，将编译chainmaker-go模块，并打包生成安装，存于路径chainmaker-go/build/release中

```shell
$ ./build_release.sh
$ tree ../build/release/
../build/release/
├── chainmaker-v2.3.0_alpha-wx-org1.chainmaker.org-20220823193812-x86_64.tar.gz
├── chainmaker-v2.3.0_alpha-wx-org2.chainmaker.org-20220823193812-x86_64.tar.gz
├── chainmaker-v2.3.0_alpha-wx-org3.chainmaker.org-20220823193812-x86_64.tar.gz
├── chainmaker-v2.3.0_alpha-wx-org4.chainmaker.org-20220823193812-x86_64.tar.gz
└── crypto-config-20220823193812.tar.gz
```



### 同时启动链和DockerVM

**启动**

- 执行cluster_quick_start.sh脚本，会解压各个安装包，调用bin目录中的start.sh脚本，启动chainmaker节点，并拉起节点需要的DockerVM容器

```shell
$ ./cluster_quick_start.sh normal
```

**查看节点是否存在**

- 查看进程

```shell
$ ps -ef|grep chainmaker | grep -v grep
2058348       1  4 19:40 pts/5    00:00:00 ./chainmaker start -c ../config/wx-org1.chainmaker.org/chainmaker.yml
2059604       1  3 19:40 pts/5    00:00:00 ./chainmaker start -c ../config/wx-org2.chainmaker.org/chainmaker.yml
2060801       1  4 19:40 pts/5    00:00:00 ./chainmaker start -c ../config/wx-org3.chainmaker.org/chainmaker.yml
2062057       1  5 19:40 pts/5    00:00:00 ./chainmaker start -c ../config/wx-org4.chainmaker.org/chainmaker.yml
```

- 查看端口

```shell
$ netstat -lptn | grep 1230
tcp6       0      0 :::12301                :::*                    LISTEN      2058348/./chainmake
tcp6       0      0 :::12302                :::*                    LISTEN      2059604/./chainmake
tcp6       0      0 :::12303                :::*                    LISTEN      2060801/./chainmake
tcp6       0      0 :::12304                :::*                    LISTEN      2062057/./chainmake
```

- 查看DockerVM容器

```shell
$ docker ps | grep "chainmakerofficial/chainmaker-vm-engine:v2.3.0"
0955ccdb6ebc   chainmakerofficial/chainmaker-vm-engine:v2.3.0   "/bin/startvm"           4 minutes ago   Up 4 minutes                                                               VM-GO-wx-org4.chainmaker.org
b48bbb69e204   chainmakerofficial/chainmaker-vm-engine:v2.3.0   "/bin/startvm"           4 minutes ago   Up 4 minutes                                                               VM-GO-wx-org3.chainmaker.org
727adbb76c58   chainmakerofficial/chainmaker-vm-engine:v2.3.0   "/bin/startvm"           4 minutes ago   Up 4 minutes                                                               VM-GO-wx-org2.chainmaker.org
2c944fb3a1d9   chainmakerofficial/chainmaker-vm-engine:v2.3.0   "/bin/startvm"           4 minutes ago   Up 4 minutes                                                               VM-GO-wx-org1.chainmaker.org
```

- 查看日志

```shell
$ cat ../build/release/*/bin/panic.log
$ cat ../build/release/*/log/system.log
$ cat ../build/release/*/log/system.log |grep "ERROR\|put block\|all necessary"
```



### 独立部署DockerVM虚拟机

**首先去`./build/release`路径下解压各个节点证书及配置的jar包:**

```shell
tar -zxvf chainmaker-v2.3.0_alpha-wx-org4.chainmaker.org-20220824115035-x86_64.tar.gz
```

**启动节点1需要的DockerVM**

- 使用*docker-vm-standalone-start.sh*脚本启动DockerVM

```shell
$ ./docker-vm-standalone-start.sh
input path to cache contract files(must be absolute path, default:'./docker-go'): /home/data/wx-org1.chainmaker.org/go
contracts path does not exist, create it or not(y|n): y
input log path(must be absolute path, default:'./log'): /home/log/wx-org1.chainmaker.org/go
log path does not exist, create it or not(y|n): y
input log level(DEBUG|INFO(default)|WARN|ERROR): DEBUG
input expose port(default 22351): 22351
input runtime port(default 32351): 32351
input container name(default 'chainmaker-docker-vm'): VM-GO-wx-org1.chainmaker.org
# 不使用配置文件启动DockerVM，忽略该项
input vm config file path(use default config(default)):
docker-vm config is nil, use default config
start docker vm container
```

- 使用Docker命令启动

```shell
$ docker run -itd \
--net=host \
-v "/home/data/wx-org1.chainmaker.org/go":/mount \
-v "/home/log/wx-org1.chainmaker.org/go":/log \
-e CHAIN_RPC_PROTOCOL="1" \
-e CHAIN_RPC_PORT="22351" \
-e SANDBOX_RPC_PORT="32351" \
-e MAX_SEND_MSG_SIZE="100" \
-e MAX_RECV_MSG_SIZE="100" \
-e MAX_CONN_TIMEOUT="10" \
-e MAX_ORIGINAL_PROCESS_NUM="20" \
-e DOCKERVM_CONTRACT_ENGINE_LOG_LEVEL="DEBUG" \
-e DOCKERVM_SANDBOX_LOG_LEVEL="DEBUG" \
-e DOCKERVM_LOG_IN_CONSOLE="false" \
--name VM-GO-wx-org1.chainmaker.org \
--privileged chainmakerofficial/chainmaker-vm-engine:v2.3.0 \
> /dev/null
```

**然后启动节点1**

```shell
$ ./start.sh -f alone
```

**查看节点和容器是否已经建立链接**

```shell
$ cat /home/log/wx-org1.chainmaker.org/go/go.log |grep "Chain RPC Service"
```

按照上述步骤依次启动剩余节点。

### 使用高级配置独立部署DockerVM

**高级配置**

如果希望使用合约引擎高级配置，需要配置`vm:go`下`dockervm_config_path`高级配置文件路径，这里我们把节点1的DockerVM高级配置文件*vm.yml*放置在`/home/config_path/wx-org1.chainmaker.org`，在*chainmaker.yml*中对应的配置如下：

```yaml
# Contract Virtual Machine(VM) configs
vm:
  # Golang runtime in docker container
  go:
    ...
    # If use a customized VM configuration file, supplement it; else, do not configure
    # Priority: chainmaker.yml > vm.yml > default settings
    dockervm_config_path: /home/config_path/wx-org1.chainmaker.org/vm.yml
   	...
```

配置文件模板如下：

```yaml
########### RPC ###########
rpc:
  chain_rpc_protocol: 1 # chain rpc protocol, 0 for unix domain socket, 1 for tcp(default)
  chain_host: 127.0.0.1 # chain tcp host
  chain_rpc_port: 22351 # chain rpc port, valid when protocol is tcp
  sandbox_rpc_port: 32351 # sandbox rpc port, valid when protocol is tcp
  max_send_msg_size: 100 # max send msg size(MiB)
  max_recv_msg_size: 100 # max recv msg size(MiB)
  server_min_interval: 60s # server min interval
  connection_timeout: 5s # connection timeout time
  server_keep_alive_time: 60s # idle duration before server ping
  server_keep_alive_timeout: 20s # ping timeout

########### Process ###########
process:
  # max original process num,
  # max_call_contract_process_num = max_original_process_num * max_contract_depth (defined in protocol)
  # max_total_process_num = max_call_contract_process_num + max_original_process_num
  max_original_process_num: 20
  exec_tx_timeout: 8s # process timeout while busy
  waiting_tx_time: 200ms # process timeout while tx completed (busy -> idle)
  release_rate: 30 # percentage of idle processes released periodically in total processes (0-100)
  release_period: 10m # period of idle processes released periodically in total processes

########### Log ###########
log:
  contract_engine:
    level: "info"
    console: true
  sandbox:
    level: "info"
    console: true

########### Pprof ###########
pprof:
  contract_engine:
    enable: false
    port: 21215
  sandbox:
    enable: false
    port: 21522

########### Contract ###########
contract:
  max_file_size: 20480 # contract size(MiB)
```

**使用高级配置启动节点需要的DockerVM**

- 使用*docker-vm-standalone-start.sh*脚本启动DockerVM

```shell
$ ./docker-vm-standalone-start.sh
input path to cache contract files(must be absolute path, default:'./docker-go'): /home/data/wx-org1.chainmaker.org/go
contracts path does not exist, create it or not(y|n): y
input log path(must be absolute path, default:'./log'): /home/log/wx-org1.chainmaker.org/go
log path does not exist, create it or not(y|n): y
## 使用配置文件，忽略该项
input log level(DEBUG|INFO(default)|WARN|ERROR):
## 使用配置文件，忽略该项
input expose port(default 22351):
## 使用配置文件，忽略该项
input runtime port(default 32351):
input container name(default 'chainmaker-docker-vm'): VM-GO-wx-org1.chainmaker.org
input vm config file path(use default config(default)): /home/config_path/wx-org1.chainmaker.org/vm.yml
start docker vm container
```

- 使用Docker命令启动DockerVM

```shell
$ docker run -itd \
--net=host \
-v "/home/data/wx-org1.chainmaker.org/go":/mount \
-v "/home/log/wx-org1.chainmaker.org/go":/log \
--name VM-GO-wx-org1.chainmaker.org \
--privileged chainmakerofficial/chainmaker-vm-engine:v2.3.0 \
> /dev/null
```

**启动节点**

```shell
$ ./start.sh -f alone
```

**查看节点是否和DockerVM已经建立连接**

```shell
$ cat /home/log/wx-org1.chainmaker.org/go/go.log |grep "Chain RPC Service"
```

按照上述步骤依次启动剩余节点。

### 本地部署的停止

如果是通过`./cluster_quick_start.sh`同时启动的节点和虚拟机，可以通过对应的stop脚本同时停止链和DockerVM容器：

```shell
$ ./cluster_quick_stop.sh
```

如果同时需要清除所有链数据可以使用：

```shell
$ ./cluster_quick_stop.sh clean
```
由于DockerVM内使用容器内的root用户启动虚拟机服务，因此日志和缓存文件也属于root用户（uid=0）。如果以非root用户启动的程序，清除数据时可能会报错提示缺少文件访问的权限，需要以root权限删除数据，或者使用[userns-remap](https://docs.docker.com/engine/security/userns-remap/)的功能将容器里用户映射成普通用户。


### 独立部署的停止

**首先关闭链节点**

```shell
$ ./stop.sh alone
```

**然后关闭该节点对应的虚拟机**

```shell
$ docker stop VM-GO-wx-org1.chainmaker.org
```

## 合约的安装与调用

合约的安装与调用请参考：[使用Golang进行智能合约开发](./使用Golang进行智能合约开发.md)

