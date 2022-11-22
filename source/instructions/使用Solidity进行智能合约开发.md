# 使用Solidity进行智能合约开发
作者：长安链团队 丁有振、韩天乐、袁太富、胡振远

读者对象：本章节主要描述使用Solidity进行ChainMaker合约编写的方法，主要面向于使用Solidity进行ChainMaker的合约开发的开发者。

Solidity 是一门面向合约的、为实现智能合约而创建的高级编程语言。这门语言受到了 C++，Python 和 Javascript 语言的影响，设计的目的是能在虚拟机（EVM）上运行。

Solidity 是静态类型语言，支持继承、库和复杂的用户定义类型等特性。


## 环境依赖

1. 操作系统

   无特殊要求，Linux、Mac和Windows均支持。

2. 软件依赖

   无

3. 长安链环境准备

   启动长安链，以及长安链CMC工具，用于将写编写好的合约，部署到链上进行测试。相关安装教程请详见：

   - [部署长安链教程。](../quickstart/通过命令行体验链.md)
   - [部署长安链CMC工具的教程。](../dev/命令行工具.md)


## 编写Solidity智能合约

### 搭建开发环境
开发者无需自己搭建开发环境，可使用[Remix](https://remix.ethereum.org)在线IDE开发solidity合约。长安链对solidity完全兼容，使用Remix开发的或者以太坊生态内的solidity合约，可直接在长安链部署运行。

### 代码编写规则

solidity语法和代码编写规则参见solidity官方开发文档：https://docs.soliditylang.org/。

#### 长安链solidity与以太坊solidity的异同

长安链的solidity在内置接口、预编译合约和跨合约调用上，与以太坊有一些区别，具体参见本章节下，各分小节详细内容。

##### solidity 内置接口

solidity为开发者提供了一些内置接口，包括内置变量和函数，可在合约中直接使用。因为长安链为无币链，所以，与原生token相关的内置接口，多默认为0，详见以下内置接口说明。


```solidity
//指定区块的区块哈希，已经不推荐使用，由 blockhash(uint blockNumber) 代替
block.blockhash(uint blockNumber) returns (bytes32)

//address类型，当前区块的出块节点地址，即 block 的 proposer
block.coinbase

//uint类型，当前区块难度，值固定为0
block.difficulty

//uint类型，当前区块 gas 限额
block.gaslimit

//uint类型，当前区块号
block.number

//uint类型，自 unix epoch 起始当前区块以秒计的时间戳
block.timestamp

//uint256类型，剩余的 gas
gasleft() returns 

//bytes类型，完整的 calldata
msg.data 

//uint类型，剩余 gas - 自 0.4.21 版本开始已经不推荐使用，由 gesleft() 代替
msg.gas 

//address类型，消息发送者（当前调用）
msg.sender 

//bytes4类型，calldata 的前 4 字节（也就是函数标识符）
msg.sig 

//uint类型，随消息发送的 wei 的数量，值固定为0
msg.value 

//uint类型，目前区块时间戳（block.timestamp）
now 

//uint类型，交易的 gas 价格，值固定为1
tx.gasprice 

//address类型，交易发起者
tx.origin 
```

##### 预编译合约

因为EVM是基于栈的虚拟机，它根据操作的内容来计算gas，所以如果牵涉到十分复杂的计算，把运算过程放在EVM中执行就可能十分地低效，同时消耗非常多的gas。

预编译合约是EVM中为了提供一些不适合写成opcode的较为复杂的库函数（多用于加密、哈希等复杂运算）而采用的一种折中方案，适用于合约逻辑简单但调用频繁，或者合约逻辑固定而计算量大的场景，与以太坊相比，长安链有五个预编译合约尚未支持，正在开发中，详见以下表格。

| **预编译合约名**                                      | **地址** | **功能**                                                     |
| ----------------------------------------------------- | -------- | ------------------------------------------------------------ |
| **ecrecover(hash, v, r, s)**                          | **0x01** | 根据给定签名计算地址——**目前长安链尚不支持，待后续版本实现** |
| **sha256(data)**                                      | **0x02** | 计算SHA256哈希                                               |
| **ripemd160(data)**                                   | **0x03** | 计算RIPEMD160哈希                                            |
| **datacopy(data)**                                    | **0x04** | 只读拷贝数据                                                 |
| **bigModExp(base, exp, mod)**                         | **0x05** | 计算base ^ exp % mod的结果                                   |
| **bn256Add(ax, ay, bx, by)**                          | **0x06** | BN256曲线点加法计算，成功返回(ax,ay)+(bx,by)，表示这两点是BN256曲线上的有效点，失败返回0——**目前长安链尚不支持，待后续版本实现** |
| **bn256ScalarMul(x, y, scalar)**                      | **0x07** | BN256曲线乘法，成功返回一个曲线点scalar\*(x,y)，表示(x,y)点是BN256曲线上的有效点，失败返回0——**目前长安链尚不支持，待后续版本实现** |
| **bn256Pairing(a1, b1, a2, b2, a3, b3, ..., ak, bk)** | **0x08** | 实现BN256曲线配对操作，进行zkSNARK验证。——**目前长安链尚不支持，待后续版本实现** |
| **blake2F(rounds, h, m, t, f)**                       | **0x09** | 实现BLAKER2b F压缩功能。——**目前长安链尚不支持，待后续版本实现** |


##### 跨合约调用对比

长安链对solidity的跨合约调用做了增量修改，除完全兼容和支持以太坊所支持的跨合约调用外，还支持对（长安链已有的）其他虚拟机合约跨合约调用。具体内容，参见[跨合约调用教程](./如何进行跨合约调用.md)。

### 合约示例源码展示

**Token合约**示例，实现功能ERC20

```
/*
SPDX-License-Identifier: Apache-2.0
*/
pragma solidity >0.5.11;
contract Token {

    string public name = "token";      //  token name
    string public symbol = "TK";           //  token symbol
    uint256 public decimals = 6;            //  token digit

    mapping (address => uint256) public balanceOf;
    mapping (address => mapping (address => uint256)) public allowance;

    uint256 public totalSupply = 0;
    bool public stopped = false;

    uint256 constant valueFounder = 100000000000000000;
    address owner = address(0x0);

    modifier isOwner {
        assert(owner == msg.sender);
        _;
    }

    modifier isRunning {
        assert (!stopped);
        _;
    }

    modifier validAddress {
        assert(address(0x0) != msg.sender);
        _;
    }

    constructor (address _addressFounder) {
        owner = msg.sender;
        totalSupply = valueFounder;
        balanceOf[_addressFounder] = valueFounder;
        
        emit Transfer(address(0x0), _addressFounder, valueFounder);
    }

    function transfer(address _to, uint256 _value) public isRunning validAddress returns (bool success) {
        require(balanceOf[msg.sender] >= _value);
        require(balanceOf[_to] + _value >= balanceOf[_to]);
        balanceOf[msg.sender] -= _value;
        balanceOf[_to] += _value;
        emit Transfer(msg.sender, _to, _value);
        return true;
    }

    function transferFrom(address _from, address _to, uint256 _value) public isRunning validAddress returns (bool success) {
        require(balanceOf[_from] >= _value);
        require(balanceOf[_to] + _value >= balanceOf[_to]);
        require(allowance[_from][msg.sender] >= _value);
        balanceOf[_to] += _value;
        balanceOf[_from] -= _value;
        allowance[_from][msg.sender] -= _value;
        emit Transfer(_from, _to, _value);
        return true;
    }

    function approve(address _spender, uint256 _value) public isRunning validAddress returns (bool success) {
        require(_value == 0 || allowance[msg.sender][_spender] == 0);
        allowance[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }

    function stop() public isOwner {
        stopped = true;
    }

    function start() public isOwner {
        stopped = false;
    }

    function setName(string memory _name) public isOwner {
        name = _name;
    }

    function burn(uint256 _value) public {
        require(balanceOf[msg.sender] >= _value);
        balanceOf[msg.sender] -= _value;
        balanceOf[address(0x0)] += _value;
        emit Transfer(msg.sender, address(0x0), _value);
    }

    event Transfer(address indexed _from, address indexed _to, uint256 _value);
    event Approval(address indexed _owner, address indexed _spender, uint256 _value);
}

```

### 编译合约

#### 使用Docker镜像搭建编译环境

开发者可使用ChainMaker已经打包好的Docker镜像编译solidity合约代码，ChainMaker官方已经将容器发布至 [docker hub](https://hub.docker.com/u/chainmakerofficial)

**拉取镜像**

请使用以下命令拉取用于编译solidity合约的镜像。

```sh
docker pull chainmakerofficial/chainmaker-solidity-contract:2.0.0
```

**启动镜像**

启动镜像前，需要指定本地开发目录，用于映射为docker镜像的home目录。请指定你本机的工作目录$WORK_DIR，例如/data/workspace/contract，挂载到docker容器中以方便后续进行必要的一些文件拷贝。

```sh
#启动并进入容器，$WORK_DIR即本地工作目录
docker run -it --name chainmaker-solidity-contract -v $WORK_DIR:/home chainmakerofficial/chainmaker-solidity-contract:2.0.0 bash
# 或者先后台启动
docker run -d --name chainmaker-solidity-contract -v $WORK_DIR:/home chainmakerofficial/chainmaker-solidity-contract:2.0.0 bash -c "while true; do echo hello world; sleep 5;done"
# 再进入容器
docker exec -it chainmaker-solidity-contract bash
```

#### 编译示例合约

在本地开发目录内的solidity合约，可以在docker编译镜像的home目录内直接看到，因为二者已映射。进入docker编译镜像后，切换到home目录，执行以下命令，即可编译solidity合约。

```sh
# cd /home/
# solc --abi --bin --hashes --overwrite -o . token.sol
```

solc为编译命令， --abi选项指示生成abi文件，--bin指示生成字节码文件， --hashes指示生成函数签名文件， --overwrite指示如果生成文件已存在则覆盖， -o 指示编译生成的文件存放的目录。solc命令更详细用法，可使用solc --help查看。

生成的字节码位于solc命令用 -o 指定的目录内，示例 中为当前目录：

```
/home/Token.bin
```

#### 编译说明

solc编译命令使用的是0.8.4+commit.c7e474f2.Linux.g++版本的编译器，被编译的solidity合约版本号必须大于等于0.8.4，否则有可能编译告警或报错。

如果开发者不愿意修改solidity合约以适应solc编译器的版本，那么也可以直接使用Remix编译。通过Remix编译出的字节码也可以在长安链上直接部署运行。

#### 模拟运行示例合约

##### 模拟部署

```
# evm Token.bin init_contract data 00000000000000000000000013f0c1639a9931b0ce17e14c83f96d4732865b58
```

evm为测试合约部署和调用的命令，示例中Token.bin文件后跟随的是被调用合约的方法，**init_contract**表示调用的是合约的构造方法，该方法只在部署合约时调用一次。

**data**选项标识紧随其后的数据 00000000000000000000000013f0c1639a9931b0ce17e14c83f96d4732865b58 为 calldata。**calldata**由被调用方法的签名和参数的ABI编码组成。calldata需要通过ABI接口生成，这里的calldata是一个地址，是示例token合约构造方法的参数。

##### calldata生成示例

长安链项目的common模块提供了abi计算包，开发者导入abi包后，可指定合约方法和参数生成calldata。

```go
import "chainmaker.org/chainmaker/common/v2/evmutils/abi"
...
//读取abi文件
abiBytes, _ := ioutil.ReadFile("xxxx.abi")
//构建abi对象
abiObj, _ := abi.JSON(strings.NewReader(string(abiBytes)))
//计算calldata
calldata, err := abiObj.Pack("methodName", big.NewInt(100))
...
```

##### 模拟调用

执行模拟部署后，把部署返回的result手动保存在DeployedToken.bin文件中。

```
/home/DeployedToken.bin
```

使用evm命令，可调用DeployedToken.bin的方法，例如：执行balanceOf(address)方法。
```solidity
# evm DeployedToken.bin 0x70a08231 data 0x70a0823100000000000000000000000013f0c1639a9931b0ce17e14c83f96d4732865b58
```

命令中，DeployedToken.bin后的0x70a08231为被调用方法的签名，签名可通过编译Token.sol合约时生成的Token.signatures文件查看，如下所示。

```
# cat Token.signatures 
dd62ed3e: allowance(address,address)
095ea7b3: approve(address,uint256)
70a08231: balanceOf(address)
42966c68: burn(uint256)
313ce567: decimals()
06fdde03: name()
c47f0027: setName(string)
be9a6555: start()
07da68f5: stop()
75f12b21: stopped()
95d89b41: symbol()
18160ddd: totalSupply()
a9059cbb: transfer(address,uint256)
23b872dd: transferFrom(address,address,uint256)
```

模拟调用命令中的**data**和后面的calldata在模拟部署小节内已阐述，此处不做赘述。





