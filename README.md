# Kvstore

## 简介
以跳表为存储引擎, 以基于协程的网络服务基础设计的KV存储项目, 支持基于字符串的KV数据库的创建和编辑.

## 功能特性
- 存储引擎: 支持数组, 红黑树, 哈希表, 跳表
- 网络服务: 基于协程(Ntyco)开发的网络服务, 实现异步网络io.
- 全量持久化: 通过SAVE操作触发, 并将内存中数据全量落盘到二进制快照文件中, 加载数据时解析二进制文件.
- 增量持久化: 向日志文件中追加用户对数据库的修改操作, 加载数据时解析并回放日志.
- 分级页式空闲链表内存池: 对 8 B 至 4KB 的对象采用分级(9级)页内分配, 对更大的对象采用页对齐的连续空间分配, 并通过空闲链表实现回收与相邻块合并.
- RESP2: 支持Redis序列化协议
- 支持多指令/不限消息长度: 客户端发送RESP2编码后拼接的多命令/超长字节流, 服务端通过双缓冲区机制, 持续将recv到的临时缓冲区的内容搬运到消息缓冲区并动态扩容, 同时解析并消费消息缓冲区中的完整消息.
- 主从同步: 采用「首次同步」+「增量同步」机制, 

## 快速开始

### 加载子模块
```
git submodule update --init
```
遇到网络问题可配置代理:
```
git config --global http.proxy http://x.x.x.1:<port>
git config --global https.proxy http://x.x.x.1:<port>
```
### 创建并把软件RDMA实例(rxe0)绑定到底层物理网卡
```
rdma link add rxe0 type rxe netdev <device>
```
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
当 `REPLICATION_MODE = EBPF` 时，在主节点启动后、从节点启动前运行独立转发转发进程：
```
sudo ./repl_forwarder $(pgrep -n -x kvstore)
```
转发进程将 uprobe 挂载到主进程的 `kvs_repl_commit_raw()` 通知点。写命令成功后，eBPF 将原始 RESP 写入 ring buffer；普通命令按实际长度选择 256 B、1 KiB 或 4 KiB 紧凑事件，大命令按 128 KiB 分块。转发进程重组完整命令、缓存首次同步期间的增量，并在首次同步完成后通过 TCP 按 FIFO 顺序转发。`REPLICATION_MODE = NETWORK` 时不需要启动该转发进程。

### 启动从节点
修改 /kvsotre/config/config.ini 中的**ROLE**的值为**SLAVE**
```
sudo ./kvstore
```

首次同步方式由 `FULL_SYNC_MODE` 控制：

```ini
FULL_SYNC_MODE = RDMA       ; 默认, RDMA READ 全量同步
FULL_SYNC_MODE = SENDFILE   ; TCP sendfile 传输快照并加载到从节点
```

`SENDFILE` 使用 `full_sync_port`，默认 `2004`；增量复制仍使用 `m2s_port`。网络基准测试请独立运行系统 `iperf3`，不要修改 `FULL_SYNC_MODE`。

```bash
sudo apt install iperf3
iperf3 -s -p 2004                 # 从节点
iperf3 -c <从节点IP> -p 2004 -t 10 # 主节点
```
### 节点属性配置
通过修改 /kvsotre/config/config.ini 修改节点属性, 包括**主节点ip, 开放端口, 持久化文件路径, 节点角色, 持久化方式, 存储引擎**。`FULL_SYNC_MODE` 控制首次同步方式：`RDMA` 使用 RDMA READ，`SENDFILE` 使用 TCP sendfile。`REPLICATION_MODE` 控制增量捕获方式：`EBPF` 使用 uprobe 捕获，`NETWORK` 在应用层直接将写命令加入 TCP 转发队列。

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
所有性能测试均在以下配置完成: 
CPU: 4核心4线程(3.5 GHz)
MEM: 3200 MT/s
SSD: KIOXIA-EXCERIA G2 1TB

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
|method|echo|AOF关闭|AOF开启|
|---|---|---|---|
|kvstore|40436|25310|19245|
|redis|45306|39277|38284|

当接收缓冲区大小设置为1024时,kvstore

