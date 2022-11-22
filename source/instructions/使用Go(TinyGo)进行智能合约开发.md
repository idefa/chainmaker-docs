# 使用TinyGo进行智能合约开发
作者：长安链团队 蔡志宏、韩天乐、柳云龙、袁太富、胡振远

读者对象：本章节主要描述使用TinyGo进行ChainMaker合约编写的方法，主要面向于使用Go进行ChainMaker的合约开发的开发者。为了最小化wasm文件尺寸，使用的是TinyGO编译器。
> 注意：TinyGo 的功能在 ChainMaker V2.3.0 之后将不再更新。推荐使用 docker-go 进行 Golang 合约的开发。

## 环境依赖

使用 TinyGo 开发用于 ChainMaker 的 wasm 合约，需要安装 TinyGo 编译器，同时注意以下几点： 

- TinyGo对wasm的支持不太完善，对内存逃逸分析、GC等方面有不足之处，比较容易造成栈溢出。在开发合约时，应尽可能减少循环、内存申请等业务逻辑，使变量的栈内存地址在64K以内，要求tinygo version >= 0.17.0，推荐使用0.17.0。

- TinyGo对导入的包支持有限，请参考：https://tinygo.org/lang-support/stdlib/，对列表中显示已支持的包，实际测试发现支持的并不完整，会发生一些错误，需要在实际开发过程中进行测试检验。

- TinyGo引擎不支持fmt和strconv包。


## 编写TinyGo智能合约

### 搭建开发环境

为了简化用户的配置，我们已将开发、编译环境做成镜像，放在 chainmakerofficial/chainmaker-go-contract:2.1.0 上，直接执行下列命令可以拉取镜像，构建开发环境：

```shell
    docker pull chainmakerofficial/chainmaker-go-contract:2.1.0
    docker run -it --name chainmaker-go-contract -v $WORK_DIR:/home chainmakerofficial/chainmaker-go-contract:2.1.0 bash
```

其中，$WORK_DIR 为本地工作目录，挂载到 docker 的 /home 下面。

### 代码编写规则

**代码入口**

```go
func main() { // sdk代码中，有且仅有一个main()方法
	// 空，不做任何事。仅用于对tinygo编译支持
}

```

**对链暴露方法写法为：**

- //export upgrade
- func  method_name(): 不可带参数，无返回值

```go
//export init_contract 表明对外暴露方法名称
func initContract() {

}
```

**其中init_contract、upgrade方法必须有且对外暴露**

- init_contract：创建合约会执行该方法
- upgrade： 升级合约会执行该方法

```rust
// 安装合约时会执行此方法，必须。ChainMaker不允许用户直接调用该方法。
//export init_contract
func initContract() {

}
// 升级合约时会执行此方法，必须。ChainMaker不允许用户直接调用该方法。
//export upgrade
func upgrade() {

}
```

### 示例代码说明

#### 存证合约示例源码展示

实现功能:

1、存储文件哈希和文件名称和该交易的ID。

2、通过文件哈希查询该条记录

