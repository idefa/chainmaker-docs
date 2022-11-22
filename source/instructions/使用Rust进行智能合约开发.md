# 使用Rust进行智能合约开发
作者：长安链团队  蔡志宏、韩天乐、袁太富

读者对象：本章节主要描述使用Rust进行ChainMaker合约编写的方法，主要面向于使用Rust进行ChainMaker的合约开发的开发者。

## 环境依赖

使用Rust开发用于ChainMaker的wasm合约,需要安装Rust开发环境，并将 wasm32-unknown-unknown（目标平台）加入到Rust开发环境的工具链中。

rust 安装及教程请参考：[rust 官网](https://www.rust-lang.org/)

安装指令如下：

```sh
# 安装 Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# 让 Rust 的环境变量生效 
source "$HOME/.cargo/env"
# 加入 wasm32-unknown-unknown
rustup target add wasm32-unknown-unknown
```


## 编写Rust智能合约

### 搭建开发环境
1）推荐使用 GoLand集成开发环境 + Rust插件

2）下载 contracts-rust 示例工程
```shell
git clone git@git.code.tencent.com:ChainMaker/contracts/contracts-rust.git
```

3）进入下载的Rust工程
```shell
cd contracts-rust/fact
```

### 代码编写规则

**对链暴露方法写法为：**

- #[no_mangle] 表示方法名编译后是固定的，不写会生成 _ZN4rustfn1_34544tert54grt5 类似的混淆名
- pub extern "C"
- method_name(): 不可带参数，无返回值

```rust
#[no_mangle]// no_mangle注解，表明对外暴露方法名称不可变
pub extern "C" fn init_contract() { // pub extern "C" 集成C
    let ctx = &mut sim_context::get_sim_context();
    // do something 
    ctx.ok("contract init success.".as_bytes());
}
```

**其中init_contract、upgrade方法必须有且对外暴露**

- init_contract：创建合约会执行该方法
- upgrade： 升级合约会执行该方法

```rust
// 安装合约时会执行此方法。ChainMaker不允许用户直接调用该方法。
#[no_mangle]
pub extern "C" fn init_contract() {
    let ctx = &mut sim_context::get_sim_context();
    // do something
    ctx.ok("contract init success.".as_bytes());
}

// 升级合约时会执行此方法。ChainMaker不允许用户直接调用该方法。
#[no_mangle]
pub extern "C" fn upgrade() {
    let ctx = &mut sim_context::get_sim_context();
    // do something
    ctx.ok("contract upgrade success.".as_bytes());
}
```

**获取与链交互的上下文sim_context**

1、在`Cargo.toml`中引入对于 contract-sdk-rust 项目的依赖
```toml
[dependencies]
contract_sdk_rust = { git = "https://git.code.tencent.com/ChainMaker/contract-sdk-rust.git", branch="v2.1.0_refer" }
```

2、在使用时引入sim_context

```rust
use contract_sdk_rust::sim_context;
use contract_sdk_rust::sim_context::SimContext;
use contract_sdk_rust::easycodec::*;

fn method_name() {
    // 获取上下文
    let ctx = &mut sim_context::get_sim_context();
    
}
```

### 存证合约示例源码展示

**存证合约示例：fact.rs** 实现如下两个功能

1、存储文件哈希和文件名称和时间。

2、通过文件哈希查询该条记录

```rust
use contract_sdk_rust::sim_context;
use contract_sdk_rust::sim_context::SimContext;
use contract_sdk_rust::easycodec::*;

// 安装合约时会执行此方法，必须
#[no_mangle]
pub extern "C" fn init_contract() {
    sim_context::log("init_contract");
    let ctx = &mut sim_context::get_sim_context();
    
    // 安装时的业务逻辑，内容可为空
    
    ctx.ok("contract init success.".as_bytes());
}

// 升级合约时会执行此方法，必须
#[no_mangle]
pub extern "C" fn upgrade() {
    sim_context::log("upgrade");
    let ctx = &mut sim_context::get_sim_context();
    
    // 升级时的业务逻辑，内容可为空

    ctx.ok("contract upgrade success".as_bytes());
}

struct Fact {
    file_hash: String,
    file_name: String,
    time: i32,
    ec: EasyCodec,
}

impl Fact {
    fn new_fact(file_hash: String, file_name: String, time: i32) -> Fact {
        let mut ec = EasyCodec::new();
        ec.add_string("file_hash", file_hash.as_str());
        ec.add_string("file_name", file_name.as_str());
        ec.add_i32("time", time);
        Fact {
            file_hash,
            file_name,
            time,
            ec,
        }
    }

    fn get_emit_event_data(&self) -> Vec<String> {
        let mut arr: Vec<String> = Vec::new();
        arr.push(self.file_hash.clone());
        arr.push(self.file_name.clone());
        arr.push(self.time.to_string());
        arr
    }

    fn to_json(&self) -> String {
        self.ec.to_json()
    }

    fn marshal(&self) -> Vec<u8> {
        self.ec.marshal()
    }

    fn unmarshal(data: &Vec<u8>) -> Fact {
        let ec = EasyCodec::new_with_bytes(data);
        Fact {
            file_hash: ec.get_string("file_hash").unwrap(),
            file_name: ec.get_string("file_name").unwrap(),
            time: ec.get_i32("time").unwrap(),
            ec,
        }
    }
}

// save 保存存证数据
#[no_mangle]
pub extern "C" fn save() {
    // 获取上下文
    let ctx = &mut sim_context::get_sim_context();

    // 获取传入参数
    let file_hash = ctx.arg_as_utf8_str("file_hash");
    let file_name = ctx.arg_as_utf8_str("file_name");
    let time_str = ctx.arg_as_utf8_str("time");

    // 构造结构体
    let r_i32 = time_str.parse::<i32>();
    if r_i32.is_err() {
        let msg = format!("time is {:?} not int32 number.", time_str);
        ctx.log(&msg);
        ctx.error(&msg);
        return;
    }
    let time: i32 = r_i32.unwrap();
    let fact = Fact::new_fact(file_hash, file_name, time);

    // 事件
    ctx.emit_event("topic_vx", &fact.get_emit_event_data());

    // 序列化后存储
    ctx.put_state(
        "fact_ec",
        fact.file_hash.as_str(),
        fact.marshal().as_slice(),
    );
}

// find_by_file_hash 根据file_hash查询存证数据
#[no_mangle]
pub extern "C" fn find_by_file_hash() {
    // 获取上下文
    let ctx = &mut sim_context::get_sim_context();

    // 获取传入参数
    let file_hash = ctx.arg_as_utf8_str("file_hash");

    // 校验参数
    if file_hash.len() == 0 {
        ctx.log("file_hash is null");
        ctx.ok("".as_bytes());
        return;
    }

    // 查询
    let r = ctx.get_state("fact_ec", &file_hash);

    // 校验返回结果
    if r.is_err() {
        ctx.log("get_state fail");
        ctx.error("get_state fail");
        return;
    }
    let fact_vec = r.unwrap();
    if fact_vec.len() == 0 {
        ctx.log("None");
        ctx.ok("".as_bytes());
        return;
    }

    // 查询
    let r = ctx.get_state("fact_ec", &file_hash).unwrap();
    let fact = Fact::unmarshal(&r);
    let json_str = fact.to_json();

    // 返回查询结果
    ctx.ok(json_str.as_bytes());
    ctx.log(&json_str);
}
```



### 合约SDK接口描述
长安链提供Rust合约与链交互的相关接口，代码存在单独的项目（https://git.code.tencent.com/ChainMaker/contract-sdk-rust）中。写合约时可直接指定依赖，并进行引用，具体信息可参考文章末尾"接口描述章节"。


### 编译示例合约

```shell
cd contracts_rust/fact/
cargo build --release --target=wasm32-unknown-unknown
```

生成合约的字节码文件在

```shell
contracts_rust/fact/target/wasm32-unknown-unknown/release/fact.wasm
```

#### 示例合约框架描述

```shell
contract-sdk-rust$ tree -I target

├── Cargo.lock # 依赖版本信息
├── Cargo.toml # 项目配置及依赖，参考：https://rustwasm.github.io/wasm-pack/book/cargo-toml-configuration.html
├── README.md  # 编译环境说明
├── src
│   ├── fact.rs			# 存证示例代码
│   └── lib.rs          # 程序入口
```

### 部署调用合约
编译完成后，将得到一个`.wasm`格式的合约文件，可将之部署到指定到长安链上，完成合约部署。
部署合约的使用教程可详见：[部署示例合约](./部署示例合约.md)。

## 迭代器使用示例

[点击此处查看接口说明](#new_iterator_interface)

使用示例如下：

```rust
#[no_mangle]
pub extern "C" fn how_to_use_iterator() {
    let ctx = &mut sim_context::get_sim_context();

    // 构造数据
    ctx.put_state("key1", "field1", "val".as_bytes());
    ctx.put_state("key1", "field2", "val".as_bytes());
    ctx.put_state("key1", "field23", "val".as_bytes());
    ctx.put_state("key1", "field3", "val".as_bytes());
    // 使用迭代器，能查出来  field1，field2，field23 三条数据
    let r = ctx.new_iterator_with_field("key1", "field1", "field3");
    if r.is_ok() {
        let rs = r.unwrap();
        // 遍历
        while rs.has_next() {
            // 获取下一行值
            let row = rs.next_row().unwrap();
            let key = row.get_string("key").unwrap();
            let field = row.get_bytes("field");
            let val = row.get_bytes("value");
            // do something
        }
        // 关闭游标
        rs.close();
    }

    ctx.put_state("key2", "field1", "val".as_bytes());
    ctx.put_state("key3", "field2", "val".as_bytes());
    ctx.put_state("key33", "field2", "val".as_bytes());
    ctx.put_state("key4", "field3", "val".as_bytes());
    // 能查出来 key2，key3，key33 三条数据
    ctx.new_iterator("key2", "key4");
    // 能查出来 key3，key33 两条数据
    ctx.new_iterator_prefix_with_key("key3");
    // 能查出来  field2，field23 三条数据
    ctx.new_iterator_prefix_with_key_field("key1", "field2");
    
    
    ctx.put_state_from_key("key5","val".as_bytes());
    ctx.put_state_from_key("key56","val".as_bytes());
    ctx.put_state_from_key("key6","val".as_bytes());
    // 能查出来 key5，key56 两条数据
    ctx.new_iterator("key5", "key6");
}
```





## Rust SDK API描述 <span id="api"></span>

### 下载 contract-sdk-rust 项目代码
```shell
git clone git@git.code.tencent.com:ChainMaker/contract-sdk-rust.git
```

### 项目文件说明
```shell
tree -I target 

├── Cargo.lock # 依赖版本信息
├── Cargo.toml # 项目配置及依赖，参考：https://rustwasm.github.io/wasm-pack/book/cargo-toml-configuration.html
├── Makefile   # build一个wasm文件
├── README.md  # 编译环境说明
├── src
│   ├── easycodec.rs                # 序列化工具类
│   ├── lib.rs                      # 程序入口
│   ├── sim_context.rs              # 合约SDK主要接口及实现
│   ├── sim_context_bulletproofs.rs # 合约SDK基于bulletproofs的范围证明接口实现
│   ├── sim_context_paillier.rs     # 合约SDK基于paillier的半同态运算接口实现
│   ├── sim_context_rs.rs           # 合约SDK sql接口实现
│   └── vec_box.rs                  # 内存管理类
```

### 内置链交互接口

用于链与SDK数据交互。

```rust
// 申请size大小内存，返回该内存的首地址
pub extern "C" fn allocate(size: usize) -> i32 {}
// 释放某地址
pub extern "C" fn deallocate(pointer: *mut c_void) {}
// 获取SDK运行时环境
pub extern "C" fn runtime_type() -> i32 { 0 }
```


### 用户与链交互接口

```rust

/// SimContext is a interface with chainmaker interaction
pub trait SimContext {
    // common method
    fn call_contract(
        &self,
        contract_name: &str,
        method: &str,
        param: EasyCodec,
    ) -> Result<Vec<u8>, result_code>;
    fn ok(&self, value: &[u8]) -> result_code;
    fn error(&self, body: &str) -> result_code;
    fn log(&self, msg: &str);
    fn arg(&self, key: &str) -> Result<Vec<u8>, String>;
    fn arg_as_utf8_str(&self, key: &str) -> String;
    fn args(&self) -> &EasyCodec;
    fn get_creator_org_id(&self) -> String;
    fn get_creator_pub_key(&self) -> String;
    fn get_creator_role(&self) -> String;
    fn get_sender_org_id(&self) -> String;
    fn get_sender_pub_key(&self) -> String;
    fn get_sender_role(&self) -> String;
    fn get_block_height(&self) -> u64;
    fn get_tx_id(&self) -> String;
    fn emit_event(&mut self, topic: &str, data: &Vec<String>) -> result_code;
    // paillier
    fn get_paillier_sim_context(&self) -> Box<dyn PaillierSimContext>;
    // bulletproofs
    fn get_bulletproofs_sim_context(&self) -> Box<dyn BulletproofsSimContext>;
    // sql
    fn get_sql_sim_context(&self) -> Box<dyn SqlSimContext>;

    // KV method
    fn get_state(&self, key: &str, field: &str) -> Result<Vec<u8>, result_code>;
    fn get_state_from_key(&self, key: &str) -> Result<Vec<u8>, result_code>;
    fn put_state(&self, key: &str, field: &str, value: &[u8]) -> result_code;
    fn put_state_from_key(&self, key: &str, value: &[u8]) -> result_code;
    fn delete_state(&self, key: &str, field: &str) -> result_code;
    fn delete_state_from_key(&self, key: &str) -> result_code;

    /// new_iterator range of [startKey, limitKey), front closed back open
    fn new_iterator(
        &self,
        start_key: &str,
        limit_key: &str,
    ) -> Result<Box<dyn ResultSet>, result_code>;

    /// new_iterator_with_field range of [key+"#"+startField, key+"#"+limitField), front closed back open
    fn new_iterator_with_field(
        &self,
        key: &str,
        start_field: &str,
        limit_field: &str,
    ) -> Result<Box<dyn ResultSet>, result_code>;

    /// new_iterator_prefix_with_key_field range of [key+"#"+field, key+"#"+field], front closed back closed
    fn new_iterator_prefix_with_key_field(
        &self,
        key: &str,
        field: &str,
    ) -> Result<Box<dyn ResultSet>, result_code>;

    /// new_iterator_prefix_with_key range of [key, key], front closed back closed
    fn new_iterator_prefix_with_key(&self, key: &str) -> Result<Box<dyn ResultSet>, result_code>;
}


pub trait SqlSimContext {
    fn execute_query_one(&self, sql: &str) -> Result<EasyCodec, result_code>;
    fn execute_query(&self, sql: &str) -> Result<Box<dyn ResultSet>, result_code>;

    /// #### ExecuteUpdateSql execute update/insert/delete sql
    /// ##### It is best to update with primary key
    ///
    /// as:
    ///
    /// - update table set name = 'Tom' where uniqueKey='xxx'
    /// - delete from table where uniqueKey='xxx'
    /// - insert into table(id, xxx,xxx) values(xxx,xxx,xxx)
    ///
    /// ### not allow:
    /// - random methods: NOW() RAND() and so on
    fn execute_update(&self, sql: &str) -> Result<i32, result_code>;

    /// ExecuteDDLSql execute DDL sql, for init_contract or upgrade method. allow table create/alter/drop/truncate
    ///
    /// ## You must have a primary key to create a table
    /// ### allow:     
    /// - CREATE TABLE tableName
    /// - ALTER TABLE tableName
    /// - DROP TABLE tableName   
    /// - TRUNCATE TABLE tableName
    ///
    /// ### not allow:
    /// - CREATE DATABASE dbName
    /// - CREATE TABLE dbName.tableName
    /// - ALTER TABLE dbName.tableName
    /// - DROP DATABASE dbName   
    /// - DROP TABLE dbName.tableName   
    /// - TRUNCATE TABLE dbName.tableName
    /// not allow:
    /// - random methods: NOW() RAND() and so on
    ///
    fn execute_ddl(&self, sql: &str) -> Result<i32, result_code>;
}


pub trait PaillierSimContext {
    // Paillier method
    fn add_ciphertext(
        &self,
        pubkey: Vec<u8>,
        ciphertext1: Vec<u8>,
        ciphertext2: Vec<u8>,
    ) -> Result<Vec<u8>, result_code>;
    fn add_plaintext(
        &self,
        pubkey: Vec<u8>,
        ciphertext: Vec<u8>,
        plaintext: &str,
    ) -> Result<Vec<u8>, result_code>;
    fn sub_ciphertext(
        &self,
        pubkey: Vec<u8>,
        ciphertext1: Vec<u8>,
        ciphertext2: Vec<u8>,
    ) -> Result<Vec<u8>, result_code>;
    fn sub_plaintext(
        &self,
        pubkey: Vec<u8>,
        ciphertext: Vec<u8>,
        plaintext: &str,
    ) -> Result<Vec<u8>, result_code>;
    fn num_mul(
        &self,
        pubkey: Vec<u8>,
        ciphertext: Vec<u8>,
        plaintext: &str,
    ) -> Result<Vec<u8>, result_code>;
}
/// BulletproofsSimContext is the trait that wrap the bulletproofs method
pub trait BulletproofsSimContext {
    /// Compute a commitment to x + y from a commitment to x without revealing the value x, where y is a scalar
    ///
    /// # Arguments
    ///
    /// * `commitment` - C = xB + rB'
    /// * `num` - the value y
    ///
    /// # return
    ///
    /// * `return1` - the new commitment to x + y: C' = (x + y)B + rB'
    ///
    fn pedersen_add_num(&self, commitment: Vec<u8>, num: &str) -> Result<Vec<u8>, result_code>;

    /// Compute a commitment to x + y from commitments to x and y, without revealing the value x and y
    ///
    /// # Arguments
    ///
    /// * `commitment1` - commitment to x: Cx = xB + rB'
    /// * `commitment2` - commitment to y: Cy = yB + sB'
    ///
    /// # return
    ///
    /// * `return1` - commitment to x + y: C = (x + y)B + (r + s)B'
    ///
    fn pedersen_add_commitment(
        &self,
        commitment1: Vec<u8>,
        commitment2: Vec<u8>,
    ) -> Result<Vec<u8>, result_code>;

    /// Compute a commitment to x - y from a commitment to x without revealing the value x, where y is a scalar
    ///
    /// # Arguments
    ///
    /// * `commitment1` - C = xB + rB'
    /// * `num` - the value y
    ///
    /// # return
    ///
    /// * `return1` - the new commitment to x - y: C' = (x - y)B + rB'
    fn pedersen_sub_num(&self, commitment: Vec<u8>, num: &str) -> Result<Vec<u8>, result_code>;

    /// Compute a commitment to x - y from commitments to x and y, without revealing the value x and y
    ///
    /// # Arguments
    ///
    /// * `commitment1` - commitment to x: Cx = xB + rB'
    /// * `commitment2` - commitment to y: Cy = yB + sB'
    ///
    /// # return
    ///
    /// * `return1` - commitment to x - y: C = (x - y)B + (r - s)B'
    fn pedersen_sub_commitment(
        &self,
        commitment1: Vec<u8>,
        commitment2: Vec<u8>,
    ) -> Result<Vec<u8>, result_code>;

    /// Compute a commitment to x * y from a commitment to x and an integer y, without revealing the value x and y
    ///
    /// # Arguments
    ///
    /// * `commitment1` - commitment to x: Cx = xB + rB'
    /// * `num` - integer value y
    ///
    /// # return
    ///
    /// * `return1` - commitment to x * y: C = (x * y)B + (r * y)B'
    fn pedersen_mul_num(&self, commitment: Vec<u8>, num: &str) -> Result<Vec<u8>, result_code>;

    /// Verify the validity of a proof
    ///
    /// # Arguments
    ///
    /// * `proof` - the zero-knowledge proof proving the number committed in commitment is in the range [0, 2^64)
    /// * `commitment` - commitment bindingly hiding the number x
    ///
    /// # return
    ///
    /// * `return1` - true on valid proof, false otherwise
    fn verify(&self, proof: Vec<u8>, commitment: Vec<u8>) -> Result<Vec<u8>, result_code>;
}

```



get_state

```rust
// 获取合约账户信息。该接口可从链上获取类别 “key” 下属性名为 “field” 的状态信息。
// @param key: 需要查询的key值
// @param field: 需要查询的key值下属性名为field
// @return: 查询到的value值，及错误代码 0: success, 1: failed
fn get_state(&self, key: &str, field: &str) -> Result<Vec<u8>, result_code>;
```

get_state_from_key

```rust
// 获取合约账户信息。该接口可以从链上获取类别为key的状态信息
// @param key: 需要查询的key值
// @return1: 查询到的值
// @return2:  0: success, 1: failed
fn get_state_from_key(&self, key: &str) -> Result<Vec<u8>, result_code>;
```

put_state

```rust
// 写入合约账户信息。该接口可把类别 “key” 下属性名为 “filed” 的状态更新到链上。更新成功返回0，失败则返回1。
// @param key: 需要存储的key值
// @param field: 需要存储的key值下属性名为field
// @param value: 需要存储的value值
// @return: 0: success, 1: failed
fn put_state(&self, key: &str, field: &str, value: &[u8]) -> result_code;
```

put_state_from_key

```rust
// 写入合约账户信息。
// @param key: 需要存储的key值
// @param value: 需要存储的value值
// @return: 0: success, 1: failed
fn put_state_from_key(&self, key: &str, value: &[u8]) -> result_code;
```

delete_state

```rust 
// 删除合约账户信息。该接口可把类别 “key” 下属性名为 “name” 的状态从链上删除。
// @param key: 需要删除的key值
// @param field: 需要删除的key值下属性名为field
// @return: 0: success, 1: failed
fn delete_state(&self, key: &str, field: &str) -> result_code;
```

delete_state_from_key

```rust
// 删除合约账户信息。该接口可把类别 “key” 下属性名为 “name” 的状态从链上删除。
// @param key: 需要删除的key值
// @return: 0: success, 1: failed
fn delete_state_from_key(&self, key: &str) -> result_code;
```

call_contract

```rust
// 跨合约调用
// @param contract_name: 合约名称
// @param method: 合约方法
// @param EasyCodec: 合约参数
fn call_contract(&self, contract_name: &str, method: &str, param: EasyCodec) -> Result<Vec<u8>, result_code>;
```

args

```rust
// @return: EasyCodec
fn args(&self) -> &EasyCodec;
```

arg

```rust
// 该接口可返回属性名为 “key” 的参数的属性值。
// @param key: 获取的参数名
// @return: 获取的参数值 或 错误信息。当未传该key的值时，报错param not found
fn arg(&self, key: &str) -> Result<Vec<u8>, String>;
```

ok

```rust
// 该接口可记录用户操作成功的信息，并将操作结果记录到链上。
// @param body: 成功返回的信息
fn ok(&self, value: &[u8]) -> result_code;
```

error

```rust
// 该接口可记录用户操作失败的信息，并将操作结果记录到链上。
// @param body: 失败信息
fn error(&self, body: &str) -> result_code;
```

log

```rust
// 该接口可记录事件日志。查看方式为在链配置的log.yml中，开启vm:debug即可看到类似：wasmer log>> + msg
// @param msg: 事件信息
fn log(&self, msg: &str);
```

get_creator_org_id

```rust
// 获取合约创建者所属组织ID
// @return: 合约创建者的组织ID
fn get_creator_org_id(&self) -> String;
```

get_creator_role

```rust
// 获取合约创建者角色
// @return: 合约创建者的角色
fn get_creator_role(&self) -> String;
```

get_creator_pub_key

```rust
// 获取合约创建者公钥
// @return: 合约创建者的公钥的SKI
fn get_creator_pub_key(&self) -> String;
```

get_sender_org_id

```rust
// 获取交易发起者所属组织ID
// @return: 交易发起者的组织ID
fn get_sender_org_id(&self) -> String;
```

get_sender_role

```rust
// 获取交易发起者角色
// @return: 交易发起者角色
fn get_sender_role(&self) -> String;
```

get_sender_pub_key()

```rust
// 获取交易发起者公钥
// @return 交易发起者的公钥的SKI
fn get_sender_pub_key(&self) -> String;
```

get_block_height

```rust
// 获取当前区块高度
// @return: 当前块高度
fn get_block_height(&self) -> i32;
```

get_tx_id

```rust
// 获取交易ID
// @return 交易ID
fn get_tx_id(&self) -> String;
```

emit_event

```rust
// 发送合约事件
// @param topic: 合约事件主题
// @data: 合约事件数据，vertor中事件数据个数不可大于16，不可小于1
fn emit_event(&mut self, topic: &str, data: &Vec<String>) -> result_code;
```

<span id="new_iterator_interface">new_iterator</span>

```rust
/// new_iterator range of [startKey, limitKey), front closed back open
// 新建key范围迭代器，key前闭后开，即：start_key <= dbkey < limit_key
// @param start_key: 开始的key
// @param limit_key: 结束的key
// @return: 结果集游标
fn new_iterator(&self,start_key: &str,limit_key: &str) -> Result<Box<dyn ResultSet>, result_code>;

/// new_iterator_with_field range of [key+"#"+startField, key+"#"+limitField), front closed back open
// 新建field范围迭代器，key需相同，field前闭后开，即：key = dbdbkey and start_field <= dbfield < limit_field
// @param key: 固定key
// @param start_field: 开始的field
// @param limit_field: 结束的field
// @return: 结果集游标
fn new_iterator_with_field(&self, key: &str, start_field: &str, limit_field: &str) -> Result<Box<dyn ResultSet>, result_code>;

/// new_iterator_prefix_with_key range of [key, key], front closed back closed
// 新建指定key前缀匹配迭代器，key需前缀一致，即dbkey.startWith(key)
// @param key: key前缀
// @return: 结果集游标
fn new_iterator_prefix_with_key(&self, key: &str) -> Result<Box<dyn ResultSet>, result_code>;

/// new_iterator_prefix_with_key_field range of [key+"#"+field, key+"#"+field], front closed back closed
// 新建指定field前缀匹配迭代器，key需相同，field前缀一致，即dbkey = key and dbfield.startWith(field)
// @param key: key前缀
// @return: 结果集游标
fn new_iterator_prefix_with_key_field(&self, key: &str, field: &str) -> Result<Box<dyn ResultSet>, result_code>;

```