#### 4.不同内存方案对比

##### 测试方案
保持其他参数不变, 对比自定义内存池/libc内存方案/jemalloc在连续插入删除10w条连续数据时, 节点的虚拟内存/物理内存的起始值/峰值/终止值和
```
cd kvstore/test
bash run_mem_test.sh
```
##### 测试结果
|custome|Start VmRSS|Peak VmRSS|End VmRSS|Start VmSize|Peak VmSize|End VmSize|
|---|---|---|---|---|---|---|
|首次插入10w条|2356|19548|19548|132660|132844|132792|
|删除后重新插入10w条,第一轮|19548|19548|19548|132792|132844|132792|
|删除后重新插入10w条,第n轮|19548|19548|19548|132792|132844|132792|
|删除后重新插入10w条,第10轮|19548|19548|19548|132792|132844|132792|

|malloc|Start VmRSS|Peak VmRSS|End VmRSS|Start VmSize|Peak VmSize|End VmSize|
|---|---|---|---|---|---|---|
|首次插入10w条|2264|19428|19428|91692|107320|107268|
|删除后重新插入10w条,第一轮|19428|19428|19428|107268|107320|107268|
|删除后重新插入10w条,第n轮|19428|19428|19428|107268|107320|107268|
|删除后重新插入10w条,第10轮|19428|19428|19428|107268|107320|107268|

|jemalloc|Start VmRSS|Peak VmRSS|End VmRSS|Start VmSize|Peak VmSize|End VmSize|
|---|---|---|---|---|---|---|
|首次插入10w条|5728|16360|16360|8992|50532|59800|59748|
|删除后重新插入10w条,第一轮|8992|16348|16348|8148|59748|59800|59748|
|删除后重新插入10w条,第n轮|-|16348|16348|-|59748|59800|59748|
|删除后重新插入10w条,第10轮|8200|16348|16348|9544|59748|59800|59748|

内存池结束状态物理内存没降: jemalloc会主动把空闲物理内存归还给操作系统, 而自定义内存池没有调用madvise主动归还空闲内存, 被分配的内存仍然属于进程地址空间, 留在进程内复用, 因此RSS不会降低
开始状态custom比malloc多: 内存池初始化时, 空闲链表等元数据被分配了额外的物理内存.
峰值状态custom比malloc多: 发现一处内存泄漏问题, 现已修复.

内存稳定性: 首次插入后, VmRSS 稳定, 后续删除、重新插入操作未导致 RSS 继续增长
虚拟内存: 首次插入后, VmSize 稳定, 后续删除、重新插入操作未导致 VmSize 继续增长
内存泄漏情况: 在本次测试的9次删除后重新插入中, 未观察到持续性内存增长, 但仅凭这些指标不能完全排除内存泄漏


#### 5.主从同步测试

##### 测试方案
分两次插入共计10w条连续数据, 首先启动主节点并插入前5w条数据, 从节点连接主节点进行首次同步, 随后向主节点插入后5w条数据(增量同步), 随后向从节点发送GET请求, 校验数据一致性
```
cd kvstore/test
bash run_ms_sync_test.sh
```

#### 6.批量指令/超长指令测试

##### 批量指令-测试方案
测试程序主线程每批次编码1/10/20/40/80/160条SET命令, 拼入缓冲区一次性send;随后接收并校验本批全部RESP应答, 再发送下一批, 统计QPS(从处理第一批SET命令到收到最后一批RESP应答)
复用校验程序, 仅用来校验数据一致性, 暂时维持逐条而不是批量发送GET命令
```
./test/insert_100k_batch <master-ip> <master-port> <start> <count> <batch-size>
```
##### 多命令拼接-测试结果
|method|1|10|20|40|80|160|
|---|---|---|---|---|---|---|
|kvstore|19131|90009|89525|87032|94876|103626|
|redis|44306|236189|238153|253164|270379|293255|

当接收缓冲区大小设置为1024时,kvstore的qps维持在3.3w左右, 增大接收缓冲区至8192能够解决这一问题