```go
/*
Copyright (C) BABEC. All rights reserved.

SPDX-License-Identifier: Apache-2.0

一个 文件存证 的存取示例 fact

*/

package main

import (
  "chainmaker.org/contract-sdk-tinygo/sdk/convert"
)

// 安装合约时会执行此方法，必须
//export init_contract
func initContract() {
  // 此处可写安装合约的初始化逻辑

}

// 升级合约时会执行此方法，必须
//export upgrade
func upgrade() {
  // 此处可写升级合约的逻辑

}

// 存证对象
type Fact struct {
  fileHash string
  fileName string
  time     int32 // second
  ec       *EasyCodec
}

// 新建存证对象
func NewFact(fileHash string, fileName string, time int32) *Fact {
  fact := &Fact{
    fileHash: fileHash,
    fileName: fileName,
    time:     time,
  }
  return fact
}

// 获取序列化对象
func (f *Fact) getEasyCodec() *EasyCodec {
  if f.ec == nil {
    f.ec = NewEasyCodec()
    f.ec.AddString("fileHash", f.fileHash)
    f.ec.AddString("fileName", f.fileName)
    f.ec.AddInt32("time", f.time)
  }
  return f.ec
}

// 序列化为json字符串
func (f *Fact) toJson() string {
  return f.getEasyCodec().ToJson()
}

// 序列化为cmec编码
func (f *Fact) marshal() []byte {
  return f.getEasyCodec().Marshal()
}

// 反序列化cmec为存证对象
func unmarshalToFact(data []byte) *Fact {
  ec := NewEasyCodecWithBytes(data)
  fileHash, _ := ec.GetString("fileHash")
  fileName, _ := ec.GetString("fileName")
  time, _ := ec.GetInt32("time")

  fact := &Fact{
    fileHash: fileHash,
    fileName: fileName,
    time:     time,
    ec:       ec,
  }
  return fact
}

// 对外暴露 save 方法，供用户由 SDK 调用
//export save
func save() {
  // 获取上下文
  ctx := NewSimContext()

  // 获取参数
  fileHash, err1 := ctx.ArgString("file_hash")
  fileName, err2 := ctx.ArgString("file_name")
  timeStr, err3 := ctx.ArgString("time")

  if err1 != SUCCESS || err2 != SUCCESS || err3 != SUCCESS {
    ctx.Log("get arg fail.")
    ctx.ErrorResult("get arg fail.")
    return
  }

  time, err := convert.StringToInt32(timeStr)
  if err != nil {
    ctx.ErrorResult(err.Error())
    ctx.Log(err.Error())
    return
  }

  // 构建结构体
  fact := NewFact(fileHash, fileName, int32(time))

  // 序列化：两种方式
  jsonStr := fact.toJson()
  bytesData := fact.marshal()

  //发送事件
  ctx.EmitEvent("topic_vx", fact.fileHash, fact.fileName)

  // 存储数据
  ctx.PutState("fact_json", fact.fileHash, jsonStr)
  ctx.PutStateByte("fact_bytes", fact.fileHash, bytesData)

  // 记录日志
  ctx.Log("【save】 fileHash=" + fact.fileHash)
  ctx.Log("【save】 fileName=" + fact.fileName)
  // 返回结果
  ctx.SuccessResult(fact.fileName + fact.fileHash)
}

// 对外暴露 find_by_file_hash 方法，供用户由 SDK 调用
//export find_by_file_hash
func findByFileHash() {
  ctx := NewSimContext()
  // 获取参数
  fileHash, _ := ctx.ArgString("file_hash")
  // 查询Json
  if result, resultCode := ctx.GetStateByte("fact_json", fileHash); resultCode != SUCCESS {
    // 返回结果
    ctx.ErrorResult("failed to call get_state, only 64 letters and numbers are allowed. got key:" + "fact" + ", field:" + fileHash)
  } else {
    // 返回结果
    ctx.SuccessResultByte(result)
    // 记录日志
    ctx.Log("get val:" + string(result))
  }

  // 查询EcBytes
  if result, resultCode := ctx.GetStateByte("fact_bytes", fileHash); resultCode == SUCCESS {
    // 反序列化
    fact := unmarshalToFact(result)
    // 返回结果
    ctx.SuccessResult(fact.toJson())
    // 记录日志
    ctx.Log("get val:" + fact.toJson())
    ctx.Log("【find_by_file_hash】 fileHash=" + fact.fileHash)
    ctx.Log("【find_by_file_hash】 fileName=" + fact.fileName)
  }
}

func main() {

}

```


### 合约SDK接口描述
长安链提供Tinygo合约与链交互的相关接口，写合约时可直接导入包，并进行引用，具体信息可参考文章末尾"接口描述章节"。



### 编译合约

#### 使用Docker镜像搭建编译环境

ChainMaker官方已经将容器发布至 https://hub.docker.com/u/chainmakerofficial

拉取镜像
```
docker pull chainmakerofficial/chainmaker-go-contract:2.1.0
```

请指定你本机的工作目录$WORK_DIR，例如/data/workspace/contract，挂载到docker容器中以方便后续进行必要的一些文件拷贝

