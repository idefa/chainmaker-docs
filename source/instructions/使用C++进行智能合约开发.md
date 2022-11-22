# 使用C++进行智能合约开发
作者：长安链团队 丁有振、韩天乐、袁太富、胡振远、董波

读者对象：本章节主要描述使用C++进行ChainMaker合约编写的方法，主要面向于使用C++进行ChainMaker的合约开发的开发者。

## 环境依赖

**操作系统**

目前仅支持Linux和MAC系统。

**软件依赖**

软件依赖表如下：

| 名称 | 版本 | 描述    | 是否必须 |
| ---- | ---- | ------- | -------- |
| GCC  | 7.3+ | C编译器 | 是       |

依赖软件安装：

Mac：brew install gcc

 Linux: 

- yum install gcc —— Ubuntu/Debian安装命令。
- apt-get install gcc —— CentOS/Redhat安装命令。	

**长安链环境准备**

准备一条支持WXVM的长安链，以及长安链CMC工具，用于将写编写好的合约，部署到链上进行测试。相关安装教程请详见：

- [部署长安链教程。](../quickstart/通过命令行体验链.md)
- [部署长安链CMC工具的教程。](../dev/命令行工具.md)

## 编写C++智能合约

### 搭建开发环境

开发者可根据ChainMaker提供的SDK开发C++合约，C++合约的SDK工程下载地址为：[chainmaker-contract-sdk-cpp](https://git.chainmaker.org.cn/chainmaker/contract-sdk-cpp)。

SDK下载完成后，开发者可根据自身习惯选择熟悉的C++编辑器或IDE。推荐使用CLion，CLion下载和安装请参见官网：https://www.jetbrains.com/clion/。

安装完成后，使用CLion打开SDK工程，通过编辑main.cc文件即可编辑自己的C++合约。


### 代码编写规则

**外部方法声明**

只有声明为外部方法的函数，才可以（被用户或其他合约）从外部调用，否则，只能用于合约内部调用。外部方法声明规则如下：

- `WASM_EXPORT`： 必须，暴露声明
- `void`： 必须，无返回值
- `method_name()`： 必须，暴露方法名称

```c++
// 示例
WASM_EXPORT void init_contract() {
    
}
```

**强制声明外部方法**

强制声明外部方法为合约必须提供且必须对外暴露的方法，有以下两个：

- `init_contract`：创建合约会自动执行该方法，无需指定方法名。
- `upgrade`： 升级合约会自动执行该方法，无需指定方法名。

```c++
// 在创建本合约时, 调用一次init方法. ChainMaker不允许用户直接调用该方法.
WASM_EXPORT void init_contract() {
    // 安装时的业务逻辑，可为空
    
}

// 在升级本合约时, 对于每一个升级的版本调用一次upgrade方法. ChainMaker不允许用户直接调用该方法.
WASM_EXPORT void upgrade() {
    // 升级时的业务逻辑，可为空
    
}
```

**获取SDK 接口上下文**

C++合约通过SDK接口上下文与链进行交互，具体信息可参考文章末尾[C++ SDK API描述](C++ SDK API描述)。

```go
//获取SDK接口上下文
Context* ctx = context();
```


### 合约示例源码展示

下文代码框内为一个C++编写的存证合约示例，该合约示例实现以下功能：

1、存储文件哈希、文件名称和该交易的ID；

2、通过文件哈希查询该条记录。

```c++
#include "chainmaker/chainmaker.h"

using namespace chainmaker;

class Counter : public Contract {
public:
    void init_contract() {}
    void upgrade() {}
    // 保存
    void save() {
        // 获取SDK 接口上下文
        Context* ctx = context();
        // 定义变量
        std::string time;
        std::string file_hash;
        std::string file_name;
        std::string tx_id;
		// 获取参数
        ctx->arg("time", time);
        ctx->arg("file_hash", file_hash);
        ctx->arg("file_name", file_name);
        ctx->arg("tx_id", tx_id);
        // 发送合约事件
        // 向topic:"topic_vx"发送2个event数据，file_hash,file_name
        ctx->emit_event("topic_vx",2,file_hash.c_str(),file_name.c_str());
		// 存储数据
        ctx->put_object("fact"+ file_hash,  tx_id+" "+time+" "+file_hash+" "+file_name);
        // 记录日志
        ctx->log("call save() result:" + tx_id+" "+time+" "+file_hash+" "+file_name);
        // 返回结果
        ctx->success(tx_id+" "+time+" "+file_hash+" "+file_name);
    }

    // 查询
    void find_by_file_hash() {
        // 获取SDK 接口上下文
    	Context* ctx = context();

		// 获取参数
        std::string file_hash;
        ctx->arg("file_hash", file_hash);
		
        // 查询数据
    	std::string value;
        ctx->get_object("fact"+ file_hash, &value);
        // 记录日志
        ctx->log("call find_by_file_hash()-" + file_hash + ",result:" + value);
        // 返回结果
        ctx->success(value);
    }

};

// 在创建本合约时, 调用一次init方法. ChainMaker不允许用户直接调用该方法.
WASM_EXPORT void init_contract() {
    Counter counter;
    counter.init_contract();
}

// 在升级本合约时, 对于每一个升级的版本调用一次upgrade方法. ChainMaker不允许用户直接调用该方法.
WASM_EXPORT void upgrade() {
    Counter counter;
    counter.upgrade();
}

WASM_EXPORT void save() {
    Counter counter;
    counter.save();
}

WASM_EXPORT void find_by_file_hash() {
    Counter counter;
    counter.find_by_file_hash();
}
```

### 编译合约

#### 搭建编译环境

开发者可使用ChainMaker已经打包好的Docker镜像编译C++合约代码，ChainMaker官方已经将容器发布至 [docker hub](https://hub.docker.com/u/chainmakerofficial)。

**拉取镜像**

请使用以下命令拉取用于编译C++合约的镜像。

```sh
docker pull chainmakerofficial/chainmaker-cpp-contract:2.1.0
```

**启动镜像**

启动镜像前，需要指定本地开发目录，用于映射为docker镜像的home目录。用于映射的本地开发目录一般为SDK工程目录，例如/data/workspace/chainmaker-contract-sdk-cpp，这样编辑开发的C++合约就可以在docker容器内的home目录直接编译了。

```sh
# 启动并进入容器，$WORK_DIR即本地工作目录
docker run -it --name chainmaker-cpp-contract -v $WORK_DIR:/home chainmakerofficial/chainmaker-cpp-contract:2.1.0 bash
# 或者先后台启动
docker run -d --name chainmaker-cpp-contract -v $WORK_DIR:/home chainmakerofficial/chainmaker-cpp-contract:2.1.0 bash -c "while true; do echo hello world; sleep 5;done"
# 再进入容器
docker exec -it chainmaker-cpp-contract bash
```

#### 编译示例合约

进入编译容器后，切换到home目录，这个home目录对应启动编译容器时映射的本地开发目录，进入后执行以下命令。

```sh
cd /home/
make clean
emmake make
```

编译完成后，将生成合约的字节码文件main.wasm

```
/home/main.wasm
```

#### SDK工程框架描述

chainmaker-contract-sdk-cpp工程的结构和文件描述如下：

- chainmaker
  - basic_iterator.cc：  迭代器实现
  - basic_iterator.h： 迭代器头文件声明
  - chainmaker.h： sdk主要接口头文件声明，详情见[SDK API描述](#sdk-api)
  - context_impl.cc：  与链交互接口实现
  - context_impl.h：  与链交互头文件声明
  - contract.cc： 合约基础工具类
  - error.h： 异常处理类
  - exports.js：  编译合约导出函数
  - safemath.h：  assert异常处理
  - syscall.cc： 与链交互入口
  - syscall.h：  与链交互头文件声明
- pb
  - contract.pb.cc：与链交互数据协议
  - contract.pb.h：与链交互数据协议头文件声明
- main.cc： 用户写合约入口
- Makefile： 常用build命令

#### 编译说明

在ChainMaker提供的Docker容器中中集成了编译器，可以对合约进行编译，集成的编译器是emcc 1.38.48版本，protobuf 使用3.7.1版本。用户如果手工编译需要先使用emcc 编译 protobuf ，编译之后执行emmake make即可。

#### 模拟运行示例合约
通过本地模拟环境运行合约(首次编译运行合约可能需要10秒左右，下面以存证作为示例)

```
# wxvm main.wasm save time 20210304 file_hash 12345678 file_name a.txt

2021-03-25 09:10:36.441      [DEBUG] [Vm]    xvm/context_service.go:257      wxvm log >>[1234567890123456789012345678901234567890123456789012345678901234] [1] call save() tx_id:
2021-03-25 09:10:36.463 [DEBUG] [Vm]    xvm/context_service.go:257      wxvm log >>[1234567890123456789012345678901234567890123456789012345678901234] [1] call save() file_hash:12345678
2021-03-25 09:10:36.464 [DEBUG] [Vm]    xvm/context_service.go:257      wxvm log >>[1234567890123456789012345678901234567890123456789012345678901234] [1] call save() file_name:a.txt
2021-03-25 09:10:36.465 [DEBUG] [Vm]    xvm/context_service.go:257      wxvm log >>[1234567890123456789012345678901234567890123456789012345678901234] [1] put success: a.txt 12345678
2021-03-25 09:10:36.466 [DEBUG] [Vm]    xvm/context_service.go:257      wxvm log >>[1234567890123456789012345678901234567890123456789012345678901234] [1] save====================================end
2021-03-25 09:10:36.467 [DEBUG] [Vm]    xvm/context_service.go:257      wxvm log >>[1234567890123456789012345678901234567890123456789012345678901234] [1] 
2021-03-25 09:10:36.467 [DEBUG] [Vm]    xvm/context_service.go:257      wxvm log >>[1234567890123456789012345678901234567890123456789012345678901234] [1] result:  a.txt 12345678
2021-03-25 09:10:36.469 [INFO]  [Vm] @chain01   main/main.go:31 contractResult :result:" a.txt 12345678"
```

### 部署调用合约
编译完成后，将得到一个`.wasm`格式的合约文件，可将之部署到指定到长安链上，完成合约部署。
部署合约的使用教程可详见：[部署示例合约](./部署示例合约.md)。
  

## C++ SDK API描述

<span id="sdk-api">arg</span>

```c++
// 该接口可返回属性名为 “name” 的参数的属性值。
// @param name: 要获取值的参数名称
// @param value: 获取的参数值
// @return: 是否成功
bool arg(const std::string& name, std::string& value){}
```

需要注意的是通过arg接口返回的参数，全都都是字符串，合约开发者有必要将其他数据类型的参数与字符串做转换，包括atoi、itoa、自定义序列化方式等。

get_object

```c++
// 获取key为"key"的值
// @param key: 获取对象的key
// @param value: 获取的对象值
// @return: 是否成功
bool get_object(const std::string& key, std::string* value){}
```

put_object

```c++
// 存储key为"key"的值
// @param key: 存储的对象key，注意key长度不允许超过64，且只允许大小写字母、数字、下划线、减号、小数点符号
// @param value: 存储的对象值install
// @return: 是否成功
bool put_object(const std::string& key, const std::string& value){}
```

delete_object

```c++
// 删除key为"key"的值
// @param key: 删除的对象key
// @return: 是否成功
bool delete_object(const std::string& key) {}
```

emit_event

```c++
// 发送合约事件
// @param topic: 合约事件主题
// @data_amount: 合约事件数据数量(data)，data_amount的值必须要和data数量一致，最多不可大于16，最少不可小于1,不可为空
// @data ...: 可变参数合约事件数据，数量与data_amount一致。
bool emit_event(const std::string &topic, int data_amount, const std::string data, ...)
```

success

```c++
// 返回成功的结果
// @param body: 成功信息
void success(const std::string& body) {}
```

error

```c++
// 返回失败结果
// @param body: 失败信息
void error(const std::string& body) {}
```

call

```c++
// 跨合约调用
// @param contract: 合约名称
// @param method: 合约方法
// @param args: 调用合约的参数
// @param resp: 调用合约的响应
// @return: 是否成功
bool call(const std::string &contract,
                          const std::string &method,
                          EasyCodecItems *args,
                          std::string *resp){}
```

log

```c++
// 输出日志事件。查看方式为在链配置的log.yml中，开启vm:debug即可看到类似：wxvm log>> + msg
// @param body: 事件信息
void log(const std::string& body) {}
```

