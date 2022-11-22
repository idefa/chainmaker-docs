# 搭建PWK模式账户体系
作者：长安链团队 韩学洋

## 使用cmc搭建长安链网络
长安链支持PermissionWithCert、PermissionWithKey、Public等三种不同账户模式的链，本章节我们将详细介绍如何通过cmc命令行工具搭建完整的长安链PermissionWithKey模式的账户体系，以及如果管理该模式下的链账户，包括组织账户、节点账户、用户账户的增删等。

本文示例说明：
- 创建新链：4组织，1共识节点/每组织，1管理员/每组织
- 链管理：组织管理、节点管理、用户管理

### 部署cmc命令行工具

长安链CMC工具可用于生成公私钥，部署PermissionWithKey模式的长安链前，我们需要通过长安链CMC工具生成相关的公私钥文件。
长安链命令行工具cmc安装, 请参考[长安链命令行工具](../dev/命令行工具pwk.html#编译&配置)

### 基于cmc工具生成公钥账户

由于pwk模式无证书概念，因此对于节点、管理员以及用户而言，都是公私钥的形式。 节点账户的地址需要添加到共识列表，管理员账户需要添加到管理员列表；
而普通用户账户，需要管理员单独进行注册。

#### 生成节点账户
```shell
  # 生成节点账户私钥
$ ./cmc key gen -a ECC_P256 -p ./ -n consensus1.key
 
  # 导出节点账户公钥
$ ./cmc key export_pub -k consensus1.key -n consensus1.pem
 
 # 计算节点账户地址
$ ./cmc cert nid --node-pk-path=./consensus1.pem
node id : QmchuAwzWLLhKWTU6ok8DHHb1QGLFG9bB8wYRUTGJpeXp5

# 查看节点账户（公钥）
$ cat consensus1.pem
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE+TKd7KW8GfQYi9hYsrD7TSarF6iJ
OHo8Hp42++YW6sevy3UmXcLDLi01z7UO5eQGm/E7O8pACp2RCsueAwbTIA==
-----END PUBLIC KEY-----
```

#### 生成admin账户
```shell
# 生成管理员私钥
 ./cmc key gen -a ECC_P256 -p ./ -n admin.key

# 导出管理员公钥
 ./cmc key export_pub -k admin.key -n admin.pem
 
# 查看管理员公钥（账户）
$ cat admin.pem
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEsIYfDrQCt1/H8Yj5KKKD+uO28zz7
nTovDgim/jezoGdpmNOUp6lwrN47pxBUpnxEXIqHBwz8uaVR1z3y9kDvTg==
-----END PUBLIC KEY-----
```

重复以上步骤分别生成org1，org2，org3以及org4组织的节点账户和管理员账户。


### 基于生成的节点账户和管理员账户创建链
获取的节点账户和管理员账户，需要在启动链时，将他们配置到链配置文件`bc1.yml`的`trust_roots`里，并将`bc1.yml`和`chainmaker.yml`中的`nodes.node_id`替换为以上获取的节点nodeId。

**配置文件修改位置如下**

- bc1.yml（链配置文件）

```yaml
#共识配置
consensus:
  # 共识类型(0-SOLO,1-TBFT,2-MBFT,3-MAXBFT,4-RAFT,10-POW)
  type: 1
  # 共识节点列表，组织必须出现在trust_roots的org_id中，每个组织可配置多个共识节点，节点地址采用libp2p格式
  nodes:
    - org_id: "wx-org1.chainmaker.org"
      node_id:
        - "QmchuAwzWLLhKWTU6ok8DHHb1QGLFG9bB8wYRUTGJpeXp5"
    - org_id: "wx-org2.chainmaker.org"
      node_id:
        - "QmeyNRs2DwWjcHTpcVHoUSaDAAif4VQZ2wQDQAUNDP33gH"
    - org_id: "wx-org3.chainmaker.org"
      node_id:
        - "QmXf6mnQDBR9aHauRmViKzSuZgpumkn7x6rNxw1oqqRr45"
    - org_id: "wx-org4.chainmaker.org"
      node_id:
        - "QmRRWXJpAVdhFsFtd9ah5F4LDQWFFBDVKpECAF8hssqj6H"
      
trust_roots:
  - org_id: "wx-org1.chainmaker.org"
    root:
      - "../config/wx-org1.chainmaker.org/keys/admin/wx-org1.chainmaker.org/admin.pem"
  - org_id: "wx-org2.chainmaker.org"
    root:
      - "../config/wx-org1.chainmaker.org/keys/admin/wx-org2.chainmaker.org/admin.pem"
  - org_id: "wx-org3.chainmaker.org"
    root:
      - "../config/wx-org1.chainmaker.org/keys/admin/wx-org3.chainmaker.org/admin.pem"
  - org_id: "wx-org4.chainmaker.org"
    root:
      - "../config/wx-org1.chainmaker.org/keys/admin/wx-org4.chainmaker.org/admin.pem"
```

- chainmaker.yml （节点配置文件）

```yaml
# Network Settings
net:
  # Network provider, can be libp2p or liquid.
  # libp2p: using libp2p components to build the p2p module.
  # liquid: a new p2p module we build from 0 to 1.
  # This item must be consistent across the blockchain network.
  provider: LibP2P

  # The address and port the node listens on.
  # By default, it uses 0.0.0.0 to listen on all network interfaces.
  listen_addr: /ip4/0.0.0.0/tcp/11301

  # The seeds peer list used to join in the network when starting.
  # The connection supervisor will try to dial seed peer whenever the connection is broken.
  # Example ip format: "/ip4/127.0.0.1/tcp/11301/p2p/"+nodeid
  # Example dns format："/dns/cm-node1.org/tcp/11301/p2p/"+nodeid
  seeds:
    - "/ip4/127.0.0.1/tcp/11301/p2p/QmchuAwzWLLhKWTU6ok8DHHb1QGLFG9bB8wYRUTGJpeXp5"
    - "/ip4/127.0.0.1/tcp/11302/p2p/QmeyNRs2DwWjcHTpcVHoUSaDAAif4VQZ2wQDQAUNDP33gH"
    - "/ip4/127.0.0.1/tcp/11303/p2p/QmXf6mnQDBR9aHauRmViKzSuZgpumkn7x6rNxw1oqqRr45"
    - "/ip4/127.0.0.1/tcp/11304/p2p/QmRRWXJpAVdhFsFtd9ah5F4LDQWFFBDVKpECAF8hssqj6H"
```

**节点部署和链启动**




节点部署目录如下图所示：
```shell
.
├── bin
│   ├── chainmaker
│   ├── chainmaker.service
│   ├── docker-vm-standalone-start.sh
│   ├── docker-vm-standalone-stop.sh
│   ├── init.sh
│   ├── panic.log
│   ├── restart.sh
│   ├── run.sh
│   ├── start.sh
│   └── stop.sh
├── config
│   └── wx-org1.chainmaker.org
├── lib
│   ├── libwasmer.dylib
│   └── wxdec
└── log
```

在启动节点之前，需要将以上部署过程中用到的各个节点公私钥、管理员公私钥替换为以上用cmc生成的公私钥对。
同时各个节点部署目录下的`bc.yml`和`chainmaker.yml`配置文件按照以上说明进行修改。

**配置更新说明**
```shell
# 以组织org1为例说明
./config
└── wx-org1.chainmaker.org
├── chainconfig
│   └── bc1.yml # 更新后的链配置文件，参考上节说明
├── chainmaker.yml # 更新后的节点配置文件，参考上节说明
├── keys
│   ├── admin
│   │   ├── wx-org1.chainmaker.org
│   │   │   └── admin.pem  #组织1管理员公钥
│   │   ├── wx-org2.chainmaker.org
│   │   │   └── admin.pem #组织2管理员公钥
│   │   ├── wx-org3.chainmaker.org
│   │   │   └── admin.pem #组织3管理员公钥
│   │   └── wx-org4.chainmaker.org
│   │       └── admin.pem #组织4管理员公钥
│   ├── node
│   │   ├── common1 #optional
│   │   │   ├── common1.key
│   │   │   ├── common1.nodeid
│   │   │   └── common1.pem
│   │   └── consensus1
│   │       ├── consensus1.key #共识节点私钥
│   │       ├── consensus1.nodeid #共识节点nodeId
│   │       └── consensus1.pem #共识节点公钥
│   └── user
│       ├── admin1  #optional
│       │   ├── admin1.key
│       │   └── admin1.pem
│       ├── client1  #optional
│       │   ├── client1.addr
│       │   ├── client1.key
│       │   └── client1.pem
│       └── light1  #optional
│           ├── light1.key
│           └── light1.pem
└── log.yml
```

**启动节点**  

启动pwk模式的链与启动证书模式的链使用的方式类似，可以参考证书模式。在密钥生成步骤使用`./prepare_pwk.sh`脚本即可。   

- docker部署方式，参考[docker方式-启动节点](./通过Docker部署链.html#启动节点)
- 命令行部署方式，参考[通过命令行部署链](../quickstart/通过命令行体验链.html#节点启动)

按照以上方式启动org1、org2、org3和org4下的共识节点，等到所有节点建立连接。表明区块链网络部署成功，并可以对外提供区块链服务。  
节点日志如下：

```shell
[INFO]  [Net]   libp2pnet/libp2p_connection_supervisor.go:116   [ConnSupervisor] all necessary peers connected.
```


### 部署/调用智能合约
链部署后，在进行部署/调用合约测试，以验证链是否正常运行。目前支持两种方式进行合约的部署和调用。示例合约可从[此处获得](../instructions/部署示例合约.md)

#### 使用CMC工具测试
客户端账户包含client和admin两种类型，admin一般具有更高的链上权限，我们这里以admin为例进行测试

使用长安链命令行工具cmc进行测试，详细教程请见命令行[交易功能](../dev/命令行工具pwk.html#交易功能)
其中cmc工具配置文件sdk_config_pwk.yaml需要进行相应修改，主要修改以下配置:

- 客户端私钥：user_sign_key_file_path

```yaml
chain_client:
  # 链ID
  chain_id: "chain1"
  # 组织ID
  org_id: "wx-org1.chainmaker.org"

  # 客户端用户交易签名私钥路径
  user_sign_key_file_path: "./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key" #组织1管理员私钥文件
  # 签名使用的哈希算法，和节点保持一直
  crypto:
    hash: SHA256
  auth_type: permissionedWithKey
  # 默认支持TimestampKey，如果开启enableNormalKey则使用NormalKey
  enable_normal_key: false

  nodes:
    - # 节点地址，格式为：IP:端口:连接数
      node_addr: "127.0.0.1:12301"
      # 节点连接数
      conn_cnt: 10
    - # 节点地址，格式为：IP:端口:连接数
      node_addr: "127.0.0.1:12302"
      # 节点连接数
      conn_cnt: 10

  archive:
    # 数据归档链外存储相关配置
    type: "mysql"
    dest: "root:123456:localhost:3306"
    secret_key: xxx

  rpc_client:
    max_receive_message_size: 16 # grpc客户端接收消息时，允许单条message大小的最大值(MB)
    max_send_message_size: 16 # grpc客户端发送消息时，允许单条message大小的最大值(MB)%
```

注：由于我们使用管理员账户进行测试，所以user_sign_key_file_path需要指定管理员私钥


#### 使用长安链SDK进行测试
- 通过长安链SDK进行部署/调用，详情[SDK使用说明章节](../instructions/GoSDK使用说明.md)
  - 需要将SDK所需的相关证书替换成，上文所生成的证书，然后进行操作。





## 链管理
在长安链中，不同的账户一般绑定不同的角色，具有不同的权限。 为了提高安全性，长安链默认设置了许多权限，部分操作需要多个管理员多签才能完成。 

### 管理链的组织
- 添加共识节点Org
```shell
./cmc client chainconfig consensusnodeorg add \
--sdk-conf-path=./testdata/sdk_config_pwk.yml \
--org-id=wx-org1.chainmaker.org \
--admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
--node-ids=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4,QmaWrR72CbT51nFVpNDS8NaqUZjVuD4Ezf8xcHcFW9SJWF \
--node-org-id=wx-org5.chainmaker.org
```
- 删除共识节点Org
```shell
./cmc client chainconfig consensusnodeorg remove \
--sdk-conf-path=./testdata/sdk_config_pwk.yml \
--org-id=wx-org1.chainmaker.org \
--admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
--node-org-id=wx-org5.chainmaker.org
```
- 更新共识节点Org
```shell
./cmc client chainconfig consensusnodeorg update \
--sdk-conf-path=./testdata/sdk_config_pwk.yml \
--org-id=wx-org1.chainmaker.org \
--admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
--node-ids=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4,QmaWrR72CbT51nFVpNDS8NaqUZjVuD4Ezf8xcHcFW9SJWF \
--node-org-id=wx-org5.chainmaker.org
```

### 管理链的节点
组织的节点账户生成，参考[生成节点账户](搭建pwk模式账户体系.html#生成节点账户)
#### 共识节点
- 增加共识节点
```shell
./cmc client chainconfig consensusnodeid add \
--sdk-conf-path=./testdata/sdk_config_pwk.yml \
--org-id=wx-org1.chainmaker.org \
--admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
--node-id=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4 \
--node-org-id=wx-org5.chainmaker.org
```

- 更新共识节点Id
```shell
./cmc client chainconfig consensusnodeid update \
--sdk-conf-path=./testdata/sdk_config_pwk.yml \
--org-id=wx-org1.chainmaker.org \
--admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
--node-id=QmXxeLkNTcvySPKMkv3FUqQgpVZ3t85KMo5E4cmcmrexrC \
--node-id-old=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4 \
--node-org-id=wx-org5.chainmaker.org
```

- 删除共识节点
```shell
./cmc client chainconfig consensusnodeid remove \
--sdk-conf-path=./testdata/sdk_config_pwk.yml \
--org-id=wx-org1.chainmaker.org \
--admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
--node-id=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4 \
--node-org-id=wx-org5.chainmaker.org
```

- 查询共识节点  
  共识节点可以通过链配置查询，在`consensus`字段下会返回当前区块链网络中的共识节点列表，命令如下：
```shell
./cmc client chainconfig query \
--sdk-conf-path=./testdata/sdk_config_pwk.yml

#...
  "consensus": {
    "nodes": [
      {
        "node_id": [
          "QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4"
        ],
        "org_id": "wx-org1.chainmaker.org"
      },
      {
        "node_id": [
          "QmVviBSVY4xK2161hFWh2v4Wh5ThGAgBiPtXV8XjzVbzPW"
        ],
        "org_id": "wx-org2.chainmaker.org"
      },
      {
        "node_id": [
          "QmbBhed1jeFMkFazYnvVJiqp9RAxnnjA6wxiPVkgAdbeDT"
        ],
        "org_id": "wx-org3.chainmaker.org"
      },
      {
        "node_id": [
          "QmdyLwr6ahCQeSixfw72E17rqn8s4vLewzJgYR64eEeQvD"
        ],
        "org_id": "wx-org4.chainmaker.org"
      }
    ],
    "type": 1
  },
```

#### 同步节点管理
- 增加同步节点

缺该类文案。

- 更新同步节点Id

缺该类文案。

- 删除同步节点

缺该类文案。

- 查询同步节点

缺该类文案。



### 管理链的用户
#### 管理链的管理员账户
- 添加管理员
```shell
# 生成管理员私钥
 ./cmc key gen -a ECC_P256 -p ./ -n admin.key

# 导出管理员公钥
 ./cmc key export_pub -k admin.key -n admin.pem
 
./cmc client chainconfig trustroot add \
--sdk-conf-path=./testdata/sdk_config_pwk.yml \
--org-id=wx-org1.chainmaker.org \
--admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
--trust-root-org-id=wx-org5.chainmaker.org \
--trust-root-path=./admin.pem
```
- 删除管理员
```shell
./cmc client chainconfig trustroot remove \
--sdk-conf-path=./testdata/sdk_config_pwk.yml \
--org-id=wx-org1.chainmaker.org \
--admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
--trust-root-org-id=wx-org5.chainmaker.org \
--trust-root-path=./admin.pem
```

- 更新管理员
```shell
./cmc client chainconfig trustroot update \
--sdk-conf-path=./testdata/sdk_config_pwk.yml \
--org-id=wx-org1.chainmaker.org \
--admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
--trust-root-org-id=wx-org5.chainmaker.org \
--trust-root-path=./admin.pem
```

- 查询管理员列表
```shell
 ./cmc client chainconfig query \
--sdk-conf-path=./testdata/sdk_config_pwk.yml 
[
  {
    "org_id": "wx-org1.chainmaker.org",
    "root": [
      "-----BEGIN PUBLIC KEY-----\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEzR4qsOr2+cwcLlU4klgXB8DugCJZ\n+XrJDzIisQeA2gmldqY3PSsF4adMSUYN+ux+EnoAB6BpejQqj0IeQ6RJ9g==\n-----END PUBLIC KEY-----\n"
    ]
  },
  {
    "org_id": "wx-org2.chainmaker.org",
    "root": [
      "-----BEGIN PUBLIC KEY-----\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEjgS2zVaZkds9dQb/ZN535lFAlmgG\nCw5Z4NITw/AOeo01zyMJNUzoQMzjkmBbvuF3rAszkVeAXIkT3eCfGLdR2Q==\n-----END PUBLIC KEY-----\n"
    ]
  },
  {
    "org_id": "wx-org3.chainmaker.org",
    "root": [
      "-----BEGIN PUBLIC KEY-----\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE4Xr9k3xGxEVAXlrKQyqLjoJoWyTe\nEQOQ9h4KJw/3ua4UhWgGOkrd5QCBgtdBh6gthmMfdUsFOTrnMPya/CSdZw==\n-----END PUBLIC KEY-----\n"
    ]
  },
  {
    "org_id": "wx-org4.chainmaker.org",
    "root": [
      "-----BEGIN PUBLIC KEY-----\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEw+mHR4CwRr/m0tzpBrX6+FXavFtw\nPqWqgl7t3hYe7VWqA74vEmnj+K0jpEfXgsvGBMfQt3mLZWNlzAzujy9Y4w==\n-----END PUBLIC KEY-----\n"
    ]
  },
]
```

#### 管理链的普通账户
在长安链pwk模式下，普通账户需要管理员注册后，才具有访问链的权限。
- 注册账户
```shell
# 生成普通账户私钥
 ./cmc key gen -a ECC_P256 -p ./ -n user.key

# 导出账户公钥
 ./cmc key export_pub -k user.key -n user.pem
 
 # 管理员注册新账户
./cmc pubkey add \
--sdk-conf-path=./testdata/sdk_config_pwk.yml \
--chain-id=chain1 \
--admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
--key-org-id=wx-org1.chainmaker.org \
--role=client \
--pubkey-file-path=./user.pem #使用公钥注册
```

- 删除账户
```shell
./cmc pubkey del \
--sdk-conf-path=./testdata/sdk_config_pwk.yml \
--chain-id=chain1 \
--admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key
--key-org-id=wx-org1.chainmaker.org \
--pubkey-file-path=./user.pem
```

- 账户查询

```shell
# 查询注册结果
./cmc pubkey query \
--sdk-conf-path=./testdata/sdk_config_pwk.yml \
--chain-id=chain1 \
--admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
--pubkey-file-path=./user.pem

==> "org_id = wx-org1.chainmaker.org, role = CLIENT"
```

### 权限管理
长安链目前支持资源级别的权限管理，可以通过cmc命令行工具或者sdk来查询、新增、修改以及删除资源权限。
- 权限定义介绍请参考[权限定义](长安链账户整体介绍.html#权限定义)
- 资源定义介绍请参考[资源定义](长安链账户整体介绍.html#资源定义)

#### 权限列表查询
```shell
  ./cmc client chainconfig permission list \
  --sdk-conf-path=./testdata/sdk_config_pwk.yml
```

pwk模式下默认权限列表，请参考[pwk模式权限定义](../tech/身份权限管理.html#PermissionedWithKey)。

#### 权限列表修改
- 新增权限
  权限修改相关的操作一般需要**多数管理员多签**授权，假如我们有个资源名叫：TEST_SUM, 需要设置为"任一用户可以访问"， 使用cmc命令设置权限权限如下：
```shell
  ./cmc client chainconfig permission add \
  --sdk-conf-path=./testdata/sdk_config_pwk.yml \
  --admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
  --admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
  --sync-result=true \
  --permission-resource-name="TEST_SUM" \
  --permission-resource-policy-rule=ANY \
  --permission-resource-policy-roleList=CLIENT
```
- 修改权限   
  使用cmc命令修改TEST_SUM资源权限为多数管理员多签操作
```shell
./cmc client chainconfig permission update \
 --sdk-conf-path=./testdata/sdk_config_pwk.yml \
  --admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
  --admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
  --sync-result=true \
  --permission-resource-name="TEST_SUM" \
  --permission-resource-policy-rule=MAJORITY \
  --permission-resource-policy-roleList=ADMIN
```

- 删除权限    
  使用cmc命令删除TEST_SUM资源的权限限制，删除后节点权限校验模块不会对该资源进行权限检查
```shell
./cmc client chainconfig permission delete \
 --sdk-conf-path=./testdata/sdk_config_pwk.yml \
  --admin-org-ids=wx-org1.chainmaker.org,wx-org2.chainmaker.org,wx-org3.chainmaker.org \
  --admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org2.chainmaker.org/admin/admin.key,./testdata/crypto-config/wx-org3.chainmaker.org/admin/admin.key \
  --sync-result=true \
  --permission-resource-name="TEST_SUM"
```

注：除了自定义资源的权限设置外，长安链也支持默认权限的修改, 但无法删除默认权限， 与证书模式类似。