```sh
docker run -it --name chainmaker-go-contract -v $WORK_DIR:/home chainmakerofficial/chainmaker-go-contract:2.1.0 bash
# 或者先后台启动
docker run -d  --name chainmaker-go-contract -v $WORK_DIR:/home chainmakerofficial/chainmaker-go-contract:2.1.0 bash -c "while true; do echo hello world; sleep 5;done"
# 再进入容器
docker exec -it chainmaker-go-contract bash
```



#### 编译示例合约

```sh
cd /home/
# 解压缩合约SDK源码
tar xvf /data/contract_go_template.tar.gz
cd contract_tinygo
# 编译main.go合约
sh build.sh
```

生成合约的字节码文件在

```
/home/contract_tinygo/chainmaker-contract-go.wasm
```

#### 示例框架描述

解压缩contract_go_template.tar.gz后，文件描述如下：

```sh
/home/contract_tinygo# ls -l
total 64
-rw-rw-r-- 1 1000 1000    56 Jul  2 12:45 build.sh            	# 编译脚本
-rw-rw-r-- 1 1000 1000  4149 Jul  2 12:44 bulletproofs.go		# 合约SDK基于bulletproofs的范围证明接口实现
-rw-rw-r-- 1 1000 1000 18871 Jul  2 12:44 chainmaker.go			# 合约SDK主要接口及实现
-rw-rw-r-- 1 1000 1000  4221 Jul  2 12:44 chainmaker_rs.go		# 合约SDK sql接口实现
-rw-rw-r-- 1 1000 1000 11777 May 24 13:27 easycodec.go			# 序列化工具类
-rw-rw-r-- 1 1000 1000  3585 Jul  2 12:44 main.go				# 存证示例代码
-rwxr-xr-x 1 root root 65122 Jul  6 07:22 main.wasm				# 编译成功后的wasm文件
-rw-rw-r-- 1 1000 1000  1992 Jul  2 12:44 paillier.go 			# 合约SDK基于paillier的半同态运算接口实现
```

#### 编译说明

在ChainMaker IDE中集成了编译器，可以对合约进行编译。集成的编译器是 TinyGo。用户如果手工编译，需要将 SDK 和用户编写的智能合约放入同一个文件夹，并在此文件夹的当前路径执行如下编译命令：

```sh
tinygo build -no-debug -opt=s -o name.wasm -target wasm
```

命令中 “name.wasm” 为生成的WASM 字节码的文件名，由用户自行指定。


### 部署调用合约
编译完成后，将得到一个`.wasm`格式的合约文件，可将之部署到指定到长安链上，完成合约部署。
部署合约的使用教程可详见：[部署示例合约](./部署示例合约.md) => 使用CMC工具部署/调用合约。




## 迭代器使用示例

```go

//export test_kv_iterator
func howToUseIterator() {
	ctx := NewSimContext()
    // 构造数据
	ctx.PutState("key1", "field1", "val")
	ctx.PutState("key1", "field2", "val")
	ctx.PutState("key1", "field23", "val")
	ctx.PutState("key1", "field3", "val")
	// 使用迭代器，能查出来  field1，field2，field23 三条数据
	rs, code := ctx.NewIteratorWithField("key1", "field1", "field3")
	if code == SUCCESS {
		for rs.HasNext() {
			key, field, val, code := rs.Next()
			if code == SUCCESS {
				// do something
			} else {
				rs.Close()
				ctx.ErrorResult("err")
				return
			}
		}
		rs.Close()
	}

	ctx.PutState("key2", "field1", "val")
	ctx.PutState("key3", "field2", "val")
	ctx.PutState("key33", "field23", "val")
	ctx.PutState("key4", "field3", "val")
	// 能查出来 key2，key3，key33 三条数据
	ctx.NewIterator("key2", "key4")
	// 能查出来 key3，key33 两条数据
	ctx.NewIteratorPrefixWithKey("key3")
	// 能查出来  key1 field2，key1 field23 三条数据
	ctx.NewIteratorPrefixWithKeyField("key1", "field2")

	ctx.PutStateFromKey("key5", "val")
	ctx.PutStateFromKey("key56", "val")
	ctx.PutStateFromKey("key6", "val")
	// 能查出来 key5，key56 两条数据
	ctx.NewIterator("key5", "key6")
}
```



