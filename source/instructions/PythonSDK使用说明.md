#  Python SDK 使用说明
作者：长安链团队 韩志超

## 概述
目前Python SDK 除了**不支持隐私计算API**、**Hibe**及**国密SSL通信**外，支持python SDK所有接口，
主要概念包括：

- ChainClient: 链客户端对象
- User: 客户端用户对象
- Node: 客户端连接节点对象

几乎所有操作需要使用ChainClient对象完成，同时utils提供了文件、证书、EVM等一些实用方法。

## 环境准备

### 环境依赖

**Python**

版本为Python3.8.0以上

下载地址：https://www.python.org/downloads/

若已安装，请通过命令查看版本：

```bash
$ python3 --version
Python 3.8.2
```

### 安装方式
1. 使用pip命令，在线安装(需要安装Git)
```bash
$ pip3 install git+https://git.chainmaker.org.cn/chainmaker/sdk-python.git
```
2. 克隆或下载[sdk-python](https://git.chainmaker.org.cn/chainmaker/sdk-python)源码，在项目根目录下运行
```bash
$ python3 setup.py install
```

## 怎么使用SDK

### 示例代码

#### 创建节点

> 注️：需要拷贝目标链chainmaker-go/build/crypto-config到脚本所在目录

```python
from chainmaker.node import Node

# 创建节点
node = Node(
    node_addr='127.0.0.1:12301',
    conn_cnt=10,
    enable_tls=True,
    cas=['./testdata/crypto-config/wx-org1.chainmaker.org/ca', './testdata/crypto-config/wx-org2.chainmaker.org/ca'],
    tls_host_name='chainmaker.org'
)
```

#### 以参数形式创建ChainClient

> 更多内容请参考：`tests/test_chain_client.py`
>
> 注：示例中证书采用路径方式去设置，也可以使用证书内容去设置，具体请参考`createClientWithCaCerts`方法

```python
from chainmaker.chain_client import ChainClient
from chainmaker.node import Node
from chainmaker.user import User
from chainmaker.utils import file_utils
    
user = User('wx-org1.chainmaker.org',
            sign_key_bytes=file_utils.read_file_bytes('./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key'),
            sign_cert_bytes=file_utils.read_file_bytes('./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt'),
            tls_key_bytes=file_utils.read_file_bytes('./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key'),
            tls_cert_bytes=file_utils.read_file_bytes('./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt')
            )

node = Node(
    node_addr='127.0.0.1:12301',
    conn_cnt=1,
    enable_tls=True,
    trust_cas=[
        file_utils.read_file_bytes('./testdata/crypto-config/wx-org1.chainmaker.org/ca/ca.crt'),
        file_utils.read_file_bytes('./testdata/crypto-config/wx-org2.chainmaker.org/ca/ca.crt')
    ],
    tls_host_name='chainmaker.org'
)

cc = ChainClient(chain_id='chain1', user=user, nodes=[node])
print(cc.get_chainmaker_server_version())
```

#### 以配置文件形式创建ChainClient

> 注：参数形式和配置文件形式两个可以同时使用，同时配置时，以参数传入为准

```python
from chainmaker.chain_client import ChainClient

# ./testdata/sdk_config.yml 中私钥/证书等如果使用相对路径应相对于当前运行起始目录
cc = ChainClient.from_conf('./testdata/sdk_config.yml')
```

> [配置文件 sdk_config.yml 格式参考](../recovery/配置文件一览.html#sdk-config-yml)

#### 创建合约

```python
from google.protobuf import json_format
from chainmaker.chain_client import ChainClient
from chainmaker.utils.evm_utils import calc_evm_contract_name
from chainmaker.keys import RuntimeType

endorsers_config = [{'org_id': 'wx-org1.chainmaker.org',
                  'user_sign_crt_file_path': './testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.sign.crt',
                  'user_sign_key_file_path': './testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.sign.key'},
                  {'org_id': 'wx-org2.chainmaker.org',
                  'user_sign_crt_file_path': './testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.sign.crt',
                  'user_sign_key_file_path': './testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.sign.key'},
                  {'org_id': 'wx-org3.chainmaker.org',
                  'user_sign_crt_file_path': './testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.sign.crt',
                  'user_sign_key_file_path': './testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.sign.key'},
                  ]

cc = ChainClient.from_conf('./testdata/sdk_config.yml')

def create_contract(contract_name: str, version: str, byte_code_path: str, runtime_type: RuntimeType, params: dict = None, 
                    with_sync_result=True) -> dict:
    """创建合约"""
    # 创建请求payload
    payload = cc.create_contract_create_payload(contract_name, version, byte_code_path, runtime_type, params)
    # 创建背书
    endorsers = cc.create_endorsers(payload, endorsers_config)
    # 携带背书发送请求
    res = cc.send_request_with_sync_result(payload, with_sync_result=with_sync_result, endorsers=endorsers)
    # 交易响应结构体转为字典格式
    return json_format.MessageToDict(res)

# 创建WASM合约，本地合约文件./testdata/claim-wasm-demo/rust-fact-2.0.0.wasm应存在
result1 = create_contract('fact', '1.0', './testdata/claim-wasm-demo/rust-fact-2.0.0.wasm', RuntimeType.WASMER, {})
print(result1)

# 创建EVM合约，本地合约文件./testdata/balance-evm-demo/ledger_balance.bin应存在

contract_name = calc_evm_contract_name('balance001')
result2 = create_contract(contract_name, '1.0', './testdata/balance-evm-demo/ledger_balance.bin', RuntimeType.EVM)
print(result2)
```

#### 调用合约

```python
from google.protobuf import json_format
from chainmaker.chain_client import ChainClient
from chainmaker.utils.evm_utils import calc_evm_contract_name, calc_evm_method_params

# 创建客户端
cc = ChainClient.from_conf('./testdata/sdk_config.yml')

# 调用WASM合约
res1 = cc.invoke_contract('fact', 'save', {"file_name":"name007","file_hash":"ab3456df5799b87c77e7f88","time":"6543234"},
                          with_sync_result=True)
# 交易响应结构体转为字典格式
print(json_format.MessageToDict(res1))

# 调用EVM合约
evm_contract_name = calc_evm_contract_name('balance001')
evm_method, evm_params = calc_evm_method_params('updateBalance', [{"uint256": "10000"}, {"address": "0xa166c92f4c8118905ad984919dc683a7bdb295c1"}])
res2 = cc.invoke_contract(evm_contract_name, evm_method, evm_params, with_sync_result=True)
# 交易响应结构体转为字典格式
print(json_format.MessageToDict(res2))
```

#### 更多示例和用法

> 更多示例和用法，请参考单元测试用例

| 功能     | 单测代码                      |
| -------- | ----------------------------- |
| 用户合约 | `tests/test_user_contract.py`   |
| 系统合约 | `tests/test_system_contract.py` |
| 链配置   | `tests/test_chain_config.py`    |
| 证书管理 | `tests/test_cert_manage.py`     |
| 消息订阅 | `tests/test_user_contract.py`   |

## 接口说明
请参看：[《chainmaker-python-sdk》](chainmaker-python-sdk.md)

### 1 用户合约接口

#### 1.1 创建合约待签名payload生成
```python
def create_contract_create_payload(self, contract_name: str, version: str, byte_code_or_file_path: str,
                                       runtime_type: RuntimeType,
                                   params: dict) -> Payload:
    """生成 创建合约的payload
    :param contract_name: 合约名
    :param version: 合约版本
    :param byte_code_or_file_path: 合约字节码：可以是字节码；合约文件路径；或者 hex编码字符串；或者 base64编码字符串。
    :param runtime_type: contract_pb2.RuntimeType.WASMER
    :param params: 合约参数，dict类型，key 和 value 尽量为字符串
    :return: Payload
    :raises ValueError: 如果 byte_code 不能转成合约字节码
    """
```


#### 1.2 升级合约待签名payload生成

```python
def create_contract_upgrade_payload(self, contract_name: str, version: str, byte_code_or_file_path: str,
                                    runtime_type: RuntimeType,
                                    params: dict) -> Payload:
    """生成 升级合约的payload
    :param contract_name: 合约名
    :param version: 合约版本
    :param byte_code_or_file_path: 合约字节码：可以是字节码；合约文件路径；或者 hex编码字符串；或者 base64编码字符串。
    :param runtime_type: contract_pb2.RuntimeType.WASMER
        eg. 'INVALID', 'NATIVE', 'WASMER', 'WXVM', 'GASM', 'EVM', 'DOCKER_python', 'DOCKER_JAVA'
    :param params: 合约参数，dict类型，key 和 value 尽量为字符串
    :return: Payload
    :raises ValueError: 如果 byte_code 不能转成合约字节码
    """

```

#### 1.3 冻结合约payload生成
```python
def create_contract_freeze_payload(self, contract_name: str) -> Payload:
    """生成 冻结合约的payload
    :param contract_name: 合约名
    :return: Payload
    """
```

#### 1.4 解冻合约payload生成
```python
def create_contract_unfreeze_payload(self, contract_name: str) -> Payload:
    """生成 解冻合约的payload
    :param contract_name: 合约名
    :return: Payload
    """
```

#### 1.5 吊销合约payload生成

```python
def create_contract_revoke_payload(self, contract_name: str) -> Payload:
    """生成 吊销合约的payload
    :param contract_name: 合约名
    :return: Payload
    """
```

#### 1.6 合约管理获取Payload签名

```python
def sign_contract_manage_payload(self, payload: Payload) -> bytes:
    """对 合约管理的 payload 字节数组进行签名，返回签名后的payload字节数组
    :param payload: 交易 payload
    :return: Payload 的字节数组
    :raises DecodeError: 如果 byte_code 解码失败
    """
```

#### 1.7 发送合约管理请求（创建、更新、冻结、解冻、吊销）

```python
def send_contract_manage_request(self, payload: Payload, endorsers: List[EndorsementEntry], timeout: int = None,
                                 with_sync_result: bool = None) -> TxResponse:
    """发送合约管理的请求
    :param endorsers: 背书列表
    :param payload: 请求的 payload
    :param timeout: 超时时间
    :param with_sync_result: 是否同步交易执行结果。如果不同步，返回tx_id，供异步查询; 同步则循环等待，返回交易的执行结果。
    :return: TxResponse
    :raises RequestError: 请求失败
    """
```

#### 1.8 合约调用

```python
def invoke_contract(self, contract_name: str, method: str, params: dict = None, tx_id: str = "",
                    timeout: int = None,
                    with_sync_result: bool = None) -> TxResponse:
    """调用 用户合约 接口
    :param contract_name: 合约名
    :param method: 调用合约方法名
    :param params: 调用参数，参数类型为dict
    :param tx_id: 交易id，如果交易id为空/空字符串，则创建新的tx_id
    :param timeout: 超时时间，默认为 3s
    :param with_sync_result: 是否同步交易执行结果。如果不同步，返回tx_id，供异步查询; 同步则循环等待，返回交易的执行结果。
    :return: TxResponse
    :raises RequestError: 请求失败
    """
```

#### 1.9 合约查询接口调用

```python
def query_contract(self, contract_name: str, method: str, params: Union[dict, list] = None,
                   timeout: int = None) -> TxResponse:
    """查询 用户合约 接口
    :param contract_name: 合约名
    :param method: 调用合约方法名
    :param params: 调用参数，参数类型为dict
    :param timeout: 超时时间，默认为 3s
    :return: TxResponse
    :raises RequestError: 请求失败
    """
```

#### 1.10 构造待发送交易体

```python
def get_tx_request(self, contract_name: str, method: str, params: Union[dict, list] = None,
                   tx_id: str = "") -> TxRequest:
    """
    获取交易请求体
    :param contract_name: 合约名
    :param method: 调用合约方法名
    :param params: 调用参数，参数类型为dict
    :param tx_id: 交易id，如果交易id为空/空字符串，则创建新的tx_id
    :return: Request
    """
```

#### 1.11 发送已构造好的交易体

```python
def send_tx_request(self, tx_request, timeout=None, with_sync_result=None) -> TxResponse:
    """发送请求
    :param tx_request: 请求体
    :param timeout: 超时时间
    :param with_sync_result: 是否同步交易执行结果。如果不同步，返回tx_id，供异步查询; 同步则循环等待，返回交易的执行结果。
    :return: Response
    :raises RequestError: 请求失败
    """
```

### 2 系统合约接口

#### 2.1 根据交易Id查询交易

```python
def get_tx_by_tx_id(self, tx_id: str) -> TransactionInfo:
    """根据交易ID获取带读写集交易详情
    :param tx_id: 交易ID，类型为字符串
    :return: Result
    :raises RequestError: 请求失败
    """
```

#### 2.2 根据区块高度查询区块

```python
def get_block_by_height(self, block_height: int, with_rw_set: bool = False) -> BlockInfo:
    """根据区块高度查询区块详情
    :param block_height: 区块高度
    :param with_rw_set: 是否返回读写集数据, 默认不返回。
    :return: 区块信息BlockInfo对象
    :raises RequestError: 请求失败，块已归档是抛出ContractFile
    """
```

#### 2.3 根据区块高度查询完整区块


```python
def get_full_block_by_height(self, block_height: int) -> BlockWithRWSet:
    """根据区块高度查询区块所有数据
    :param block_height: 区块高度
    :return: BlockInfo
    :raises RequestError: 请求失败
    """
```

#### 2.4 根据区块哈希查询区块


```python
def get_block_by_hash(self, block_hash: str, with_rw_set: bool = False) -> BlockInfo:
    """根据区块 Hash 查询区块详情
    :param block_hash: 区块Hash, 二进制hash.hex()值，
                       如果拿到的block_hash字符串是base64值, 需要用 base64.b64decode(block_hash).hex()
    :param with_rw_set: 是否返回读写集数据
    :return: BlockInfo
    :raises RequestError: 请求失败
    """
```

#### 2.5 根据交易Id查询区块

```python
def get_block_by_tx_id(self, tx_id: str, with_rw_set: bool = False) -> BlockInfo:
    """根据交易id 查询交易所在区块详情
    :param tx_id: 交易ID
    :param with_rw_set: 是否返回读写集数据
    :return: BlockInfo
    :raises RequestError: 请求失败
    """
```

#### 2.6 查询最新的配置块

```python
def get_last_config_block(self, with_rw_set: bool = False) -> BlockInfo:
    """查询最新的配置块
    :param with_rw_set: 是否返回读写集数据
    :return: BlockInfo
    :raises RequestError: 请求失败
    """
```

#### 2.7 查询最新区块

```python
def get_last_block(self, with_rw_set: bool = False) -> BlockInfo:
    """查询最新的块
    :param with_rw_set: 是否返回读写集数据
    :return: BlockInfo
    :raises RequestError: 请求失败
    """
```

#### 2.8 查询节点加入的链信息

```python
def get_node_chain_list(self) -> ChainList:
        """查询节点加入的链信息，返回chain id 清单
        :return: 链列表
        :raises RequestError: 请求失败
        """
```

#### 2.9 查询链信息

```python
def get_chain_info(self) -> ChainInfo:
    """查询链信息
    :return: ChainInfo
    :raises RequestError: 请求失败
    """
```

#### 2.10 根据交易Id获取区块高度

```python
def get_block_height_by_tx_id(self, tx_id: str) -> int:
    """根据交易ID查询区块高度
    :param tx_id: 交易ID
    :return: 区块高度
    :raises RequestError: 请求失败
    """
```

#### 2.11 根据区块Hash获取区块高度

```python
def get_block_height_by_hash(self, block_hash: str) -> int:
    """根据区块hash查询区块高度
    :param block_hash: 区块Hash 二进制hash.hex()值,
           如果拿到的block_hash字符串是base64值, 需要用 base64.b64decode(block_hash).hex()
    :return: 区块高度
    :raises RequestError: 请求失败
    """
```

#### 2.12 查询当前最新区块高度

```python
def get_current_block_height(self) -> int:
    """
    查询当前区块高度
    :return: 区块高度
    """
```

#### 2.13 根据区块高度查询区块头

```python
def get_block_header_by_height(self, block_height) -> BlockHeader:
    """
    根据高度获取区块头
    :param block_height: 区块高度
    :return: 区块头
    """
```

#### 2.14 系统合约调用

```python
def invoke_system_contract(self, contract_name: str, method: str, params: dict = None,
                           tx_id: str = "", timeout: int = None, with_sync_result=False) -> TxResponse:
    """
    调用系统合约
    :param contract_name: 系统合约名称
    :param method: 系统合约方法
    :param params: 系统合约方法所需的参数
    :param tx_id: 指定交易Id，默认为空是生成随机交易Id
    :param timeout: RPC请求超时时间
    :param with_sync_result: 是否同步轮询交易结果
    :return: 交易响应信息
    """
```

#### 2.15 系统合约查询接口调用

```python
def query_system_contract(self, contract_name: str, method: str, params: Union[dict, list] = None,
                          tx_id: str = "", timeout: int = None) -> TxResponse:
    """
    查询系统合约
    :param contract_name: 系统合约名称
    :param method: 系统合约方法
    :param params: 系统合约方法所需的参数
    :param tx_id: 指定交易Id，默认为空是生成随机交易Id
    :param timeout: RPC请求超时时间
    :return: 交易响应信息
    """
```

#### 2.16 根据交易Id获取Merkle路径

```python
def get_merkle_path_by_tx_id(self, tx_id: str) -> bytes:
    """
    根据交易ID获取Merkle树路径
    :param tx_id: 交易ID
    :return: Merkle树路径
    """
```

#### 2.17 开放系统合约

```python
def create_native_contract_access_grant_payload(self, grant_contract_list: List[str]) -> Payload:
    """
    生成系统合约授权访问待签名Payload
    :param List[str] grant_contract_list: 授予权限的访问系统合约名称列表 # TODO 确认 合约状态必须是FROZEN
    :return: 待签名Payload
    """

```

#### 2.18 弃用系统合约

```python
def create_native_contract_access_revoke_payload(self, revoke_contract_list: List[str]) -> Payload:
    """
    生成原生合约吊销授权访问待签名Payload
    :param revoke_contract_list: 吊销授予权限的访问合约列表
    :return: 待签名Payload
    """
```

#### 2.19 查询指定合约的信息，包括系统合约和用户合约

```python
def get_contract_info(self, contract_name: str) -> Union[Contract, None]:
    """
    获取合约信息
    :param contract_name: 用户合约名称
    :return: 合约存在则返回合约信息Contract对象，合约不存在抛出ContractFail
    :raise: RequestError: 请求出错
    :raise: AssertionError: 响应code不为0,检查响应时抛出断言失败
    :raise: 当数据不是JSON格式时，抛出json.decoder.JSONDecodeError
    """
```

#### 2.20 查询所有的合约名单，包括系统合约和用户合约

```python
def get_contract_list(self) -> List[Contract]:
    """
    获取合约列表
    :return: 合约Contract对象列表
    :raise: RequestError: 请求出错
    :raise: AssertionError: 响应code不为0,检查响应时抛出断言失败
    :raise: 当数据不是JSON格式时，抛出json.decoder.JSONDecodeError
    """
```

#### 2.21 查询已禁用的系统合约名单

```python
def get_disabled_native_contract_list(self) -> List[str]:
    """
    获取禁用系统合约名称列表
    :return: 禁用合约名称列表
    """
```

<span id="chainConfig"></span>

### 3 链配置接口

#### 3.1 查询最新链配置

```python
def get_chain_config(self) -> ChainConfig:
    """查询 chain config
    :return: ChainConfig
    :raises RequestError: 请求失败
    """
```

#### 3.2 根据指定区块高度查询最近链配置

```python
def get_chain_config_by_block_height(self, block_height: int) -> ChainConfig:
    """根据指定区块高度查询最近链配置
    如果当前区块就是配置块，直接返回当前区块的链配置
    :param block_height: 块高
    :return: ChainConfig
    :raises RequestError: 请求失败
    """
```

#### 3.3 查询最新链配置序号Sequence

- 用于链配置更新

```python
def get_chain_config_sequence(self) -> int:
    """查询最新链配置序号Sequence
    :return: 最新配置序号
    :raises RequestError: 请求失败
    """
```

#### 3.4 链配置更新获取Payload签名


```python
def sign_chain_config_payload(self, payload_bytes) -> Payload:
    """对链配置的payload 进行签名
    如果当前区块就是配置块，直接返回当前区块的链配置
    :param payload_bytes: payload.SerializeToString() 序列化后的payload bytes数据
    :return: 签名的背书
    :raises
    """
```

#### 3.5 发送链配置更新请求

```python
 def send_chain_config_update_request(self, payload: Payload, endorsers: list, timeout: int = None,
                                         with_sync_result: bool = True) -> TxResponse:
    """
    发送链配置更新请求
    :param payload: 待签名链配置更新请求Payload
    :param endorsers: 背书列表
    :param timeout: RPC请求超时时间
    :param with_sync_result: 是否同步轮交易结果
    :return: 交易响应信息
    """
```

#### 3.6 更新Core模块待签名payload生成

```python
def create_chain_config_core_update_payload(self, tx_scheduler_timeout: int = None,
                                            tx_scheduler_validate_timeout: int = None) -> Payload:
    """更新Core模块待签名payload生成
    :param tx_scheduler_timeout: 交易调度器从交易池拿到交易后, 进行调度的时间，其值范围为[0, 60]，若无需修改，请置为-1
    :param tx_scheduler_validate_timeout: 交易调度器从区块中拿到交易后, 进行验证的超时时间，其值范围为[0, 60]，若无需修改，请置为-1
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.7 更新Block模块待签名payload生成

```python
def create_chain_config_block_update_payload(self, tx_timestamp_verify: bool = None, tx_timeout: int = None,
                                                 block_tx_capacity: int = None, block_size: int = None,
                                                 block_interval: int = None, tx_parameter_size=None) -> Payload:
    """更新区块生成模块待签名payload生成
    :param tx_timestamp_verify: 是否需要开启交易时间戳校验
    :param tx_timeout: 交易时间戳的过期时间(秒)，其值范围为[600, +∞)（若无需修改，请置为-1）
    :param block_tx_capacity: 区块中最大交易数，其值范围为(0, +∞]（若无需修改，请置为-1）
    :param block_size: 区块最大限制，单位MB，其值范围为(0, +∞]（若无需修改，请置为-1）
    :param block_interval: 出块间隔，单位:ms，其值范围为[10, +∞]（若无需修改，请置为-1）
    :param tx_parameter_size: 交易参数大小
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.8 添加信任组织根证书待签名payload生成


```python
def create_chain_config_trust_root_add_payload(self, trust_root_org_id: str,
                                                   trust_root_crts: List[str]) -> Payload:
    """添加信任组织根证书待签名payload生成
    :param str trust_root_org_id: 组织Id  eg. 'wx-or5.chainmaker.org'
    :param List[str] trust_root_crts: 根证书文件内容列表
       eg. [open('./testdata/crypto-config/wx-org5.chainmaker.org/ca/ca.crt').read()]
    :return: Payload
    :raises RequestError: 无效参数
    """
```

#### 3.9 更新信任组织根证书待签名payload生成

```python
def create_chain_config_trust_root_update_payload(self, trust_root_org_id: str,
                                                      trust_root_crts: List[str]) -> Payload:
    """更新信任组织根证书待签名payload生成
    :param str trust_root_org_id: 组织Id
    :param List[str] trust_root_crts: 根证书内容列表
    :return: Payload
    :raises RequestError: 无效参数
    """
```

#### 3.10 删除信任组织根证书待签名payload生成

```python
def create_chain_config_trust_root_delete_payload(self, trust_root_org_id) -> Payload:
    """删除信任组织根证书待签名payload生成
    :param trust_root_org_id: 组织Id
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.11 添加信任成员证书待签名payload生成

```python
def create_chain_config_trust_member_add_payload(self, trust_member_org_id: str, trust_member_node_id: str,
                                                 trust_member_info: str, trust_member_role: str,
                                                 ) -> Payload: 
    """
    生成添加三方TRUST_ROOT待签名Payload
    :param trust_member_org_id: 组织ID
    :param trust_member_node_id: 节点ID
    :param trust_member_info: 节点信息
    :param trust_member_role: 节点角色
    :return: 待签名Payload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.12 删除信任成员证书待签名payload生成

```python
def create_chain_config_trust_member_delete_payload(self, trust_member_info: str) -> Payload: 
    """
    生成删除三方TRUST_ROOT待签名Payload
    :param trust_member_info: 节点证书信息
    :return: 待签名Payload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.13 添加权限配置待签名payload生成
```python
def create_chain_config_permission_add_payload(self, permission_resource_name, policy) -> Payload:
    """添加权限配置待签名payload生成
    :param permission_resource_name: 权限名
    :param policy: 权限规则
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.14 更新权限配置待签名payload生成

```python
def create_chain_config_permission_update_payload(self, permission_resource_name,
                                                  policy: policy_pb2.Policy) -> Payload:
    """更新权限配置待签名payload生成
    :param permission_resource_name: 权限名
    :param policy: 权限规则
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.15 删除权限配置待签名payload生成

```python
def create_chain_config_permission_delete_payload(self, permission_resource_name) -> Payload:
    """删除权限配置待签名payload生成
    :param permission_resource_name: 权限名
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.16 添加共识节点地址待签名payload生成


```python
def create_chain_config_consensus_node_id_add_payload(self, node_org_id: str,
                                                      node_ids: list) -> Payload:
    """添加共识节点地址待签名payload生成
    :param node_org_id: 节点组织Id eg. 'wx-org5.chainmaker.org'
    :param node_ids: 节点Id列表 eg. ['QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4']
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.17 更新共识节点地址待签名payload生成

```python
def create_chain_config_consensus_node_id_update_payload(self, node_org_id: str, node_old_id: str,
                                                             node_new_id: str) -> Payload:
    """更新共识节点地址待签名payload生成
    :param node_org_id: 节点组织Id
    :param node_old_id: 节点原Id
    :param node_new_id: 节点新Id
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.18 删除共识节点地址待签名payload生成

```python
def create_chain_config_consensus_node_id_delete_payload(self, node_org_id: str,
                                                         node_id: str) -> Payload:
    """删除共识节点地址待签名payload生成
    :param node_org_id: 节点组织Id
    :param node_id: 节点Id
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.19 添加共识节点待签名payload生成

```python
def create_chain_config_consensus_node_org_add_payload(self, node_org_id: str,
                                                       node_ids: list) -> Payload:
    """添加共识节点待签名payload生成
    :param node_org_id: 节点组织Id
    :param node_ids: 节点Id
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.20 更新共识节点待签名payload生成


```python
def create_chain_config_consensus_node_org_update_payload(self, node_org_id: str,
                                                              node_ids: list) -> Payload:
    """更新共识节点待签名payload生成
    :param node_org_id: 节点组织Id
    :param node_ids: 节点Id
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.21 删除共识节点待签名payload生成

```python
def create_chain_config_consensus_node_org_delete_payload(self, node_org_id) -> Payload:
    """删除共识节点待签名payload生成
    :param node_org_id: 节点组织Id
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.22 添加共识扩展字段待签名payload生成

```python
def create_chain_config_consensus_ext_add_payload(self, params) -> Payload:
    """添加共识扩展字段待签名payload生成
    :param params: 字段key、value对
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.23 更新共识扩展字段待签名payload生成

```python
def create_chain_config_consensus_ext_update_payload(self, params) -> Payload:
    """更新共识扩展字段待签名payload生成
    :param params: 字段key、value对
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```

#### 3.24 删除共识扩展字段待签名payload生成

```python
def create_chain_config_consensus_ext_delete_payload(self, keys) -> Payload:
    """删除共识扩展字段待签名payload生成
    :param keys: 待删除字段
    :return: request_pb2.SystemContractPayload
    :raises InvalidParametersError: 无效参数
    """
```
#### 3.25 生成链配置变更地址类型待签名Payload
```python
 def create_chain_config_alter_addr_type_payload(self, addr_type: AddrType) -> Payload:
    """
    生成链配置变更地址类型待签名Payload
    :param addr_type: 地址类型
    :return: 待签名Payload
    """
```
#### 3.26 生成链配置切换启用/禁用Gas待签名Payload

```python
def create_chain_config_enable_or_disable_gas_payload(self) -> Payload:
    """
    生成链配置切换启用/禁用Gas待签名Payload
    :return: 待签名Payload
    """
```

### 4 证书管理接口

#### 4.1 用户证书添加

```python
def add_cert(self, timeout=None, with_sync_result=False) -> TxResponse:
    """添加用户的证书
    :param timeout: 设置请求超时时间
    :param with_sync_result: 同步返回请求结果
    :return: Response
    :raises RequestError: 请求失败
    """
```

#### 4.2 用户证书删除


```python
def delete_cert(self, cert_hashes: List[str], endorse_users: List[EndorsementEntry] = None, timeout=None,
                    with_sync_result=True) -> TxResponse:
    """删除用户的证书
    :param cert_hashes: 证书hash列表, 每个证书hash应转为hex字符串
    :param endorse_users 背书用户,为None时使用self.endorse_users
    :param timeout: 超时时长
    :param with_sync_result: 是否同步返回请求结果
    :return: Response
    :raises RequestError: 请求失败
    """
```

#### 4.3 用户证书查询

```python
 def query_cert(self, cert_hashes: List[str], timeout: int = None) -> CertInfos:
    """查询证书的hash是否已经上链
    :param cert_hashes: 证书hash列表(List)，每个证书hash应转为hex字符串
    :param timeout: 设置请求超时时间
    :return: result_pb2.CertInfos
    :raises 查询不到证书抛出 RequestError
    """
```

#### 4.4 获取用户证书哈希

```python
def get_cert_hash(self) -> bytes:
    """
    返回用户的签名证书哈希
    :return: 证书hash值
    """
```

#### 4.5 生成证书管理操作Payload（三合一接口）

```python
def _create_cert_manage_payload(self, method: str, params: Union[dict, list] = None):
    """创建证书管理payload
    :param method: 方法名。CERTS_FROZEN(证书冻结)/CERTS_UNFROZEN(证书解冻)/CERTS_REVOCATION(证书吊销)
    :param params: 证书管理操作参数，dict格式
    :return: 证书管理payload
    :raises
    """
```

#### 4.6 生成证书冻结操作Payload


```python
def create_cert_freeze_payload(self, certs: List[str]):
    """对证书管理的payload 进行签名
    :param certs: 证书列表(List)，证书为证书文件读取后的字符串格式
    :return: Payload
    :raises
    """
```

#### 4.7 生成证书解冻操作Payload

```python
def create_cert_unfreeze_payload(self, certs: List[str]):
    """对证书管理的payload 进行签名
    :param certs: 证书列表，证书为证书文件读取后的字符串格式
    :return: Payload
    :raises
    """
```

#### 4.8 生成证书吊销操作Payload

```python
def create_cert_revoke_payload(self, cert_crl: str):
    """对证书管理的payload 进行签名
    :param cert_crl: 证书吊销列表 文件内容，字符串形式
    :return: Payload
    :raises
    """
```

#### 4.9 待签payload签名

*一般需要使用具有管理员权限账号进行签名*

```python
def sign_cert_manage_payload(self, payload) -> EndorsementEntry:
    """
    对证书管理的payload 进行签名
    :param payload_bytes: 待签名的payload字节数组
    :return: paylaod的背书信息
    :raises
    """
```

#### 4.10 发送证书管理请求（证书冻结、解冻、吊销）



- payload: 交易payload
- endorsers: 背书签名信息列表
- timeout: 超时时间，单位：s，若传入-1，将使用默认超时时间：10s
- withSyncResult: 是否同步获取交易执行结果 当为true时，若成功调用，common.TxResponse.ContractResult.Result为common.TransactionInfo
  当为false时，若成功调用，common.TxResponse.ContractResult为空，可以通过common.TxResponse.TxId查询交易结果

```python
def send_cert_manage_request(self, payload, endorsers: List[EndorsementEntry], timeout=None,
                             with_sync_result=False) -> TxResponse:
    """
    发送证书管理请求
    :param payload: 交易payload
    :param endorsers: 背书列表
    :param timeout: 超时时间
    :param with_sync_result: 是否同步交易执行结果。如果不同步，返回tx_id，供异步查询; 同步则循环等待，返回交易的执行结果。
    :return: Response
    :raises RequestError: 请求失败
    """
```

### 5 消息订阅接口

#### 5.1 区块订阅

```python
def subscribe_block(self, start_block: int, end_block: int, with_rw_set=False, only_header=False,
                    timeout: int = Defaults.SUBSCRIBE_TIMEOUT) -> Generator[
        Union[BlockInfo, BlockWithRWSet, BlockHeader], None, None]:
        """
        订阅区块
        :param start_block: 订阅的起始区块
        :param end_block: 订阅的结束区块
        :param with_rw_set: 是否包含读写集
        :param only_header: 是否只订阅区块头
        :param timeout: 订阅尚未产生区块的等待超时时间, 默认60s
        :return: 生成器，如果only_header=True,其中每一项为BlockHeader
                        另外如果with_rw_set=True,其中每一项为BlockWithRWSet
                        否则其中每一项为BlockInfo
        """
```

#### 5.2 交易订阅

```python
def subscribe_tx(self, start_block: int, end_block: int, contract_name: str, tx_ids: List[str],
                 timeout: int = None) -> Generator[Transaction, None, None]:
    """
    订阅交易
    :param start_block: 订阅的起始区块
    :param end_block: 订阅的结束区块
    :param contract_name: 交易所属合约名称
    :param tx_ids: 指定交易Id列表进行订阅
    :param timeout: RPC请求超时时间
    :return: 生成器，其中每一项为Transaction
    """
```

#### 5.3 合约事件订阅

```python
def subscribe_contract_event(self, start_block, end_block, topic: str, contract_name: str,
                             timeout: int = Defaults.SUBSCRIBE_TIMEOUT) -> Generator[ContractEventInfoList, None, None]:
    """
    定于合约事件
    :param start_block: 订阅的起始区块
    :param end_block: 订阅的结束区块
    :param topic: 订阅待事件主题
    :param contract_name: 事件所属合约名称
    :param timeout: RPC请求超时时间
    :return: 生成器，其中每一项为ContractEventInfoList
    """
```

#### 5.4 多合一订阅

```python
def subscribe(self, payload: Payload, timeout: int = Defaults.SUBSCRIBE_TIMEOUT, callback: Callable = None) -> None:
    """
    订阅区块、交易、合约事件
    :param payload: 订阅Payload
    :param timeout: 订阅待产生区块等待超时时间
    :param callback: 回调函数，默认为self._callback
    :return: None
    """
```

### 6 证书压缩

*开启证书压缩可以减小交易包大小，提升处理性能*

#### 6.1 启用压缩证书功能

```python
 def enable_cert_hash(self):
    """启用证书hash，会修改实例的enabled_crt_hash值。默认为不启用。
    :raises OnChainFailedError: 证书上链失败
    """
```

#### 6.2 停用压缩证书功能

```python
def disable_cert_hash(self):
    """
    关闭证书hash，会修改实例的enabled_crt_hash值。
    """
```

### 7 层级属性加密类接口
> 暂不支持

### 8 数据归档接口

**（注意：请使用归档工具cmc进行归档操作，以下接口是归档原子接口，并不包括归档完整流程）**

#### 8.1 获取已归档区块高度

```python
def get_archived_block_height(self) -> int:
    """查询已归档的区块高度
    :return: 区块高度
    :raises RequestError: 请求失败
    """
```

#### 8.2 构造数据归档区块Payload

```python
def create_archive_block_payload(self, target_block_height: int) -> Payload:
    """
    构造数据归档区块Payload
    :param target_block_height: 归档目标区块高度
    :return: 待签名Payload
    """
```

#### 8.3 构造归档数据恢复Payload

```python
def create_restore_block_payload(self, full_block: bytes) -> Payload:
    """
    构造归档数据恢复Payload
    :param full_block: 完整区块数据（对应结构：store.BlockWithRWSet）
    :return: 待签名Payload
    """
```

#### 8.4 获取归档操作Payload签名

```python
def sign_archive_payload(self, payload: Payload) -> Payload:
    """
    签名归档请求
    :param payload: 待签名归档请求Payload
    :return: 签名后的Payload
    """
```

#### 8.5 发送归档请求

```python
def send_archive_block_request(self, payload: Payload, timeout: int = None)->TxResponse:
    """
    发送归档请求
    :param payload: 归档待签名Payload
    :param timeout: 超时时间
    :return: 交易响应TxResponse
    :raise: 已归档抛出InternalError
    """
```

#### 8.6 归档数据恢复

```python
def send_restore_block_request(self, payload, timeout: int = None)->TxResponse:
    """
    发送恢复归档区块请求
    :param payload: 归档请求待签名Payload
    :param timeout: RPC请求超时事件
    :return: 交易响应信息
    """
```

#### 8.7 根据交易Id查询已归档交易


```python
def get_archived_block_by_tx_id(self, tx_id: str, with_rwset: bool = False) -> BlockInfo:
    """
    根据交易id查询已归档的区块
    :param tx_id: 交易ID
    :param with_rwset: 是否包含读写集
    :return: 区块详情 BlockInfo
    :raises RequestError: 请求失败
    """
```

#### 8.8 根据区块高度查询已归档区块

```python
def get_archived_block_by_height(self, block_height: int, with_rwset: bool = False) -> BlockInfo:
    """
    根据区块高度，查询已归档的区块
    :param block_height: 区块高度
    :param with_rwset: 是否包含读写集
    :return: 区块详情 BlockInfo
    :raises RequestError: 请求失败
    """
```

#### 8.9 根据区块高度查询已归档完整区块(包含：区块数据、读写集、合约事件日志)


```python
def get_archived_full_block_by_height(self, block_height: int) -> BlockWithRWSet:
    """
    根据区块高度，查询已归档的完整区块（包含合约event info）
    :param block_height: 区块高度
    :return: 区块详情 BlockInfo
    :raises RequestError: 请求失败
    """
```

#### 8.10 根据区块哈希查询已归档区块


```python
def get_archived_block_by_hash(self, block_hash: str, with_rwset: bool = False) -> BlockInfo:
    """
    根据区块hash查询已归档的区块
    :param block_hash: 区块hash
    :param with_rwset: 是否包含读写集
    :return: 区块详情 BlockInfo
    :raises RequestError: 请求失败
    """
```

#### 8.11 根据交易Id查询已归档区块

```python
 def get_archived_block_by_tx_id(self, tx_id: str, with_rwset: bool = False) -> BlockInfo:
    """
    根据交易id查询已归档的区块
    :param tx_id: 交易ID
    :param with_rwset: 是否包含读写集
    :return: 区块详情 BlockInfo
    :raises RequestError: 请求失败
    """
```

### 9 隐私计算系统合约接口
> 暂时不支持

### 10 系统类接口

#### 10.1 SDK停止接口

*关闭连接池连接，释放资源*

```python
def stop(self) -> None:
    """停止客户端-关闭所有channel连接"""
```

#### 10.2 获取链版本

```python
def get_chainmaker_server_version(self) -> str:
    """获取chainmaker版本号"""
```

### 11 公钥身份类接口

#### 11.1 构造添加公钥身份请求

```python
def create_pubkey_add_payload(self, pubkey: str, org_id: str, role: Role)->Payload:
    """
    创建公钥添加Payload
    :param pubkey: 公钥文件内容
    :param org_id: 组织ID
    :param role: 角色
    :return: 生成的payload
    """
```

#### 11.2 构造删除公钥身份请求

```python
def create_pubkey_delete_payload(self, pubkey: str, org_id: str)->Payload:
    """
    创建公钥删除Payload
    :param pubkey: 公钥
    :param org_id: 组织id
    :return: 生成的payload
    """
```

#### 11.3 构造查询公钥身份请求

```python
def create_pubkey_query_payload(self, pubkey: str)->Payload:
    """
    创建公钥查询Payload
    :param pubkey: 公钥文件内容
    :return: 生成的payload
    """
```

#### 11.4 发送公钥身份管理请求（添加、删除）


```python
def send_pubkey_manage_request(self, payload, endorsers: list = (), timeout: int = None,
                                   with_sync_result: bool = False)->TxResponse:
    """
    发送公钥管理请求
    :param payload: 公钥管理payload
    :param endorsers: 背书列表
    :param timeout: 超时时间
    :param with_sync_result: 是否同步结果
    :return: 交易响应或事务交易信息
    """
```

### 12 多签类接口

#### 12.1 发起多签请求

```python
def multi_sign_contract_req(self, payload: Payload, timeout: int = None, with_sync_result: bool = False) -> TxResponse:
    """
    发起多签请求
    :param payload: 待签名payload
    :param timeout: 请求超时时间
    :param with_sync_result: 是否同步获取交易结果
    :return: 交易响应或交易信息
    """
```

#### 12.2 发起多签投票


```python
def multi_sign_contract_vote(self, multi_sign_req_payload, endorser: User, is_agree: bool = True,
                                 timeout: int = None, with_sync_result: bool = False) -> TxResponse:
    """
    对请求payload发起多签投票
    :param multi_sign_req_payload: 待签名payload
    :param endorser: 投票用户对象
    :param is_agree: 是否同意，true为同意，false则反对
    :param timeout: 请求超时时间
    :param with_sync_result: 是否同步获取交易结果
    :return: 交易响应或交易信息
    """
```

#### 12.3 根据txId查询多签状态

```python
def multi_sign_contract_vote_tx_id(self, tx_id, endorser: User, is_agree: bool,
                                       timeout: int = None, with_sync_result: bool = False) -> TxResponse:
    """
    对交易ID发起多签投票
    :param tx_id: 交易ID
    :param endorser: 投票用户对象
    :param is_agree: 是否同意，true为同意，false则反对
    :param timeout: 请求超时时间
    :param with_sync_result: 是否同步获取交易结果
    :return: 交易响应或交易信息
    """
```

#### 12.4 根据发起多签请求所需的参数构建payload

```python
def create_multi_sign_req_payload(self, params: Union[list, dict]) -> Payload:
    """
    根据发起多签请求所需的参数构建payload
    :param params: 发起多签请求所需的参数
    :return: 待签名Payload
    """
```