##### 超长指令-测试方案
构造`*1\r\n$N\r\n[N个X]\r\n`超大单元素 RESP 请求; 按 4096 字节分片发送, 每片休眠 1ms; 最后一片和 EXIST 命令合并一次 send 制造粘包; 阻塞依次读取两行 RESP 响应, 校验两条返回是否匹配预期
```
cd kvstore/test
bash run_long_command_test.sh
```

#### 7.SAVE指令对性能的影响

##### 测试方案
对比不同数据量下save指令耗时
##### 测试结果
|Data|time|qps|
|---|---|---|
|100w(内存池)|26.4|37788|
|100w(malloc)|14.6|68012|
|10w|1.24s|80364|
|1w|0.16|61185|
|1k|0.03|31612|

##### 结果分析
100w条记录测试性能发生退化, 主要受内存池free策略影响. 当前free策略会从第一个尺寸档开始遍历所有页面寻找指针归属, 因此每次释放都会扫描大量页面(时间复杂度O(N²)), 使用glibc的内存方案时则没有大幅退化, 进一步验证了这一分析.
1k条记录测试性能低, 推测主要原因是save操作固定开销(文件的打开与关闭, ring初始化等)占比较大无法被充分分摊.

#### 8.主从同步性能的影响

##### 8.1增量数据同步性能测试方案
对比开启ebpf转发和网络转发情况下插入10w条数据的qps

##### 测试结果
|Method|qps|
|---|---|
|ebpf|14283|
|network|12318|

##### 结果分析
100w条记录测试性能发生退化, 主要受内存池free策略影响. 当前free策略会从第一个尺寸档开始遍历所有页面寻找指针归属, 因此每次释放都会扫描大量页面(时间复杂度O(N²)), 使用glibc的内存方案时则没有大幅退化, 进一步验证了这一分析.
1k条记录测试性能低, 推测主要原因是save操作固定开销(文件的打开与关闭, ring初始化等)占比较大无法被充分分摊.

##### 8.2已有数据(首次)同步性能测试方案
RDMA、sendfile 的同步速率可通过程序输出的 `FULL_SYNC` 日志对比；网络基准吞吐请单独运行 `iperf3`。

##### 测试结果
|Method|rate(2.8MB)|rate(14MB)|
|---|---|---|
|iperf3|1.12 MiB/s|1.12 MiB/s|
|sendfile|1.02 MiB/s|1.03 MiB/s|
|rdma|36 MiB/s|2.53 MiB/s|



##### 结果分析



## 核心功能实现

<details>
<summary>存储引擎的实现</summary>

**功能解决的问题:** 解决最基础的kv数据操作问题

**功能入口:** 由kvs_filter_protocol()调用的kvs_engine_opnd()

**输入数据:** 用户输入数据, 全局引擎数据结构

**核心步骤:** 见各引擎实现

**关键代码:** `kvsotre/src/kvengine`

</details>

<details>
<summary>全量持久化的实现</summary>

**功能解决的问题:** 解决内存数据掉电丢失、日志重放慢的问题

**功能入口:** 配置文件选择RDB持久化模式时:

1. 数据保存:由SAVE操作触发kvs_engine_save()
2. 数据加载:调用链init_kvengine()->kvs_engine_create()->kvs_engine_load()

**输入数据:** 持久化文件路径, 全局引擎数据结构

**核心步骤:**

1. **数据保存:** 打开持久化文件->创建io_uring的SQ和CQ->遍历内存引擎数据结构逐条处理数据->对每条KV数据构造rdb_write_req_t结构体(二进制数据部分=keylen,key,vallen,value,crc32,逗号为结构性说明,实际不保存)->将结构体中的二进制数据提交到写请求->累计多个写请求提交一次

2. **数据加载:** 打开持久化文件并记录文件长度->使用mmap(只读)映射磁盘文件内容到进程虚拟内存, 并返回指针->循环读二进制文件,加载kv数据并校验crc32

**关键代码:** `kvsotre/src/kvs_engine_rdb_save`, `kvsotre/src/kvs_engine_rdb_load`

</details>

<details>
<summary>增量持久化的实现</summary>

**功能解决的问题:** 不用每次全量备份, 弥补全量快照两次快照间丢数据的缺陷

