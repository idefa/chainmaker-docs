# 版本迭代-长安链·ChainMaker ChangeLog

## v2.3.0_alpha

```
新增流水线共识算法：MaxBFT正式版
新增SDF接口硬件加密机支持
新增liquid中继功能
新增liquid NAT穿透（NAT traversal）功能
新增SDK-Go提供Gas预估功能
新增交易池模式：normal、batch
新增交易池新特性（Dump、Recover、Query、Retry机制、Clean机制）
新增native合约添加地址功能，可通过地址可查询系统合约
新增合约批量getState接口
新增同步模块提供主动关闭/重启请求服务的功能
新增查询接口：查询当前权限列表
新增整个区块在一笔交易中统一扣费
新增各模块日志可独立配置
新增国密TLS双证书体系
新增跨不同语言的合约调用
新增docker go合约SDK支持StoreMap
新增创建合约时除使用name进行存储合约外，同时生成合约地址并按地址存储
新增各合约语言的示例合约
重构TBFT，独立TBFT共识仓库，优化TBFT消息机制
重构docker-go虚拟机引擎
重构内部模块回调机制
重构项目结构调整，所有个人Fork的第三方包迁移到工蜂上
优化同步模块性能，默认快速同步并且不需要进行交易查重
优化共识verify阶段增加多交易时间戳的超时校验
优化默认地址格式兼容以太坊
优化Docker-vm-go初始化函数扣费计算改为固定值
优化solidity合约revert时可携带合约预定义的错误message
优化取消solidtiy合约的method检查
优化TBFT可去除含随机函数合约交易
优化SDK和cmc查询区块和交易时支持截断长度太长的Value
优化docker go合约内日志输出接口增加级别功能
优化跨合约调用进程复用
优化sdk-go、sdk-java同步获取交易结果方式，从轮询改为订阅
优化存储模块增加Slow Log
优化docker go容器支持重启
优化getstate时间的统计
优化flower节点验证DAG一致性逻辑
修复交易失败不扣除gas费问题
修复ac模块自定义权限删除不生效问题
修复wasmer不支持windows问题
修复跨合约调用循环调用时，导致阻塞的问题
修复创建合约报合约名非十六进制字符串问题
修复TBFT重启，导致共识状态不对，共识卡住的问题
修复TBFT删除节点可能导致panic的问题
修复查询不到带读写集区块的问题
修复windows启用文件系统存储失败问题
修复部署合约与跨合约调用的事件丢失问题
修复CA模式下gas充值问题
修复rust合约重复升级一个合约1000次以上，会有一定概率出现panic异常的问题
修复tbft在round为1时，若重启节点可能会导致卡住，无法出块
```



