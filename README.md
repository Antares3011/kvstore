# Kvstore

## 简介
以跳表为存储引擎, 以基于协程的网络服务基础设计的KV存储项目, 支持基于字符串的KV数据库的创建和编辑.

## 功能特性
- 存储引擎: 支持数组, 红黑树, 哈希表, 跳表
- 网络服务: 基于协程(Ntyco)开发的网络服务, 实现异步网络io.
- 全量持久化: 通过SAVE操作触发, 并将内存中数据全量落盘到二进制快照文件中, 加载数据时解析二进制文件.
- 增量持久化: 向日志文件中追加用户对数据库的修改操作, 加载数据时解析并回放日志.
- 分级页式空闲链表内存池: 对 8 B 至 4KB 的对象采用分级(10级)页内分配, 对更大的对象采用页对齐的连续空间分配, 并通过空闲链表实现回收与相邻块合并.
- RESP2: 支持Redis序列化协议
- 支持多指令/不限消息长度: 客户端发送RESP2编码后拼接的多命令/超长字节流, 服务端通过双缓冲区机制, 持续将recv到的临时缓冲区的内容搬运到消息缓冲区并动态扩容, 同时解析并消费消息缓冲区中的完整消息.
- 主从同步: 采用「首次同步」+「增量同步」机制, 

## 快速开始

### 编译
```
cd kvstore      #进入项目目录
make            #编译kvstore程序/客户端程序
```
编译时提供三种内存池(custom, system, jemalloc), 可以通过替换MEMORY的值实现, 其中custom表示使用自定义内存池, system表示使用libc提供的内存池, jemalloc表示使用jemalloc

### 启动主节点
修改 /kvsotre/config/config.ini 中的**ROLE**的值为**MASTER**
```
sudo ./kvstore
```
### 启动从节点
修改 /kvsotre/config/config.ini 中的**ROLE**的值为**SLAVE**
```
sudo ./kvstore
```
### 节点属性配置
通过修改 /kvsotre/config/config.ini 修改节点属性, 包括**主节点ip, 开放端口, 持久化文件路径, 节点角色, 持久化方式, 存储引擎**

### 客户端操作
```
./kvs_client <ip> <port> #开启客户端并连接节点, 输入命令+空行+回车发送一条完整的命令

SET "key" "value"        #设置键值对
GET "key"                #查找Key
MOD "key"  "value"       #修改键值对
DEL "key"                #删除key所在的键值对
SAVE                     #全量持久化
```
## 性能测试
### 编译测试文件
```
cd kvstore/test
make
```
建议以管理员模式运行以下测试脚本

#### 1.对比不同引擎下的qps(多次测试取平均值)

##### 测试方案
将10w条连续数值每条由原始命令进行编码→完整发送, 发送完成后, 校验数据一致性, 记录set操作qps\get操作qps到kvstore/test/10w_qps_result.log中
```
cd kvstore/test
bash run_10w_qps_test.sh
```
##### 测试结果
|method|ARRAY|RBTREE|HASH|SKIPTABLE|
|---|---|---|---|---|
|set_qps|3470|25043|16313|16946|
|get_qps|4730|27624|16342|16725|

#### 2.全量持久化测试

##### 测试方案
将10w条连续数值作为kv插入主节点后保存快照, 主节点宕机后, 重新启动并加载快照, 随后校验一致性
```
cd kvstore/test
bash run_rdb_test.sh
```

#### 3.增量持久化测试

##### 测试方案
将10w条连续数值作为kv插入主节点, 主节点宕机后, 重新启动并加载操作日志, 随后校验一致性
```
cd kvstore/test
bash run_aof_test.sh
```

#### 4.不同内存方案对比

##### 测试方案
保持其他参数不变, 对比自定义内存池/libc内存方案/jemalloc在插入10w条连续数据时, 节点的虚拟内存/物理内存的起始值/峰值/终止值和qps
```
cd kvstore/test
bash run_mem_test.sh
```
##### 测试结果
|method|Start VmRSS|Peak VmRSS|End VmRSS|Start VmSize|Peak VmSize|End VmSize|qps|
|---|---|---|---|---|---|---|---|
|custome|2308|21068|21068|132656|132796|132788|17001|
|malloc|2256|19072|19072|91688|107272|107272|25322|
|jemalloc|5648|16224|16224|50528|59752|59744|20648|


#### 5.主从同步测试

##### 测试方案
分两次插入共计10w条连续数据, 首先启动主节点并插入前5w条数据, 从节点连接主节点进行首次同步, 随后向主节点插入后5w条数据(增量同步), 随后向从节点发送GET请求, 校验数据一致性
```
cd kvstore/test
bash run_ms_sync_test.sh
```