**功能入口:** 配置文件选择AOF持久化模式时:

1. 操作保存:kvstore.c: persist_request()
2. 数据加载:调用链init_kvengine()->kvs_engine_create()->kvs_engine_load()

**输入数据:** 持久化文件路径, 全局引擎数据结构

**核心步骤:**

1. **数据保存:** kvs_filter_protocol()过滤到写操作时, 如果操作成功执行->persist_request()将RESP编码后的命令写入缓冲区->wal_write()读取,并落盘命令

2. **数据加载:** kvs_engine_load()->engine_replay_log()->mmap()将日志文件映射到进程虚拟内存->循环读取文件.解码并重放日志命令

**关键代码:** `kvsotre/src/kvs_engine_aof_save`, `kvsotre/src/kvs_engine_aof_load/`

</details>

<details>
<summary>内存池的实现</summary>

**功能解决的问题:** 频繁 malloc/free 的系统调用开销; 避免内存泄漏风险

**功能入口:** mem_init()初始化内存池 mem_init()销毁内存池 mem_alloc()分配内存 mem_free()释放内存

**核心结构:**

**多级分配:**

- **小对象(<=4096B):** 首次分配时, 根据申请大小归入 16B～4096B 共 9 个规格之一, 并从底层内存池申请一个 4096B Page；Page 按对应规格划分为若干等大小 block, 页内通过空闲链表管理 block. 相同规格的小对象优先从该规格已有且存在空闲 block 的 Page 中分配; 当已有 Page 均满时创建新 Page; 当某 Page 中所有 block 均被释放时.将整个 Page 回收到底层空闲链表。

- **大对象(>4096B):** 将申请大小向上对齐到 4096B 的整数倍, 直接从底层内存池分配连续空间, 并通过大块链表 large_list 记录该大对象的地址和实际分配大小; 释放时根据该记录将整块空间归还到底层空闲链表.

**关键代码:** `kvsotre/src/kvs_mempool.c`

</details>

<details>
<summary>RESP2协议的实现</summary>

