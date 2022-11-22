### 安装部署
####  拉取官方镜像

Docker VM官方镜像：

```shell
docker pull chainmakerofficial/chainmaker-vm-engine:v2.3.0
```

#### 在chainmaker中启用Docker VM

1. 方式一： [通过命令行体验链](../quickstart/通过命令行体验链.md)，在执行`prepare.sh`、`prepare_pk.sh`、`prepare_pwk.sh`时，`enable docker vm` 选择 YES
    ```sh
    enable docker vm (YES|NO(default))
    ```

2. 方式二： 修改节点配置目录下的`chainmaker.yml`文件（`enable`设置为`true`）:

    ```yaml
    vm:
      go:
        # 是否启用新版Golang容器
        enable: true
    ```

   详细内容参考[chainmaker.yml vm模块配置](#chainmaker.yml_vm)

#### Docker VM的部署和启动
按照 Docker VM 与chainmaker通信方式的不同，支持两种部署方式，默认为 TCP 方式，v2.3.0之前默认使用 UNIX Domain Socket 方式。

##### 本机同时启动节点和Docker VM

本机部署时，合约引擎（Docker VM）运行在单独的容器中，容器中每个合约运行实例都用单独的进程来启动。此时chainmaker与合约引擎的通信方式可以选择 **TCP 或 UNIX Domain Socket**

1. `chainmaker.yml`配置

    * `chainmaker-go`和`Docker VM`部署在同一host

    * protocol: tcp(默认) / uds

    * 文件挂载和其他配置说明：
        - 在 UNIX Domain Socket 的连接模式中，合约文件、socket文件和日志文件都通过docker mount的形式挂载到Docker VM容器中:
            - `data_mount_path`会挂载到Docker VM容器中的`/mount`路径下
            - `log_mount_path`会挂载到`/log`路径下
            - 此时关于log的两个配置生效：`log_in_console, log_level`
            - 关于Docker VM虚拟机服务的ip和端口的配置不生效：`runtime_server_host, contract_engine_host, contract_engine_port`

2. 启动docker vm：

   在 UNIX Domain Socket 的连接模式中，由`prepare.sh`生成的`bin/start.sh`会自动拉起Docker VM容器，不需要额外启动。

3. 停止docker vm：

   在 UNIX Domain Socket 的连接模式中，停止节点时，由`prepare.sh`生成的`bin`目录下的 `stop.sh` 脚本自动停止Docker VM容器。
   也可以使用docker命令单独停止Docker VM容器

<span id="dockerVmStandalone"></span>

##### 节点与Docker VM分离部署

分离部署后，chainmaker与合约引擎间**只允许通过TCP**通信。

1. chainmaker.yml配置
    * protocol: tcp
    * 文件挂载和其他配置说明：在 tcp 的连接模式中，文件挂载通过docker-vm启动时-v指定挂载目录  
      此时以下关于Docker VM虚拟机服务的ip和端口的配置生效：`runtime_server_host, contract_engine_host, contract_engine_port`
      此时以下关于log的两个配置不生效：`log_in_console, log_level`

2. 启动docker vm：

   `chainmaker-go`提供了用于在 tcp 的连接模式中启动Docker VM容器的脚本：
   由`prepare.sh`生成的`bin`目录下的 `docker-vm-standalone-start.sh`
   链需要使用`start.sh alone`启动。

4. 停止docker vm：

   由`prepare.sh`生成的`bin`目录下的 `docker-vm-standalone-stop.sh`


<span id="chainmaker.yml_vm"></span>


##### Docker VM 容器配置说明

容器的运行需要privileged的权限，启动命令需要添加 `--privileged` 参数。

配置支持使用chainmaker.yml文件、自定义配置文件、使用默认配置

chainmaker.yml文件相关配置如下：

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

如果希望使用合约引擎高级配置，需要配置`vm_go`下`dockervm_config_path`中配置文件，配置文件模板如下：

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

**注：**

在容器启动脚本中，`max_concurrency`（最大启用的进程数量）默认为`20`（跨合约调用会额外拉起新的进程）。

如果是在生产环境下，建议根据cpu核数配置这三个参数。

|                    | 8C CPU | 16C CPU   | 32C CPU   |
|--------------------|--------|-----------|-----------|
| max_concurrency    | 20     | 100       | 1500      |

如果有较多跨合约调用交易，请根据跨合约调用的深度按比例减少`max_concurrency`的值，例如32核系统下，如果有较多两层跨合约调用，则`max_concurrency`值建议设为`750（1500/2）`。

如果使用脚本启动，请按需修改脚本里的参数配置。