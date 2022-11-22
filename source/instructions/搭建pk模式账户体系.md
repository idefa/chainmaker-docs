# 搭建PK模式账户体系
作者：长安链团队 韩学洋

## 使用cmc搭建长安链网络
长安链支持PermissionWithCert、PermissionWithKey、Public等三种不同账户模式的链，本章节我们将详细介绍如何通过cmc命令行工具搭建完整的长安链Public模式的账户体系，以及如果管理该模式下的链账户，包括节点账户、用户账户的增删等。

本文示例说明：
- 创建新链：4节点，4管理员
- 链管理：节点管理、用户管理

### 部署cmc命令行工具

长安链CMC工具可用于生成公私钥，部署Public模式的长安链前，我们需要通过长安链CMC工具生成相关的公私钥文件。
长安链命令行工具cmc安装, 请参考[长安链命令行工具](../dev/命令行工具pk.html#编译&配置)

### 基于cmc工具生成公钥账户

由于pk模式无证书概念，因此对于节点、管理员以及用户而言，都是公私钥的形式。 节点账户的地址需要添加到共识列表，管理员账户需要添加到管理员列表；
而普通用户账户，可以直接调用管理员部署的合约，无需注册。

#### 生成节点账户
```shell
  # 生成节点账户私钥
$ ./cmc key gen -a ECC_P256 -p ./ -n consensus1.key
 
  # 导出节点账户公钥
$ ./cmc key export_pub -k consensus1.key -n consensus1.pem
 
 # 计算节点账户地址
$ ./cmc cert nid --node-pk-path=./consensus1.pem
node id : QmeqnZEgGeQYyc4qX92XV3SxafqRJqCQ9388jWp2N1oA93
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

重复以上步骤分别生成4个节点账户和4个管理员账户。


### 基于生成的节点账户和管理员账户创建链
获取的节点账户和管理员账户，需要在启动链时，将他们配置到链配置文件`bc1.yml`的`trust_roots`里，并将`bc1.yml`和`chainmaker.yml`中的`nodes.node_id`替换为以上获取的节点nodeId。

**配置文件修改位置如下**

- bc1.yml（链配置文件）

```yaml
#共识配置
consensus:
  # Consensus type: 1-TBFT,5-DPOS
  type: 1
  nodes:
    - org_id: "public"
      node_id:
        - "QmeqnZEgGeQYyc4qX92XV3SxafqRJqCQ9388jWp2N1oA93"
        - "QmSXhWkujKh2PEN5tFuiTBnSbkx7vN6P9zPoa6RViLdpdA"
        - "QmfR4jNLsBK3FedeCLyTmzaCeaECK3VjDRoY6XcjiUpWYJ"
        - "QmRKZyDH89CH3zJaSC5VHn9tgcBbs7jc1mDVd9QgyzmKan"
      
trust_roots:
  - org_id: "public"
    root:
      - "../config/node1/admin/admin1/admin1.pem"
      - "../config/node1/admin/admin2/admin2.pem"
      - "../config/node1/admin/admin3/admin3.pem"
      - "../config/node1/admin/admin4/admin4.pem"
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
    - "/ip4/127.0.0.1/tcp/11301/p2p/QmeqnZEgGeQYyc4qX92XV3SxafqRJqCQ9388jWp2N1oA93"
    - "/ip4/127.0.0.1/tcp/11302/p2p/QmSXhWkujKh2PEN5tFuiTBnSbkx7vN6P9zPoa6RViLdpdA"
    - "/ip4/127.0.0.1/tcp/11303/p2p/QmfR4jNLsBK3FedeCLyTmzaCeaECK3VjDRoY6XcjiUpWYJ"
    - "/ip4/127.0.0.1/tcp/11304/p2p/QmRKZyDH89CH3zJaSC5VHn9tgcBbs7jc1mDVd9QgyzmKan"

```


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
│   └── node1
├── lib
│   ├── libwasmer.dylib
│   └── wxdec
└── log
```

在启动节点之前，需要将以上部署过程中用到的各个节点公私钥、管理员公私钥替换为以上用cmc生成的公私钥对。
同时各个节点部署目录下的`bc.yml`和`chainmaker.yml`配置文件按照以上说明进行修改。

**配置更新说明**
```shell
./config
└── node1
    ├── admin
    │   ├── admin1 #optional
    │   │   ├── admin1.key 
    │   │   └── admin1.pem
    │   ├── admin2
    │   │   ├── admin2.key
    │   │   └── admin2.pem
    │   ├── admin3
    │   │   ├── admin3.key
    │   │   └── admin3.pem
    │   ├── admin4
    │   │   ├── admin4.key
    │   │   └── admin4.pem
    ├── chainconfig
    │   └── bc1.yml # 更新后的链配置文件，参考上节说明
    ├── chainmaker.yml # 更新后的链配置文件，参考上节说明
    ├── log.yml
    ├── node1.key #节点1密钥
    ├── node1.nodeid #节点1的nodeid
    ├── node1.pem #节点1的公钥
    └── user #optional
        └── client1
            ├── client1.addr
            ├── client1.key
            └── client1.pem


```


**节点部署和链启动**

启动pk模式的链与启动证书模式的链使用的方式类似，可以参考证书模式。在密钥生成步骤使用`./prepare_pk.sh`脚本即可。

- 使用命令行方式部署链，参考[通过命令行启动pk模式的链](../instructions/启动pk模式的链.html#通过命令行启动pk模式的链)
- 
自行将上述两文章内的公私钥文件替换成上述流程新生成的公私钥文件，然后按步骤执行，直至链启动成功。

按照以上方式启动org1、org2、org3和org4下的共识节点，等到所有节点建立连接，表明区块链网络部署成功，并可以对外提供区块链服务。  
可通过遗下命令查看节点日志，若看到all necessary peers connected则表示节点已经准备就绪。

```shell
[INFO]  [Net]   libp2pnet/libp2p_connection_supervisor.go:116   [ConnSupervisor] all necessary peers connected.
```


### 部署/调用智能合约
链部署后，在进行部署/调用合约测试，以验证链是否正常运行，目前支持三种方式进行合约的部署和调用。示例合约可从[此处获得](../instructions/部署示例合约.md)
#### 使用CMC工具测试
使用长安链命令行工具cmc进行测试，详细教程请见命令行[交易功能](../dev/命令行工具pk.html#交易功能)
其中，cmc工具配置文件sdk_config_pk.yaml需要进行相应修改，主要修改以下配置:
- 客户端私钥：user_sign_key_file_path

```yaml
chain_client:
  # 链ID
  chain_id: "chain1"

  # 客户端用户交易签名私钥路径
  user_sign_key_file_path: "./testdata/crypto-config/node1/admin/admin1/admin1.key" # 管理员1私钥

  # 签名使用的哈希算法，和节点保持一直
  crypto:
    hash: SHA256
  auth_type: public
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
    max_send_message_size: 16 # grpc客户端发送消息时，允许单条message大小的最大值(MB)
```

注：由于我们使用管理员账户进行测试，所以user_sign_key_file_path需要指定管理员私钥

#### 使用长安链管理台进行测试
- 通过长安链管理台进行部署/调用，详细教程请见[长安链管理台订阅外部链，以及部署合约章节](../dev/长安链管理台.md)
  - 需要注意，通过长安链管理台订阅上述创建的链时，需要确保链与管理台间网络通畅。

#### 使用长安链SDK进行测试
- 通过长安链SDK进行部署/调用，详情[SDK使用说明章节](../instructions/GoSDK使用说明.md)
  - 需要将SDK所需的相关证书替换成，上文所生成的证书，然后进行操作。


## 链管理

### 管理链的节点
节点账户生成，参考[生成节点账户](搭建pk模式账户体系.html#生成节点账户)
#### 共识节点管理
- 增加共识节点
```shell
./cmc client chainconfig consensusnodeid add \
--sdk-conf-path=./testdata/sdk_config_pk.yml \
--user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
--admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
--node-id=QmeqnZEgGeQYyc4qX92XV3SxafqRJqCQ9388jWp2N1oA93 \
--node-org-id=public
```

- 更新共识节点Id
```shell
./cmc client chainconfig consensusnodeid update \
--sdk-conf-path=./testdata/sdk_config_pk.yml \
--user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
--admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
--node-id=QmXxeLkNTcvySPKMkv3FUqQgpVZ3t85KMo5E4cmcmrexrC \
--node-id-old=QmeqnZEgGeQYyc4qX92XV3SxafqRJqCQ9388jWp2N1oA93 \
--node-org-id=public
```

- 删除共识节点
```shell
./cmc client chainconfig consensusnodeid remove \
--sdk-conf-path=./testdata/sdk_config_pk.yml \
--user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
--admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
--node-id=QmeqnZEgGeQYyc4qX92XV3SxafqRJqCQ9388jWp2N1oA93 \
--node-org-id=public
```

- 查询共识节点  
  共识节点可以通过链配置查询，在`consensus`字段下会返回当前区块链网络中的共识节点列表，命令如下：
```shell
./cmc client chainconfig query \
--sdk-conf-path=./testdata/sdk_config_pk.yml

  "consensus": {
    "nodes": [
      {
        "node_id": [
          "QmeqnZEgGeQYyc4qX92XV3SxafqRJqCQ9388jWp2N1oA93",
          "QmSXhWkujKh2PEN5tFuiTBnSbkx7vN6P9zPoa6RViLdpdA",
          "QmfR4jNLsBK3FedeCLyTmzaCeaECK3VjDRoY6XcjiUpWYJ",
          "QmRKZyDH89CH3zJaSC5VHn9tgcBbs7jc1mDVd9QgyzmKan"
        ],
        "org_id": "public"
      }
    ],
    "type": 1
  },
```

#### 同步节点管理
- 添加同步节点  
同步节点账户生成，参考[生成节点账户](搭建pk模式账户体系.html#生成节点账户)， 在pk模式下，同步节点可以自由加入和退出网络，并同步账本，但不参与共识。

- 更新同步节点  
暂不支持同步节点更新

- 删除同步节点  
暂不支持同步节点删除

- 查询同步节点  
暂不支持同步节点删除

### 管理链的用户
#### 管理链的管理员账户
目前公钥模式只支持批量更新管理员，新管理员列表会替换旧管理员列表，请谨慎操作。
```shell
# 生成管理员私钥
 ./cmc key gen -a ECC_P256 -p ./ -n admin.key

# 导出管理员公钥
 ./cmc key export_pub -k admin.key -n admin.pem
 
 
./cmc client chainconfig trustroot update \
--sdk-conf-path=./testdata/sdk_config_pk.yml \
--user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
--admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
--trust-root-org-id=public \
--trust-root-path=./admin.pem,./testdata/crypto-config/node1/admin/admin1/admin1.pem,./testdata/crypto-config/node1/admin/admin2/admin2.pem,./testdata/crypto-config/node1/admin/admin3/admin3.pem

- 查询管理员列表
```shell
 ./cmc client chainconfig query \
--sdk-conf-path=./testdata/sdk_config_pk.yml 

 "trust_roots": [
    {
      "org_id": "public",
      "root": [
        "-----BEGIN PUBLIC KEY-----\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEi95yJxLXrKEeBi5ZJqjk2lEFMKfM\n4pydPq78oTbnHQgQc47eTUENVxBIAEI/mAKjsK82i32amXG0Q9dyqZUWRw==\n-----END PUBLIC KEY-----\n",
        "-----BEGIN PUBLIC KEY-----\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEfVzW4O+RjSi+0mPl7HE80LfDup+E\n3s1mziNwP/d5r6X5D5pSdtcGhR80+9rOnIaayM2Eb61m147K72HmgH0I5A==\n-----END PUBLIC KEY-----\n",
        "-----BEGIN PUBLIC KEY-----\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE3cFf3ISXtD+vyc6LjuohlHX8A4yG\nIHlMpbwB+H1411TYCgutRoyjXbUy9kcJrXySLS7UCb+/c/yNZ+tz0a6dmA==\n-----END PUBLIC KEY-----\n",
        "-----BEGIN PUBLIC KEY-----\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEPuPoOk1otXmNnY0nF0B/eBhIMhEC\niE19OneK0AA4nsk6lsef5PoOG8rI5EljCNIxJQ4pOthTMhX6B0gVjlWoZw==\n-----END PUBLIC KEY-----\n"
      ]
    }
  ],

```

#### 管理链的普通账户
在长安链pk模式下，普通账户无需注册，可直接调用合约。
- 生成普通账户
```shell
# 生成普通用户私钥user.key
 ./cmc key gen -a ECC_P256 -p ./ -n user.key
```

- 普通账户调用合约
```shell
# 管理员创建合约
./cmc client contract user create \
--contract-name=fact \
--runtime-type=WASMER \
--byte-code-path=./testdata/claim-wasm-demo/rust-fact-2.0.0.wasm \
--version=1.0 \
--sdk-conf-path=./testdata/sdk_config_pk.yml \
--admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key \
--sync-result=true \
--params="{}"

# 使用新账户user.key发起合约调用
./cmc client contract user invoke \
--contract-name=fact \
--method=save \
--sdk-conf-path=./testdata/sdk_config_pk.yml \
--params="{\"file_name\":\"name007\",\"file_hash\":\"ab3456df5799b87c77e7f88\",\"time\":\"6543234\"}" \
--sync-result=true \
--user-signkey-file-path=./user.key
{
  "contract_result": {
    "contract_event": [
      {
        "contract_name": "fact",
        "contract_version": "1.0",
        "event_data": [
          "ab3456df5799b87c77e7f88",
          "name007",
          "6543234"
        ],
        "topic": "topic_vx",
        "tx_id": "17032ccc06c11470ca52fdfc07218265dc3cc24a84bc4252941f9096227ee0f3"
      }
    ],
    "gas_used": 237929
  },
  "tx_id": "17032ccc06c11470ca52fdfc07218265dc3cc24a84bc4252941f9096227ee0f3"
}
```

### 权限管理

- 权限定义介绍请参考[权限定义](长安链账户整体介绍.html#权限定义)  
- 资源定义介绍请参[资源定义](长安链账户整体介绍.html#资源定义)

#### 权限列表查询
```shell
  ./cmc client chainconfig permission list \
  --sdk-conf-path=./testdata/sdk_config_pk.yml
```

pk模式下默认权限列表，请参考[pk模式权限定义](../tech/身份权限管理.html#Public)。

#### 权限列表修改
pk模式暂不支持权限修改