#### 5.多命令拼接/超长指令测试

##### 多命令拼接-测试方案
测试程序主线程批量编码100条SET命令, 拼入缓冲区一次性send;单独子线程持续循环接收校验所有RESP应答, 统计QPS(从发送第一批SET命令到收到最后一批RESP应答)
复用校验程序, 仅用来校验数据一致性, 暂时维持逐条而不是批量发送GET命令
```
cd kvstore/test
bash run_batch_100k_test.sh
```
##### 多命令拼接-测试结果
|SKIPTABLE|QPS|
|---|---|
|single|16946|
|batch|57208|

##### 超长指令-测试方案
构造`*1\r\n$N\r\n[N个X]\r\n`超大单元素 RESP 请求; 按 4096 字节分片发送, 每片休眠 1ms; 最后一片和 EXIST 命令合并一次 send 制造粘包; 阻塞依次读取两行 RESP 响应, 校验两条返回是否匹配预期
```
cd kvstore/test
bash run_long_command_test.sh
```

## 核心功能实现

<details>
<summary>存储引擎的实现</summary>

功能解决的问题|功能入口|输入数据|核心步骤|关键代码
|---|---|---|---|---|
|解决最基础的kv数据操作问题|由kvs_filter_protocol()调用的kvs_engine_opnd()|用户输入数据, 全局引擎数据结构|见各引擎实现|kvsotre/src/kvengine|
</details>


<details>
<summary>全量持久化的实现</summary>

功能解决的问题|功能入口|输入数据|核心步骤|关键代码|
|---|---|---|---|---|
|解决内存数据掉电丢失、日志重放慢的问题|配置文件选择RDB持久化模式时, 1.数据保存: 由SAVE操作触发kvs_engine_save(), 2.数据加载: init_kvengine()初始化引擎时完成数据加载, kvs_hash_create|持久化文件路径, 全局引擎数据结构|数据保存: 打开持久化文件->创建io_uring的SQ和CQ->遍历内存引擎数据结构逐条处理数据->对每条KV数据构造rdb_write_req_t结构体(二进制数据部分=keylen,key,vallen,value,crc32,逗号为结构性说明,实际不保存)->将结构体中的二进制数据提交到写请求->累计多个写请求提交一次\r\n数据加载: 打开持久化文件并记录文件长度->使用mmap(只读)映射磁盘文件内容到进程虚拟内存, 并返回指针->循环读二进制文件,加载kv数据并校验crc32|kvsotre/src/kvs_engine_rdb_save,kvsotre/src/kvs_engine_rdb_load|
</details>

<details>
<summary>增量持久化的实现</summary>

功能解决的问题|功能入口|输入数据|核心步骤|关键代码|
|---|---|---|---|---|
|解决内存数据掉电丢失、日志重放慢的问题|配置文件选择RDB持久化模式时, 1.数据保存: 由SAVE操作触发kvs_engine_save(), 2.数据加载: init_kvengine()初始化引擎时完成数据加载, kvs_hash_create|持久化文件路径, 全局引擎数据结构|数据保存: 打开持久化文件->创建io_uring的SQ和CQ->遍历内存引擎数据结构逐条处理数据->对每条KV数据构造rdb_write_req_t结构体(二进制数据部分=keylen,key,vallen,value,crc32,逗号为结构性说明,实际不保存)->将结构体中的二进制数据提交到写请求->累计多个写请求提交一次\r\n数据加载: 打开持久化文件并记录文件长度->使用mmap(只读)映射磁盘文件内容到进程虚拟内存, 并返回指针->循环读二进制文件,加载kv数据并校验crc32|kvsotre/src/kvs_engine_rdb_save,kvsotre/src/kvs_engine_rdb_load|
</details>

<details>
<summary>RESP2协议的实现</summary>
功能解决的问题: 解决了TCP字节流没有消息边界的问题, 二进制安全(按给定长度读取原始字节，只对 " 转义), 支持批量指令
核心逻辑: 
|请求|请求结构体|实际内容|请求字节流|
|---|---|---|---|
|SET "key" "value"|typedef struct resp_request {<br> int argc;                 //参数量<br> char **argv;              //指向参数内容的指针<br>size_t *argv_len;         //参数长度<br> } resp_request_t;           //请求结构体<br> |argc = 3 <br>argv = ["SET", "key", "value"] <br>argv_len = {3, 3, 5}<br>|*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n
   
    resp_encode_request(): 将用户输入命令解析为请求结构体
    resp_encode_stream(): 将请求结构体编码为RESP2协议格式的字节流
    resp_decode_request(): 将字节流解析为请求结构体
关键代码: kvsotre/src/RESP2/kvs_resp.c
</details>
