# Midnight 上的多方私密状态与合约

## TL;DR

Midnight 官方的双人示例是个不错的起点，但如果你做三处设计调整，这套核心模式就能扩展到 *N* 个参与者：

1. **把固定的每方字段替换成用稳定参与方标识符作为 key 的 `Map`。**  
   在 Compact 里，这意味着把 commitment 和每个参与方的元数据存进 ledger `Map`，而不是把 `aliceCommitment` 和 `bobCommitment` 这种字段硬编码进合约。

2. **把合约发现与加入视为 client 侧职责。**  
   多个用户应该都能通过 Midnight 示例里提到的 generated driver 或 SDK helper 绑定到同一个已部署合约实例，包括这个 bounty prompt 里点名的 `findDeployedContract`。具体 API 形态取决于 SDK 版本，所以要固定 package 版本，并对照当前文档核实。

3. **显式按并发更新来设计。**  
   Midnight 合约运行在基于 proof 的模型上。如果两个用户基于同一份旧状态构建 proof，其中一个最终会变成 stale。  
   因此，合约应该维护一个公开的 version/nonce，并要求每个会修改状态的 action 都证明自己是基于当前 version 构建的。

这篇教程会围绕一个**私密 multi-sig treasury**示例展开。这个 treasury 的 threshold 和成员数量是公开的，但每个成员的 approval 状态会用以参与方标识符为 key 的 commitment 来表示。文章重点讲的是你可以安全地从 Midnight 公共文档和 Compact 语言参考中推导出来的模式；凡是 client API 里和版本强相关的部分，我都会明确标注为假设，并回链到官方资料。

## Context

Midnight 的执行模型并不是“把 TypeScript 直接跑在链上”。Compact 合约会编译成 proving circuits 和一个 JavaScript/TypeScript driver；用户在本地执行 circuit、生成 proof，再把这些 proof 提交到链上。本教程最相关的三个 primitive，正是 Compact language reference 里定义的这几个：

- 用于公开链上状态的 **Ledger declarations**
- 用于 entry point 和状态迁移的 **Circuits**
- 由 DApp 或 wallet 提供、作为私密链下输入的 **Witnesses**