## Go SDK API描述

### 用户与链交互接口

```go

// SimContextCommon common context
type SimContextCommon interface {
	// Arg get arg from transaction parameters, as:  arg1, code := ctx.Arg("arg1")
	Arg(key string) ([]byte, ResultCode)
	// Arg get arg from transaction parameters, as:  arg1, code := ctx.ArgString("arg1")
	ArgString(key string) (string, ResultCode)
	// Args return args
	Args() []*EasyCodecItem
	// Log record log to chain server
	Log(msg string)
	// SuccessResult record the execution result of the transaction, multiple calls will override
	SuccessResult(msg string)
	// SuccessResultByte record the execution result of the transaction, multiple calls will override
	SuccessResultByte(msg []byte)
	// ErrorResult record the execution result of the transaction. multiple calls will append. Once there is an error, it cannot be called success method
	ErrorResult(msg string)
	// CallContract cross contract call
	CallContract(contractName string, method string, param map[string][]byte) ([]byte, ResultCode)
	// GetCreatorOrgId get tx creator org id
	GetCreatorOrgId() (string, ResultCode)
	// GetCreatorRole get tx creator role
	GetCreatorRole() (string, ResultCode)
	// GetCreatorPk get tx creator pk
	GetCreatorPk() (string, ResultCode)
	// GetSenderOrgId get tx sender org id
	GetSenderOrgId() (string, ResultCode)
	// GetSenderOrgId get tx sender role
	GetSenderRole() (string, ResultCode)
	// GetSenderOrgId get tx sender pk
	GetSenderPk() (string, ResultCode)
	// GetBlockHeight get tx block height
	GetBlockHeight() (string, ResultCode)
	// GetTxId get current tx id
	GetTxId() (string, ResultCode)
	// EmitEvent emit event, you can subscribe to the event using the SDK
	EmitEvent(topic string, data ...string) ResultCode
}

// SimContext kv context
type SimContext interface {
	SimContextCommon
	// GetState get [key+"#"+field] from chain and db
	GetState(key string, field string) (string, ResultCode)
	// GetStateByte get [key+"#"+field] from chain and db
	GetStateByte(key string, field string) ([]byte, ResultCode)
	// GetStateByte get [key] from chain and db
	GetStateFromKey(key string) ([]byte, ResultCode)
	// PutState put [key+"#"+field, value] to chain
	PutState(key string, field string, value string) ResultCode
	// PutStateByte put [key+"#"+field, value] to chain
	PutStateByte(key string, field string, value []byte) ResultCode
	// PutStateFromKey put [key, value] to chain
	PutStateFromKey(key string, value string) ResultCode
	// PutStateFromKeyByte put [key, value] to chain
	PutStateFromKeyByte(key string, value []byte) ResultCode
	// DeleteState delete [key+"#"+field] to chain
	DeleteState(key string, field string) ResultCode
	// DeleteStateFromKey delete [key] to chain
	DeleteStateFromKey(key string) ResultCode
	// NewIterator range of [startKey, limitKey), front closed back open
	NewIterator(startKey string, limitKey string) (ResultSetKV, ResultCode)
	// NewIteratorWithField range of [key+"#"+startField, key+"#"+limitField), front closed back open
	NewIteratorWithField(key string, startField string, limitField string) (ResultSetKV, ResultCode)
	// NewIteratorPrefixWithKeyField range of [key+"#"+field, key+"#"+field], front closed back closed
	NewIteratorPrefixWithKeyField(key string, field string) (ResultSetKV, ResultCode)
	// NewIteratorPrefixWithKey range of [key, key], front closed back closed
	NewIteratorPrefixWithKey(key string) (ResultSetKV, ResultCode)
}


// ResultSet iterator query result
type ResultSet interface {
	// NextRow get next row,
	// sql: column name is EasyCodec key, value is EasyCodec string val. as: val := ec.getString("columnName")
	// kv iterator: key/value is EasyCodec key for "key"/"value", value type is []byte. as: k, _ := ec.GetString("key") v, _ := ec.GetBytes("value")
	NextRow() (*EasyCodec, ResultCode)
	// HasNext return does the next line exist
	HasNext() bool
	// close
	Close() (bool, ResultCode)
}

type ResultSetKV interface {
	ResultSet
	// Next return key,field,value,code
	Next() (string, string, []byte, ResultCode)
}

type SqlSimContext interface {
	SimContextCommon
	// sql method
	// ExecuteQueryOne
	ExecuteQueryOne(sql string) (*EasyCodec, ResultCode)
	ExecuteQuery(sql string) (ResultSet, ResultCode)
	// #### ExecuteUpdateSql execute update/insert/delete sql
	// ##### It is best to update with primary key
	//
	// as:
	//
	// - update table set name = 'Tom' where uniqueKey='xxx'
	// - delete from table where uniqueKey='xxx'
	// - insert into table(id, xxx,xxx) values(xxx,xxx,xxx)
	//
	// ### not allow:
	// - random methods: NOW() RAND() and so on
	// return: 1 Number of rows affected;2 result code
	ExecuteUpdate(sql string) (int32, ResultCode)
	// ExecuteDDLSql execute DDL sql, for init_contract or upgrade method. allow table create/alter/drop/truncate
	//
	// ## You must have a primary key to create a table
	// ### allow:
	// - CREATE TABLE tableName
	// - ALTER TABLE tableName
	// - DROP TABLE tableName
	// - TRUNCATE TABLE tableName
	//
	// ### not allow:
	// - CREATE DATABASE dbName
	// - CREATE TABLE dbName.tableName
	// - ALTER TABLE dbName.tableName
	// - DROP DATABASE dbName
	// - DROP TABLE dbName.tableName
	// - TRUNCATE TABLE dbName.tableName
	// not allow:
	// - random methods: NOW() RAND() and so on
	//
	ExecuteDdl(sql string) (int32, ResultCode)
} 
```



