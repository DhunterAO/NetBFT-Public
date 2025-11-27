# NetBFT-Public (Kauri) 项目逻辑概览

## 一、项目概述

**Kauri** 是一个基于 HotStuff 共识算法的 BFT（Byzantine Fault Tolerance）通信抽象层，主要特点：

1. **树形传播和聚合**：使用树结构平衡消息传播和处理负载
2. **BLS 签名**：通过签名聚合减少带宽消耗
3. **流水线化**：通过额外的流水线抵消树结构的延迟成本

## 二、系统架构

### 2.1 核心组件层次

```
应用层 (examples/hotstuff_app.cpp)
    ↓
共识层 (include/hotstuff/hotstuff.h, consensus.h)
    ↓
实体层 (include/hotstuff/entity.h) - Block, Command, ReplicaConfig
    ↓
网络层 (salticidae) - 消息传递、事件循环
    ↓
加密层 (bls/, secp256k1/) - 签名和验证
```

### 2.2 节点类型

- **内部节点 (Internal Nodes)**：树结构中的中间节点，负责消息聚合和转发
- **叶子节点 (Leaf Nodes)**：树的叶子，只接收和发送消息，不进行聚合

### 2.3 树结构构建

在 `src/hotstuff.cpp` 的 `start()` 方法中构建树：

1. **计算层级**：根据 fanout（扇出）和总节点数计算树的层级
2. **分配父子关系**：
   - 第 0 层：根节点（leader）
   - 第 1 层：fanout 个内部节点
   - 后续层：每个内部节点有最多 fanout 个子节点
3. **建立连接**：每个节点知道自己的 parent 和 children

## 三、共识流程

### 3.1 HotStuff 协议基础

HotStuff 是一个三阶段提交的 BFT 共识协议：

1. **Propose**：Leader 提出新区块
2. **Vote**：Replicas 投票
3. **Commit**：收集足够票数后提交

### 3.2 Kauri 的改进

#### 3.2.1 树形传播 (Tree-Based Dissemination)

**向下传播（Proposal）**：
```
Leader → 内部节点 → 叶子节点
```

**向上聚合（Votes）**：
```
叶子节点 → 内部节点（聚合签名）→ Leader
```

#### 3.2.2 BLS 签名聚合

- 每个节点投票时使用 BLS 签名
- 内部节点可以聚合子节点的签名
- 最终 Leader 收到的是聚合后的签名，减少带宽

#### 3.2.3 流水线化 (Pipelining)

- `piped_latency`：流水线块之间的延迟
- `async_blocks`：允许同时处理的异步块数量
- 允许在等待前一个块确认时就开始处理下一个块

## 四、关键参数

### 4.1 实验参数（experiments 文件）

格式：`['crypto','fanout','pipedepth','pipelatency','latency','bandwidth','blocksize']:internals:total:machines`

- **crypto**: 加密算法（'bls' 或 'secp256k1'）
- **fanout**: 树的分支因子（每个内部节点的子节点数）
- **pipedepth**: 流水线深度（async_blocks）
- **pipelatency**: 流水线延迟（毫秒）
- **latency**: 网络延迟（毫秒）
- **bandwidth**: 网络带宽（Mbps）
- **blocksize**: 每个块包含的交易数

### 4.2 节点配置

- **内部节点数 (internals)**: 树结构中的内部节点数量
- **总节点数 (total)**: 所有节点数量
- **建议物理机数 (machines)**: 建议的物理机器数量（每20个节点1台机器）

## 五、执行流程

### 5.1 实验启动流程

```
runexperiment.sh
    ↓
读取 experiments 文件
    ↓
生成 net-temp.yaml (替换占位符)
    ↓
docker stack deploy
    ↓
容器启动 → server.sh 执行
    ↓
server.sh 流程：
    1. 获取参数（crypto, fanout, pipedepth 等）
    2. 通过 DNS 发现其他节点
    3. 确定自己的 ID
    4. 生成配置文件（gen_conf.py）
    5. 编译代码
    6. 启动 hotstuff-app
    7. 配置网络限制（tc qdisc）
    8. 启动客户端（仅 ID=0 的节点）
```

### 5.2 节点发现机制

在 `server.sh` 中：

1. 使用 DNS 查询服务名：
   - `server1-$KAURI_UUID`：内部节点服务
   - `server-$KAURI_UUID`：叶子节点服务

2. 通过 `dig` 命令获取所有节点 IP

3. 通过 `ifconfig` 获取本机 IP，匹配确定自己的 ID

4. 生成 `ips` 文件，包含所有节点信息

### 5.3 配置生成

`scripts/gen_conf.py` 生成配置文件：

1. **主配置文件** (`hotstuff.gen.conf`)：
   - 包含所有 replica 信息
   - 设置 fanout、pipelatency、async_blocks 等

2. **每个节点的配置文件** (`hotstuff.gen-sec{id}.conf`)：
   - 私钥
   - TLS 证书
   - 节点索引

## 六、消息类型