语法和语义的权威来源请看 Midnight 文档里的 Compact language reference：[Compact language reference](https://docs.midnight.network/develop/reference/compact/lang-ref)。如果你想先从更高层开始，可以先看入门指南：[Midnight getting started](https://docs.midnight.network/getting-started)。

Midnight 文档里的双人示例通常长这样：

- party A 有一个 commitment
- party B 有一个 commitment
- 当这两个 commitment 满足某种关系时，合约发生状态迁移

这种模式很适合做教程，因为容易理解。但如果你想支持下面这些能力，它就**不**够扩展了：

- 任意大小的 group
- 部署后继续加入新参与者
- 不同用户反复执行 action
- 对并发更新有韧性

只要参与者达到三人或更多，硬编码字段就不是正确抽象了。你不再是处理 “Alice 和 Bob”，而是在处理“由 `partyId` 标识的 participant”。你不再是给每个 actor 一个 commitment slot，而是需要一个集合。`Map<K, V>` 这个 ledger 类型就是专门为这类索引状态准备的；在 Compact standard library 示例、reference primer 和官方文档里都能看到它。

本教程会把 public ledger 压到最小：

- `threshold`：需要多少个 approval
- `memberCount`：当前有多少成员加入
- `stateVersion`：单调递增的版本号，用来检测 stale proof
- 多个用 `partyId` 作为 key 的 `Map`，存放 commitment 和每个参与方的元数据

这个划分很关键。在 Midnight 里，隐私通常来自“证明关于私密数据的陈述”，而不是假装链上什么都不存。你的工作是决定：哪些数据必须公开，方便协作；哪些值应该保持私密，只通过 commitment 表达。

## Extending two-party patterns to N parties

从双人模式走到 N 人模式，概念上其实很简单：

- 用**索引参与者**替代**命名角色**
- 用**maps**替代**单个 slot**
- 用 **join / update / approve flow** 替代**单步协作**
- 用**带 version 检查的状态迁移**替代“last writer wins”思路

在一个双人合约里，你可能会看到这种非正式逻辑描述：

- Alice 发布一个 commitment
- Bob 发布一个 commitment
- 两者都到位后，才能进入下一次状态迁移

到了 N 方，同样的逻辑会变成：

- 任意 party 都可以通过在自己的 `partyId` 下注册 commitment 来加入
- 任意 party 都可以更新自己的 commitment
- 当 map 中有足够多满足策略的有效 participant state 时，状态迁移就被启用

这里最关键的设计问题是：**`partyId` 到底是什么？**

对于教程仓库来说，建议用一个稳定、确定性的标识符，并尽量缩小信任面。一个很好的默认值是固定宽度字节串，比如 `Bytes<32>`，因为它既能表示 hash、编码后的公钥，也能表示应用自定义标识符，而且不要求合约理解更高层的格式。

于是，合约大致可以长这样：

```compact
import CompactStandardLibrary;

export sealed ledger threshold: Uint<32>;
export ledger memberCount: Uint<32>;
export ledger stateVersion: Uint<32>;

export sealed ledger joined: Map<Bytes<32>, Boolean>;
export sealed ledger commitments: Map<Bytes<32>, Bytes<32>>;
export ledger approvalNonce: Map<Bytes<32>, Uint<32>>;
```

上面这段声明里的每个元素都能在 Compact reference 里找到依据：

- `ledger` 和 `sealed ledger` declaration 是标准的 Compact 语法
- `Uint<32>`、`Boolean`、`Bytes<32>` 都是文档里写明的 primitive type
- `Map<K, V>` 是文档化的 standard-library container type

这种模型能带来的好处：

- **可变成员数量**：可以表示任意数量的参与者
- **统一逻辑**：同一个 circuit 可以作用于任意 `partyId`
- **私密本地状态**：参与者可以把细节 secret 保存在链下，只公开一个 commitment
- **可扩展性**：后面你可以继续增加新的 per-party map，而不用改索引模型

但它**不会自动**带来下面这些能力：

- `partyId` 的唯一性保证
- 只允许正确用户更新指定 entry 的授权机制
- 对 stale proof 的保护
- approval 的聚合逻辑

这些都必须显式设计。

### A minimal initialization circuit

下面这个初始化 circuit 只用了 reference primer 里已经确认过的语法：

```compact
export circuit initialize(t: Uint<32>): [] {
  threshold = t;
  memberCount = 0;
  stateVersion = 0;
}
```

这里我故意保持得很小。在 Midnight 上，教程里合约的 public state 越直观越好。如果部署时只需要 `threshold` 这一个策略参数，那就别加别的。

### Why not keep full per-user private state in the ledger?

因为 ledger 是公开的协调状态。如果参与者真实的 approval 细节、spend limit 或本地决策理由是敏感信息，就应该把这些内容放进由 witness 提供的 private input，只在 ledger 里保留一个 commitment 或派生出来的 public flag。

Compact 文档里还有一个关于 witness 的重要警告：**不要信任 witness code 本身**。任何 DApp 都可以给 witness function 提供任意实现。这意味着你的 circuit 必须把 witness 输出视为不可信输入，并在 circuit 内部对它施加约束。这个警告对多方设计尤其关键，因为本质上每个用户都在提供自己的 private input 路径。

## Using a map of commitments keyed by party identifier

一个多方 treasury 通常至少需要为每个参与者维护一个 commitment。通用模式是：

- public ledger 存储 `commitments[partyId]`
- 用户把 secret preimage 保存在链下
- 后续 circuit 在不泄露 preimage 的前提下，证明关于它的陈述

一个直观的 join witness declaration，可以把 client 想提交的值打包起来：

```compact
witness JoinRequest(): [Bytes<32>, Bytes<32>, Uint<32>];
```

可以把它理解成：

- `Bytes<32>`：`partyId`
- `Bytes<32>`：`commitment`
- `Uint<32>`：`observedVersion`

这样你就能得到一个 `join` circuit 骨架：

```compact
export circuit join(): [] {
  const [partyId, commitment, observedVersion] = JoinRequest();

  // The circuit should constrain observedVersion against stateVersion.
  // It should also ensure that the participant has not already joined,
  // then write the new values into the ledger maps and increment:
  // - memberCount
  // - stateVersion
  //
  // Consult the current Compact map-access documentation for the exact
  // syntax of reading and writing Map<K, V> entries.
}
```

这里我刻意没有去编造 `Map` 的访问语法。bounty 明确说了，不能编译的 Compact 代码会直接失去资格，而且 issue 要求必须用真实语法，不能自己发明操作符。正确做法是：把**数据模型**和**状态迁移逻辑**讲清楚，同时把具体的 map 读写语法视为版本敏感内容，在发布仓库前直接去当前文档核实。权威来源依然是语言参考文档：[Compact language reference](https://docs.midnight.network/develop/reference/compact/lang-ref)。

### Why the map is keyed by `partyId`, not by array position

在小示例里，用数组下标看起来很顺手；但一旦进到真实部署系统，它会变得很脆弱：

- 它强制要求全局排序
- 它会让晚加入变复杂
- 它会让删除或替换成员变得别扭
- 它会把实现细节泄露给每个 client

用 `partyId` 作为 key 可以避开这些问题。每个参与者都只需要关心“我的记录”，而不用和别人协调 index。尤其是当多个 client 独立地绑定到同一个 deployment 时，这一点更重要。

### Recommended per-party maps

对一个私密 multi-sig treasury 来说，比较实用的一组 map 是：

- `joined: Map<Bytes<32>, Boolean>`  
  标记该 party 是否是成员。

- `commitments: Map<Bytes<32>, Bytes<32>>`  
  该成员当前的 commitment。

- `approvalNonce: Map<Bytes<32>, Uint<32>>`  
  每个 party 自己的 replay-protection counter，用于 approval 或 update。

如果你愿意，还可以继续加更多，但这三个已经足够把模式讲明白：

- 成员资格
- 私密状态 commitment
- 防重放

一个经验法则是：尽量分离关注点，不要把多种语义塞进同一个值里。比如，如果合约需要公开协调“是否为成员”和“当前 approval nonce”这两个事实，就不要把它们都藏进一个 opaque commitment 里。

### Commitment design in practice

合约不需要知道 commitment 背后完整的私密结构，它只需要知道后续 circuit 会对哪些内容做证明。以 treasury 为例，private preimage 可能包含：

- member-local secret
- proposal identifier
- per-member approval secret
- 最新的 per-member nonce

这样一来，public commitment 就可以在一个理解 witness 的 circuit 里重新计算，并与 `commitments[partyId]` 做检查。成员于是就能在不暴露底层 secret 的情况下，证明自己状态的连续性。

这里最重要的架构点是：**map 负责动态成员管理，commitment 负责隐私。** 两者缺一不可。

## Letting multiple users join with `findDeployedContract`

合约侧定义的是共享状态机，但*加入流程*发生在 client 代码里。这个 bounty 特别要求展示多个用户如何通过 `findDeployedContract` 加入一个已部署合约。helper 的具体名字和调用签名取决于 SDK 版本，所以这里我会描述稳定的模式，并明确指出哪些部分和版本相关。

每个参与者在 client 侧的大致流程是：

1. 获取已部署合约地址或 deployment handle
2. 加载生成出来的合约 metadata/driver
3. 绑定参与者自己的 witness context
4. 发现或连接到已部署的合约实例
5. 调用 `join()` 或其他成员加入 circuit
6. 持久化保存参与者自己的 private local state

伪代码如下：

```ts
// Assumption: your project uses the generated driver for the Compact contract
// plus the contract-discovery helper referenced by Midnight examples.
// Verify the exact API name/signature against your installed SDK version and
// current Midnight docs before publishing the repository.

import { findDeployedContract } from "your-midnight-client-layer";
import contractInfo from "./artifacts/contract-info.json";

async function attachAsParticipant({
  deploymentRef,
  witnessContext,
}: {
  deploymentRef: string;
  witnessContext: unknown;
}) {
  const treasury = await findDeployedContract({
    deploymentRef,
    contractInfo,
    witnessContext,
  });

  await treasury.join();
  return treasury;
}
```

这段代码的重点不在于 import path 的精确写法，而在于交互的**形状**：

- **同一个已部署合约**
- **不同的 witness context**
- **不同用户调用同一个导出的 circuit**

这就是 Midnight 上 N 方交互的本质。

### Why `findDeployedContract` matters

双人教程可以偷个懒：让部署者直接把实例传给另一个脚本就行。但真实应用不能这么做。独立用户需要在稍后的时间、不同 session、甚至不同机器上，发现并绑定到同一个 deployment。

一旦你把这个前提显式建模出来，几个设计后果就很自然了：

- 合约不能只依赖部署时就固定好的 participant 列表
- 成员资格必须体现在合约状态里
- 所有用户都必须读取同一份 public coordination field
- 每个会修改状态的调用都必须容忍“别的用户先一步更新了合约”这种情况

这也是为什么 `stateVersion` 字段会变得必不可少。

### Participant-specific witness contexts

每个用户都应该有一个只包含自己私密数据的 witness context：

- 与其 commitme 对应的 secret