GetState

```  go
// 获取合约账户信息。该接口可从链上获取类别 “key” 下属性名为 “field” 的状态信息。
// @param key: 需要查询的key值
// @param field: 需要查询的key值下属性名为field
// @return1: 查询到的value值
// @return2: 0: success, 1: failed
func GetState(key string, field string) (string, ResultCode)
```

GetStateFromKey

```go
// 获取合约账户信息。该接口可以从链上获取类别为key的状态信息
// @param key: 需要查询的key值
// @return1: 查询到的值
// @return: 0: success, 1: failed
func GetStateFromKey(key string) ([]byte, ResultCode)
```

PutState

```go
// 写入合约账户信息。该接口可把类别 “key” 下属性名为 “filed” 的状态更新到链上。更新成功返回0，失败则返回1。
// @param key: 需要存储的key值，注意key长度不允许超过64，且只允许大小写字母、数字、下划线、减号、小数点符号
// @param field: 需要存储的key值下属性名为field，注意field长度不允许超过64，且只允许大小写字母、数字、下划线、减号、小数点符号
// @param value: 需要存储的value值，注意存储的value字节长度不能超过200
// @return: 0: success, 1: failed
func PutState(key string, field string, value string) ResultCode
```

PutStateFromKey

```go
// 写入合约账户信息。
// @param key: 需要存储的key值
// @param value: 需要存储的value值
// @return: 0: success, 1: failed
func PutStateFromKey(key string, value string) ResultCode
```

DeleteState

```go
// 删除合约账户信息。该接口可把类别 “key” 下属性名为 “name” 的状态从链上删除。
// @param key: 需要删除的key值
// @param field: 需要删除的key值下属性名为field
// @return: 0: success, 1: failed
func DeleteState(key string, field string) ResultCode {}
```

CallContract

```go
// 跨合约调用。
// @param contractName 合约名称
// @param method 合约方法
// @param param 参数
// @return 0:合约返回结果， 1：合约执行结果
func CallContract(contractName string, method string, param map[string][]byte) ([]byte, ResultCode) {}
```

Args