在 `include/hotstuff/hotstuff.h` 中定义：

1. **MsgPropose** (0x0)：提案消息
2. **MsgVote** (0x1)：投票消息
3. **MsgReqBlock** (0x2)：请求区块
4. **MsgRespBlock** (0x3)：响应区块
5. **MsgRelay** (0x4)：中继消息（用于树形传播）

## 七、关键数据结构

### 7.1 Block（区块）

```cpp
class Block {
    std::vector<uint256_t> parent_hashes;  // 父区块哈希
    std::vector<uint256_t> cmds;            // 命令（交易）
    quorum_cert_bt qc;                      // 法定人数证书
    uint32_t height;                        // 高度
    // ...
}
```

### 7.2 ReplicaConfig（副本配置）

```cpp
class ReplicaConfig {
    size_t nreplicas;      // 副本总数
    size_t nmajority;      // 多数派数量
    int32_t fanout;        // 扇出
    int32_t piped_latency; // 流水线延迟
    int32_t async_blocks;  // 异步块数
}
```

## 八、日志系统

### 8.1 日志级别

- **ERROR**: 始终启用
- **INFO/WARN**: 需要 `HOTSTUFF_NORMAL_LOG`
- **DEBUG**: 需要 `HOTSTUFF_DEBUG_LOG`
- **PROTO**: 需要 `HOTSTUFF_PROTO_LOG`

### 8.2 日志输出

- 默认输出到 stderr
- 运行时通过 `> log${id} 2>&1` 重定向到文件
- 日志文件：`log0`, `log1`, `log2` 等

### 8.3 关键日志信息

- `x now state`: 当前状态（包含 hqc.height，即已确认的区块高度）
- `Average`: 平均延迟
- `commit <block>`: 提交的区块

## 九、性能指标

### 9.1 吞吐量计算

从日志中提取 `hqc.height`，例如：
```
hqc.height=2700  (5分钟 = 300秒)
吞吐量 = 2700/300 = 9 blocks/秒
如果 blocksize=1000，则 = 9000 ops/秒
```

### 9.2 延迟

日志中的 `Average` 值表示平均区块延迟（毫秒）

## 十、网络配置

### 10.1 Docker Swarm 设置

- 使用 overlay 网络 (`kauri_network`)
- DNS 服务发现（dnsrr 模式）
- 每个服务有多个副本（replicas）

### 10.2 网络限制

在 `server.sh` 中使用 `tc qdisc` 模拟网络条件：
```bash
tc qdisc add dev eth0 root netem delay ${latency}ms rate ${bandwidth}mbit
```

## 十一、故障处理

### 11.1 Pacemaker（节奏器）

- **Dummy**: 固定 proposer
- **RR (Round-Robin)**: 轮询 proposer

### 11.2 弹劾机制

如果 leader 超时，其他节点可以弹劾（impeach）leader

## 十二、代码关键路径

### 12.1 启动路径

```
main() [hotstuff_app.cpp]
  ↓
HotStuffApp::start()
  ↓
HotStuff::start() [hotstuff.cpp]
  ↓
构建树结构（建立 parent/children 关系）
  ↓
初始化网络连接
  ↓
启动事件循环 ec.dispatch()
```

### 12.2 提案路径

```
客户端请求 → HotStuffApp::client_request_cmd_handler()
  ↓
exec_command() → 添加到 cmd_pending_buffer
  ↓
beat() → 检查是否为 proposer
  ↓
on_propose() → 创建 Block
  ↓
do_broadcast_proposal() → 通过树结构传播
```

### 12.3 投票路径

```
收到 Proposal → on_receive_proposal()
  ↓
验证并投票 → on_receive_vote()
  ↓
通过树结构向上聚合签名
  ↓
收集足够签名 → on_qc_finish()
  ↓
更新 hqc → commit
```

## 十三、实验运行

### 13.1 准备步骤

1. 在所有机器上构建 Docker 镜像
2. 设置 Docker Swarm
3. 创建 overlay 网络
4. 配置 experiments 文件

### 13.2 运行实验

```bash
cd runkauri
./runexperiment.sh
```

### 13.3 结果收集

脚本会自动：
1. 部署实验
2. 等待运行（默认 150 秒）
3. 收集日志
4. 提取关键指标（commit, state, Average）
5. 清理服务

## 十四、关键文件说明

| 文件 | 作用 |
|------|------|
| `runexperiment.sh` | 实验主脚本 |
| `server.sh` | 容器启动脚本 |
| `gen_conf.py` | 生成配置文件 |
| `hotstuff_app.cpp` | 应用主程序 |
| `hotstuff.cpp` | 共识核心实现 |
| `consensus.cpp` | 共识状态机 |
| `entity.h` | 数据实体定义 |
| `experiments` | 实验配置 |

## 十五、扩展阅读

- HotStuff 论文：了解基础共识算法
- Kauri 论文（SOSP 2021）：了解树形传播和聚合机制
- BLS 签名：了解签名聚合原理

