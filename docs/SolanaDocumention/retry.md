# 重试交易

## 重试交易

在 Solana 网络中，看似有效的交易有时会在被包含进区块之前被丢弃。这种情况常发生在网络拥塞期间，此时 RPC 节点无法将交易重新广播给[领导者](https://solana.com/docs/terminology#leader)节点。对于最终用户来说，他们的交易可能看起来完全消失了。虽然 RPC 节点配备了通用重播算法，但应用程序开发人员也可以根据需要开发自己的自定义重播逻辑。

## 简述

- 节点将尝试使用通用算法重新广播交易
- 应用程序开发人员可以实现自己的自定义重播逻辑
- 开发人员应该利用 `sendTransaction` JSON-RPC 方法上的 `maxRetries` 参数
- 开发人员应启用预检检查以在提交交易之前引发错误
- 在重新签署任何交易之前，确保初始交易的区块哈希已过期非常重要

## 交易过程

### 客户如何提交交易

在 Solana 中，没有内存池的概念。所有交易，无论是通过编程方式发起还是由最终用户发起，都会直接路由到领导者节点，以便将它们打包成区块。有两种主要方式可以将交易发送给领导者：

1. 通过 RPC 服务器和 `sendTransaction` JSON-RPC 方法进行代理
2. 通过 [TPU Client ](https://docs.rs/solana-client/latest/solana_client/tpu_client/index.html) 直接与领导者节点联系
   绝大多数最终用户将通过 RPC 服务器提交交易。当客户端提交交易时，接收 RPC 节点将依次尝试将交易广播给当前和下一个领导者节点。在领导者节点处理交易之前，除了客户端和中继 RPC 节点所知之外，没有任何交易记录。对于 TPU 客户端，重播和领导者节点转发完全由客户端软件处理。

![image-20240525140005154](https://solana-developer-content.vercel.app/assets/docs/rt-tx-journey.png)

### RPC 节点如何广播交易

RPC 节点通过 `sendTransaction` 接收到交易后，会将交易转换为 UDP 数据包，然后转发给相关领导者。 UDP 允许验证者快速相互通信，但不提供有关交易交付的任何保证。

由于 Solana 的领导者节点时间表在每个 [epoch](https://solana.com/docs/terminology#epoch)（约 2 天）之前都是已知的，因此 RPC 节点将直接向当前和下一个领导者节点广播其交易。这与以太坊等其他八卦协议形成鲜明对比，后者在整个网络中随机且广泛地传播交易。默认情况下，RPC 节点将尝试每两秒将交易转发给领导者节点，直到交易最终确定或交易的区块哈希过期（150 个区块或截至撰写本文时约 1 分 19 秒）。如果未完成的重播队列大小大于 [10,000 个交易](https://github.com/solana-labs/solana/blob/bfbbc53dac93b3a5c6be9b4b65f679fdb13e41d9/send-transaction-service/src/send_transaction_service.rs#L20)，则新提交的交易将被丢弃。 RPC 操作员可以调整命令行[参数](https://github.com/solana-labs/solana/blob/bfbbc53dac93b3a5c6be9b4b65f679fdb13e41d9/validator/src/main.rs#L1172)来更改此重试逻辑的默认行为。

当 RPC 节点广播交易时，它将尝试将交易转发到领导者节点的[交易处理单元 (TPU)](https://github.com/solana-labs/solana/blob/cd6f931223181d5a1d47cba64e857785a175a760/core/src/validator.rs#L867)。 TPU 分五个不同的阶段处理交易：

- [Fetch Stage](https://github.com/solana-labs/solana/blob/cd6f931223181d5a1d47cba64e857785a175a760/core/src/fetch_stage.rs#L21)
- [SigVerify Stage](https://github.com/solana-labs/solana/blob/cd6f931223181d5a1d47cba64e857785a175a760/core/src/tpu.rs#L91)
- [Banking Stage](https://github.com/solana-labs/solana/blob/cd6f931223181d5a1d47cba64e857785a175a760/core/src/banking_stage.rs#L249)
- [Proof of History Service](https://github.com/solana-labs/solana/blob/cd6f931223181d5a1d47cba64e857785a175a760/poh/src/poh_service.rs)
- [Broadcast Stage](https://github.com/solana-labs/solana/blob/cd6f931223181d5a1d47cba64e857785a175a760/core/src/tpu.rs#L136)

![](https://solana-developer-content.vercel.app/assets/docs/rt-tpu-jito-labs.png)

在这五个阶段中，Fetch Stage 负责接收交易。在获取阶段，验证器将根据三个端口对传入交易进行分类：

- tpu 处理常规交易，例如代币转移、NFT 铸币和程序指令
- tpu_vote 专注于投票交易
- tpu_forwards 将未处理的数据包转发给下一个领导者节点

有关 TPU 的更多信息，请参阅 [Jito Labs](https://jito-labs.medium.com/solana-validator-101-transaction-processing-90bcdc271143) 撰写的这篇精彩文章。

## 交易如何被丢弃

在整个交易过程中，在某些情况下交易可能会无意中从网络中丢失。

### 在处理交易之前

如果网络丢弃一笔交易，它很可能会在领导者节点处理该交易之前这样做。 UDP 数据包丢失是发生这种情况的最简单原因。在网络负载严重的时期，验证器也可能会因处理所需的大量交易而不堪重负。虽然验证器可以通过 tpu_forwards 转发剩余交易，但可以转发的数据量是有限的。此外，每个转发仅限于验证器之间的单跳。也就是说，`tpu_forwards` 端口上收到的交易不会转发到其他验证器。

还有两个鲜为人知的原因可能会导致交易在处理之前被删除。第一个场景涉及通过 RPC 池提交的交易。有时，RPC 池的一部分可能会远远领先于池的其余部分。当池中的节点需要协同工作时，这可能会导致问题。在这个例子中，交易的 recentBlockhash 是从池的高级部分（后端 A）查询的。当交易被提交到池的滞后部分（后端 B）时，节点将无法识别高级区块哈希并会丢弃交易。如果开发人员对 `sendTransaction` 启用预检检查，则可以在交易提交时检测到这一点。

![image-20240525140425764](https://solana-developer-content.vercel.app/assets/docs/rt-dropped-via-rpc-pool.png)

​

临时网络分叉也可能导致交易丢失。如果验证者在银行阶段重播其区块的速度很慢，它可能最终会创建一个少数派分叉。当客户端构建交易时，交易可能会引用仅存在于少数派叉上的 `recentBlockhash`。提交交易后，集群可以在处理交易之前切换出少数派分叉。在这种情况下，由于找不到区块哈希，交易被丢弃。

![image-20240525140557114](https://solana-developer-content.vercel.app/assets/docs/rt-dropped-minority-fork-pre-process.png)

### 交易处理之后和最终确定之前

如果交易引用来自少数分叉的 `recentBlockhash`，则该交易仍然有可能被处理。然而，在这种情况下，它将由少数分叉上的领导者节点处理。当这个领导者节点试图与网络的其他部分共享其处理过的交易时，它将无法与不识别少数分叉的大多数验证者达成共识。此时，交易将在最终确定之前被删除。

![image-20240525140633521](https://solana-developer-content.vercel.app/assets/docs/rt-dropped-minority-fork-post-process.png)

## 处理丢弃的交易

虽然 RPC 节点会尝试重新广播交易，但它们采用的算法是通用的，并且通常不适合特定应用程序的需求。为了应对网络拥塞，应用程序开发人员应该定制自己的重播逻辑。

### 深入了解 sendTransaction

`sendTransaction` RPC 方法是开发人员用于提交交易的主要工具。 `sendTransaction` 只负责将交易从客户端中继到 RPC 节点。如果节点收到交易，`sendTransaction` 将返回可用于跟踪交易的交易 id。成功的响应并不表明交易是否将由集群处理或完成。

### 请求参数

- `stringtransaction: string `- 完全签名的交易，作为编码字符串
- (optional) `configuration: object`
  - `skipPreflight`: `boolean` - 如果为 true，则跳过预检交易检查（默认值：false）
  - (optional) `preflightCommitment`: `string` - 用于针对银行槽进行预检模拟的承诺级别（默认值：“最终确定”）。
  - (optional) `encoding`: `string` - 用于交易数据的编码。 “base58”（慢）或“base64”。 （默认值：“base58”）。
  - (optional) `maxRetries`: `usize` - RPC 节点重试向领导者节点发送交易的最大次数。如果未提供此参数，RPC 节点将重试交易，直到交易完成或区块哈希过期。

**返回值:**

- `transaction id`: `string` - 嵌入交易中的第一个交易签名，作为 base-58 编码的字符串。此交易 ID 可以与 `getSignatureStatuses` 一起使用来轮询状态更新。

## 自定义重播逻辑

Solana 提供了 `sendTransaction` API 的 `maxRetries` 参数，允许开发人员自定义交易重播逻辑。`maxRetries` 将覆盖 RPC 节点的默认重试逻辑，允许开发人员在合理的范围内手动控制重试过程。

手动重试交易的常见模式涉及临时存储来自 getLatestBlockhash 的 `LastValidBlockHeight`。一旦隐藏，应用程序就可以轮询集群的区块高度并以适当的时间间隔手动重试交易。在网络拥塞时，将 `maxRetries` 设置为 0 并通过自定义算法手动重播是有利的。虽然某些应用程序可能采用指数退避算法，但其他应用程序（例如 Mango）选择以恒定间隔不断重新提交交易，直到发生超时。

```js
import {
  Keypair,
  Connection,
  LAMPORTS_PER_SOL,
  SystemProgram,
  Transaction,
} from "@solana/web3.js";
import * as nacl from "tweetnacl";

const sleep = async (ms: number) => {
  return new Promise((r) => setTimeout(r, ms));
};

(async () => {
  const payer = Keypair.generate();
  const toAccount = Keypair.generate().publicKey;

  const connection = new Connection("http://127.0.0.1:8899", "confirmed");

  const airdropSignature = await connection.requestAirdrop(
    payer.publicKey,
    LAMPORTS_PER_SOL
  );

  await connection.confirmTransaction({ signature: airdropSignature });

  const blockhashResponse = await connection.getLatestBlockhashAndContext();
  const lastValidBlockHeight = blockhashResponse.context.slot + 150;

  const transaction = new Transaction({
    feePayer: payer.publicKey,
    blockhash: blockhashResponse.value.blockhash,
    lastValidBlockHeight: lastValidBlockHeight,
  }).add(
    SystemProgram.transfer({
      fromPubkey: payer.publicKey,
      toPubkey: toAccount,
      lamports: 1000000,
    })
  );
  const message = transaction.serializeMessage();
  const signature = nacl.sign.detached(message, payer.secretKey);
  transaction.addSignature(payer.publicKey, Buffer.from(signature));
  const rawTransaction = transaction.serialize();
  let blockheight = await connection.getBlockHeight();

  while (blockheight < lastValidBlockHeight) {
    connection.sendRawTransaction(rawTransaction, {
      skipPreflight: true,
    });
    await sleep(500);
    blockheight = await connection.getBlockHeight();
  }
})();
```

当应用程序通过 `getLatestBlockhash` 获取最新区块哈希时，应用程序应指定其预期的承诺级别。通过将其承诺设置为 `confirmed`（已确认投票）或 `finalized` 最终确定（ `confirmed` 确认后约 30 个区块），应用程序可以避免从少数分叉中轮询区块哈希。

如果应用程序可以访问负载均衡器后面的多个 RPC 节点，可以选择在不同节点之间分配工作负载。对于处理数据密集型请求（例如 `getProgramAccounts`）的节点，其响应速度可能较慢，因此不适合转发交易。对于处理时间敏感交易的应用程序，建议使用专用的 RPC 节点，仅用于处理 `sendTransaction` 请求。

### 跳过预检的成本

默认情况下，`sendTransaction` 将在提交交易之前执行三项预检检查。具体来说，`sendTransaction` 将：

- 验证所有签名均有效
- 检查引用的区块哈希是否在最近 150 个区块内
- 根据 `preflightCommitment` 指定的银行槽模拟交易

如果这三个预检检查中的任何一个失败，`sendTransaction` 将在提交交易之前引发错误。预检检查通常是丢失交易和允许客户端优雅地处理错误之间的区别。为了确保解决这些常见错误，建议开发人员将 `skipPreflight` 设置为 false。

### 何时重新签署交易

即使经过多次重播尝试，有时仍需要客户端重新签署交易。在重新签署任何交易之前，必须确保原始交易的区块哈希已失效。如果原始区块哈希仍然有效，则两笔交易都可能被网络接受，导致交易混乱。

在 Solana 中，只要交易引用的区块哈希早于从 `getLatestBlockhash` 获取的 `lastValidBlockHeight`，就可以安全地丢弃该交易。开发人员可以通过查询 `getEpochInfo` 并与响应中的 `blockHeight` 进行比较来跟踪 `lastValidBlockHeight`。一旦区块哈希失效，客户端可以使用新获取的区块哈希重新签名交易。