**功能解决的问题:** 解决了TCP字节流没有消息边界的问题, 二进制安全(按给定长度读取原始字节,只对 " 转义), 支持批量指令

**核心逻辑:**

|请求|请求结构体|实际内容|请求字节流|
|---|---|---|---|
|SET "key" "value"|typedef struct resp_request {<br> int argc;                 //参数量<br> char **argv;              //指向参数内容的指针<br>size_t *argv_len;         //参数长度<br> } resp_request_t;           //请求结构体<br> |argc = 3 <br>argv = ["SET", "key", "value"] <br>argv_len = {3, 3, 5}<br>|*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n

- `resp_encode_request()`:将用户输入命令解析为请求结构体
- `resp_encode_stream()`:将请求结构体编码为RESP2协议格式的字节流
- `resp_decode_request()`:将字节流解析为请求结构体

**关键代码:** `kvsotre/src/RESP2/kvs_resp.c`

</details>

<details>
<summary>批量指令多指令识别的实现</summary>

**功能解决的问题:** 单指令并发时, 网络层多次执行send()触发系统调用, cpu频繁切换开销大, 因而实现支持多指令拼接一次性发送和接收

**核心步骤:**

1. **客户端实现多指令拼接:** 为了让输入更加直观, 客户端要求逐行输入命令, 输入空行+回车时, 逐行编码用户命令为resp2格式命令并拼接至发送缓冲区, 随后执行一次send.

2. **服务端实现多指令帧分隔:** 服务端server_reader()协程循环将对应fd的数据recv到buf1, 随后执行recv_buf_append()扩容函数动态扩容buf2, 并将buf1复制到buf2尾部, 当buf2长度不为0时(存在命令), 循环执行resp_decode_request()解码buf中的命令并返回命令完整度, 解码到完整命令则消费该命令, 消费缓冲区并执行下一轮解码, 解码到不完整命令则退出到上层循环继续接收命令.(命令执行速度瓶颈在resp_decode_request()解码命令)

**关键代码:** `kvsotre/src/network/kvs_ntyco.c`, `kvsotre/src/RESP2/kvs_resp.c`

</details>

<details>
<summary>主从同步->首次同步的实现</summary>

**功能解决的问题:** 解决从节点首次连接主节点时的数据同步问题, 从节点通过CM与主节点连接,并用 RDMA Read 拉取主库内存里的全量数据,CPU 不参与数据拷贝

**功能入口:** m_sync_server(), s_sync_server()

**核心步骤:**

**首次同步逻辑:** 主节点启动客户端服务协程后, rdma线程在同步端口监听, rdma线程会开启主节点的同步服务m_sync_server()并等待从节点链接, 从节点启动 s_sync_server()服务与主节点链接, 主节点生成二进制快照并注册内存区域, 从节点rdma read完成同步.

**m_sync_server()的rdma逻辑:**

1. **等待连接事件:** 通过rdma_create_event_channel()建立连接管理事件通道->rdma_create_id()绑定监听fd->rdma_bind_addr()绑定地址->rdma_listen()启动监听->rdma_get_cm_event()阻塞等待来自从节点的连接事件->

2. **事件到达:** rdma_setup_qp()创建PD保护域,创建完成队列CQ,绑定队列对(SQ,RQ)

3. **生成快照:** kvs_rdma_rdb_create()生成kv数据快照并使用ibv_reg_mr()注册内存区域, 填充快照信息(remote_addr/rkey/data_len), 提前提交接收从节点对快照的ACK的请求

4. **建立连接:** rdma_accept()接收来自从节点的connect, 发送应答报文顺带返回快照元数据

5. **等待同步成功:** rdma_wait_cq()

**s_sync_server()的rdma逻辑:**

1. **主动连接:** rdma_create_event_channel()->rdma_create_id()->rdma_setup_qp()->rdma_connect()

2. **获取快照:** 客户端收到后主节点的应答报文后, 解析其捎带的param, 获取remote_addr/rkey/data_len, 包装读请求后, ibv_post_send()发送请求,rdma_wait_cq()等待读请求完成

3. **解析快照:** kvs_rdma_rdb_load()

**关键代码:** `kvsotre/src/network/kvs_ntyco.c`, `kvsotre/src/kvs_sync.c`

</details>

<details>
<summary>主从同步->增量同步的实现</summary>

**功能解决的问题:** eBPF 用来捕获主库新增写操作,数据库源代码改动小, 低延迟拿到增量数据

**功能入口:** m_sync_server(), replication_server()

**核心步骤:**

**整体逻辑:** 独立 `repl_forwarder` 转发进程加载 eBPF 并挂载主进程的成功写通知点。首次同步开始后捕获原始 RESP，RDMA 快照完成前缓存，完成后通过 TCP 转发给从节点的 `replication_server()` 增量服务。

**主节点的ebpf逻辑:**

1. **成功后通知:** 网络层保留当前命令的原始 RESP；`SET/MOD/DEL` 执行并持久化成功、返回 `+OK` 后调用 `kvs_repl_commit_raw(resp, len)`。从节点回放连接不会触发通知，避免复制循环。

2. **eBPF 分块捕获:** uprobe 挂载到 `kvs_repl_commit_raw()`，普通命令按长度使用 256 B、1 KiB 或 4 KiB 事件，较大的原始 RESP 按 128 KiB 分块，最大支持 4 MiB。每块携带命令序号、总长度和偏移；ring buffer 满或读取失败时转发进程停止当前复制会话，避免静默发送残缺命令。大块路径使用最多 32 次 tail call，每次程序只处理一块，避免 Linux 5.15 verifier 展开循环状态。

3. **转发进程重组转发:** `repl_forwarder` 校验序号和连续偏移后重组完整 RESP。收到主进程的 `TARGET` 控制消息才开启捕获；收到 RDMA ACK 对应的 `LIVE` 消息后先清空同步期间 FIFO，再实时发送后续命令。主进程不加载 libbpf、不查询 BPF map，也不为 EBPF 模式重新编码或逐条发送 IPC。

**关键代码:** `kvsotre/src/network/kvs_ntyco.c`, `kvsotre/src/kvs_sync.c`

</details>
