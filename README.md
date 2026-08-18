# SecurEscrow
Decentralized Time-Locked Escrow 

1. 项目背景
开发一个去中心化的资金托管（Escrow）系统。买方将代币存入智能合约，设置一个时间锁和解锁条件（如达成某项服务）。卖方在条件满足且时间锁到期后可以提取资金；若超时未达成条件，买方可撤回资金。
特殊审计要求： 第一版（v1）需故意留下常见的安全漏洞，随后用 TypeScript 写 Exploit 脚本攻击它，最后在第二版（v2）中用安全的 Rust 代码修复它。

2. 核心功能与角色
买方 (Depositor)： 能够初始化 Escrow，存入资金（SOL 或 SPL Token），设定时间锁（Timestamp）和指定收款人。

卖方 (Beneficiary)： 能够在时间锁到期且状态为“已批准”时，调用提取资金接口。

合约 (Program)： 负责安全地持有资金，校验签名者、时间戳、账户所有权及状态流转。  

3. Rust 合约端需求 (Smart Contract)
指令 (Instructions) 需求：

initialize_escrow：初始化托管账户，记录金额、时间锁、买卖方公钥。

deposit_funds：买方将代币转入合约的 PDA（程序派生地址）金库。

approve_release：买方签署同意，将状态标记为“可提取”。

withdraw_funds：卖方提取资金（需校验：状态为已批准 + 当前链上时间 > 时间锁）。

refund：若超时且未批准，买方可取回资金。

埋点漏洞（用于后续审计练习）：

漏洞 1：缺失签名者校验（Missing Signer Check）。 approve_release 中不校验调用者是否真的是买方。

漏洞 2：错误的 PDA 校验。 withdraw_funds 中没有验证传入的 Vault 账户是否与该 Escrow 绑定，导致任意金库被排空。

漏洞 3：算术溢出。 如果存入金额逻辑使用原生 + 而非 checked_add（在特定 Rust 版本下）。

4. TypeScript 测试与利用套件需求 (Exploit SDK)
模块 1：客户端 SDK

封装与合约交互的强类型 API 工具类（EscrowClient）。

实现账户数据的自动反序列化，将链上的 Buffer 数据转为易读的 TS 对象。

模块 2：常规单元测试 (Happy Path)

编写脚本模拟正常的初始化、存款、批准和提取流程，确保业务逻辑畅通。

模块 3：审计漏洞利用套件 (PoC Suite)

编写 attack_missing_signer.ts：以黑客身份构建交易，强行调用 approve_release（不提供买方签名），验证是否能篡改状态。

编写 attack_fake_vault.ts：构造一个虚假的 PDA 账户传入 withdraw_funds 指令，测试能否骗过合约转移资金。

编写测试断言（Assertions），预期这些恶意交易在 v1 合约中会成功，但在 v2 修复版中会抛出特定的自定义错误。

5. 实施阶段计划
Phase 1: 环境搭建与基础语法

安装 Rust, Solana CLI, Anchor, Node.js, Yarn。

搭建 Anchor 工程，熟悉目录结构。

Phase 2: 编写不安全的 Rust v1 合约

定义账户结构（Account Structs）。

实现 5 个核心业务逻辑的 Instruction，故意忽略部分安全性校验。

Phase 3: 编写 TypeScript SDK 与测试用例

编写 TS 类型定义，生成 IDL 的类型绑定。

编写标准的业务流程测试，确保合约“能用”。

Phase 4: 编写攻击脚本 (Hack Time)

运用审计思维，用 TS 编写恶意 payload 攻击自己的合约并成功盗取资金。

Phase 5: 修复与防守 (Rust v2)

回到 Rust 代码，引入 Signer<'info> 校验、PDA seeds 约束、以及安全数学计算。

重新运行 TS 攻击脚本，验证漏洞已被成功拦截。


## 目标技能
掌握的 Rust 技能（防守侧）
所有权与生命周期（Ownership & Lifetimes）： 理解账户数据的引用与借用，这在防范 Solana 的“账户混淆（Account Confusion）”漏洞时至关重要。

类型安全与宏（Macros & Traits）： 理解底层框架（如 Anchor）是如何展开代码的，审计时能看透宏背后的隐式校验（如 #[derive(Accounts)]）。

错误处理（Result & Option）： 熟练使用 ? 操作符和自定义错误，理解未处理错误可能导致的逻辑绕过。

数学与溢出保护： 强制使用 checked_add 等安全数学操作，防范算术溢出。

掌握的 TypeScript 技能（攻击侧 / 验证侧）
强类型接口（Interfaces & Generics）： 定义链上数据的 TS 序列化/反序列化结构（如 Borsh 布局）。

异步编程（Async/Await & Promises）： 处理多步骤的交易组装、签名与广播。

测试框架与 Mocking： 使用 Mocha/Chai 或 Jest 构建端到端的“漏洞复现（PoC）”脚本。

RPC 交互深度： 熟练使用 @solana/web3.js，手动构建恶意的 Instruction（指令）来攻击合约。
