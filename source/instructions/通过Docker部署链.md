
## docker部署（普通证书模式）
作者：长安链团队 宋丹阳

### 配置证书生成节点个数

- 进入`chainmaker-go/tools/chainmaker-cryptogen/config`目录，修改`crypto_config_template.yml`文件中`count`
```
crypto_config:
  - domain: chainmaker.org
    host_name: wx-org
    count: 4                # change this what you want, example is 7 node
```

### 证书生成

- 进入chainmaker-go/scripts目录，执行prepare.sh脚本生成单链4节点集群配置，存于路径chainmaker-go/build中

```
# 进入脚本目录
$ cd ../scripts
# 查看脚本帮助
$ ./prepare.sh -h
Usage:  
  prepare.sh node_cnt(1/4/7/10/13/16) chain_cnt(1-4) p2p_port(default:11301) rpc_port(default:12301)
    eg1: prepare.sh 4 1
    eg2: prepare.sh 4 1 11301 12301

# 生成单链4节点集群的证书和配置
$ ./prepare.sh 4 1
begin check params...
begin generate certs, cnt: 4
input consensus type (0-SOLO,1-TBFT(default),3-HOTSTUFF,4-RAFT,5-DPOS):
input log level (DEBUG|INFO(default)|WARN|ERROR):
enable docker vm (YES|NO(default))
begin generate node1 config...
begin generate node2 config...
begin generate node3 config...
begin generate node4 config...

# 查看生成好的节点证书和配置
$ tree -L 3 ../build/
../build/
├── config
│   ├── node1
│   │   ├── certs
│   │   ├── chainconfig
│   │   ├── chainmaker.yml
│   │   └── log.yml
│   ├── node2
│   │   ├── certs
│   │   ├── chainconfig
│   │   ├── chainmaker.yml
│   │   └── log.yml
│   ├── node3
│   │   ├── certs
│   │   ├── chainconfig
│   │   ├── chainmaker.yml
│   │   └── log.yml
│   └── node4
│       ├── certs
│       ├── chainconfig
│       ├── chainmaker.yml
│       └── log.yml
├── crypto-config
│   ├── wx-org1.chainmaker.org
│   │   ├── ca
│   │   ├── node
│   │   └── user
│   ├── wx-org2.chainmaker.org
│   │   ├── ca
│   │   ├── node
│   │   └── user
│   ├── wx-org3.chainmaker.org
│   │   ├── ca
│   │   ├── node
│   │   └── user
│   └── wx-org4.chainmaker.org
│       ├── ca
│       ├── node
│       └── user
└── crypto_config.yml
```

### 镜像配置

- 进入`chainmaker-go/scripts/docker/multi_node`目录，修改`create_docker_compose_yml.sh`中IMAGE为所需镜像

```
P2P_PORT=$1
RPC_PORT=$2
NODE_COUNT=$3
CONFIG_DIR=$4
SERVER_COUNT=$5
IMAGE="chainmakerofficial/chainmaker:v2.2.0" # change this
```

### 创建docker-composer.yaml文件

- 将生成的证书配置文件放到`scripts/docker/multi_node`目录中，然后修改对应`chainmaker.yml`文件中的本地ip
- 使用`create_docker_compose_yml.sh`进行docker-composer.yml文件生成

```
$ cd chainmaker-go
$ cp -rf build/config scripts/docker/multi_node/
$ cd scripts/docker/multi_node
# change ip what you want
$ sed -i "s%127.0.0.1%192.168.1.35%g" config/node*/chainmaker.yml
# check 
$ ./create_docker_compose_yml.sh
Usage:  
  create_yml.sh P2P_PORT RPC_PORT NODE_COUNT CONFIG_DIR SERVER_NODE_COUNT
    P2P_PORT:          peer to peer connect
    RPC_PORT:          sdk to peer connect
    NODE_COUNT:        total node count
    CONFIG_DIR:        all node config path, relative or absolute
    SERVER_NODE_COUNT: default:100, number of nodes per server

    eg: ./create_docker_compose_yml.sh 11301 12301 4 ./config : 4 nodes in 1 machine
    eg: ./create_docker_compose_yml.sh 11301 12301 16 ./config 2 : 4 nodes in 8 machine, 2 nodes per machine
    eg: ./create_docker_compose_yml.sh 11301 12301 4 /data/workspace/chainmaker-go/build/config 4
$ ./create_docker_compose_yml.sh 11331 12331 7 ./config 4
$ tree -L 1
├── config
├── create_docker_compose_yml.sh
├── docker-compose1.yml
├── readme.md
├── start.sh
├── stop.sh
└── tpl_docker-compose_services.yml
```
### 启动节点

```
$ cd chainmaker-go/scripts/docker/multi_node/
# 启动节点
$ docker-compose -f docker-compose1.yml up -d
# or 
$./start.sh docker-compose1.yml
```

### 停止节点

```
$ cd chainmaker-go/scripts/docker/multi_node/
# 停止节点
$ docker-compose -f docker-compose1.yml down
# or 
$./stop.sh docker-compose1.yml
```


### 查看节点是否存在
- 查看进程
```
$ ps -ef|grep chainmaker | grep -v grep
25261  2146  4 19:55 pts/20   00:00:01 ./chainmaker start -c ../config/wx-org1.chainmaker.org/chainmaker.yml
25286  2146  4 19:55 pts/20   00:00:01 ./chainmaker start -c ../config/wx-org2.chainmaker.org/chainmaker.yml
25309  2146  4 19:55 pts/20   00:00:01 ./chainmaker start -c ../config/wx-org3.chainmaker.org/chainmaker.yml
25335  2146  4 19:55 pts/20   00:00:01 ./chainmaker start -c ../config/wx-org4.chainmaker.org/chainmaker.yml
```
- 查看端口
```
$ netstat -lptn | grep 1230
tcp6       0      0 :::12301                :::*                    LISTEN      25261/./chainmaker  
tcp6       0      0 :::12302                :::*                    LISTEN      25286/./chainmaker  
tcp6       0      0 :::12303                :::*                    LISTEN      25309/./chainmaker  
tcp6       0      0 :::12304                :::*                    LISTEN      25335/./chainmaker 
```
- 查看日志
```
$ cat ../build/release/*/bin/panic.log
$ cat ../build/release/*/log/system.log
$ cat ../build/release/*/log/system.log |grep "ERROR\|put block\|all necessary"
//若看到all necessary peers connected则表示节点已经准备就绪。
```

### 部署/调用合约验证链是否正常
启动成功后，可进行部署/调用示例合约，以检查链功能是否正常。部署合约的使用教程可详见：[部署示例合约](./部署示例合约.md)。