```go
// 该接口调用 getArgsMap() 接口，把 json 格式的数据反序列化，并将解析出的数据返还给用户。
// @return: 返回值类型为*EasyCodecItem数组
func Args() []*EasyCodecItem {}  
```

Arg

```go
// 该接口可返回属性名为 “key” 的参数的属性值。
// @param key: 获取的参数名
// @return: 获取的参数值,结果返回值
func Arg(key string) ([]byte, ResultCode) {}  
```

SuccessResult

```go
// 该接口可记录用户操作成功的信息，并将操作结果记录到链上。
// @param msg: 成功信息
func SuccessResult(msg string) {}  
```

ErrorResult

```go
// 该接口可记录用户操作失败的信息，并将操作结果记录到链上。
// @param msg: 失败信息
func ErrorResult(msg string) {}
```

LogMessage

```go
// 该接口可记录事件日志。查看方式为在链配置的log.yml中，开启vm:debug即可看到类似：gasm log>> + msg
// @param msg: 事件信息
func LogMessage(msg string) {}
```

GetCreatorOrgId

```go
// 获取合约创建者所属组织ID
// @return: 合约创建者的组织ID,结果返回值
func GetCreatorOrgId() (string, ResultCode) {}  
```

GetCreatorRole

```go
// 获取合约创建者角色
// @return: 合约创建者的角色,结果返回值
func GetCreatorRole() (string, ResultCode) {}  
```

GetCreatorPk

```go
// 获取合约创建者公钥
// @return: 合约创建者的公钥,结果返回值
func GetCreatorPk() (string, ResultCode) {} 
```

GetSenderOrgId

```go
// 获取交易发起者所属组织ID
// @return: 交易发起者的组织ID,结果返回值
func GetSenderOrgId() (string, ResultCode) {}  
```

GetSenderRole

```go
// 获取交易发起者角色
// @return: 交易发起者角色,结果返回值
func GetSenderRole() (string, ResultCode) {} 
```

GetSenderPk()

```go
// 获取交易发起者公钥
// @return 交易发起者的公钥,结果返回值
func GetSenderPk() (string, ResultCode) {}  
```

GetBlockHeight

```go
// 获取当前区块高度
// @return: 当前块高度,结果返回值
func GetBlockHeight() (string, ResultCode) {} 
```

GetTxId

```go
// 获取交易ID
// @return 交易ID,结果返回值
func GetTxId() (string, ResultCode) {}

```

EmitEvent

```go
// 发送合约事件
// @param topic: 合约事件主题
// @data ...: 可变参数,合约事件数据，参数数量不可大于16，不可小于1。
func EmitEvent(topic string, data ...string) ResultCode {}

```

NewIterator

```go
// NewIterator range of [startKey, limitKey), front closed back open
// 新建key范围迭代器，key前闭后开，即：startKey <= dbkey < limitKey
// @param startKey: 开始的key
// @param limitKey: 结束的key
// @return: 结果集游标
NewIterator(startKey string, limitKey string) (ResultSetKV, ResultCode)
// NewIteratorWithField range of [key+"#"+startField, key+"#"+limitField), front closed back open
// 新建field范围迭代器，key需相同，field前闭后开，即：key = dbdbkey and startField <= dbfield < limitField
// @param key: 固定key
// @param startField: 开始的field
// @param limitField: 结束的field
// @return: 结果集游标
NewIteratorWithField(key string, startField string, limitField string) (ResultSetKV, ResultCode)
// NewIteratorPrefixWithKey range of [key, key], front closed back closed
// 新建指定key前缀匹配迭代器，key需前缀一致，即dbkey.startWith(key)
// @param key: key前缀
// @return: 结果集游标
NewIteratorPrefixWithKey(key string) (ResultSetKV, ResultCode)
// NewIteratorPrefixWithKeyField range of [key+"#"+field, key+"#"+field], front closed back closed
// 新建指定field前缀匹配迭代器，key需相同，field前缀一致，即dbkey = key and dbfield.startWith(field)
// @param key: key前缀
// @param field: 指定field
// @return: 结果集游标
NewIteratorPrefixWithKeyField(key string, field string) (ResultSetKV, ResultCode)
```


