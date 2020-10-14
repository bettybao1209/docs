# Poly Network

Poly Network基于侧链/中继模式，采用双层结构设计，使用Poly中继链作为跨链协调器，多条异构链作为跨链交易执行器，Relayer作为跨链信息的搬运工，通过解决跨链信息的有效性、安全性和事务性等问题，实现了一套安全、易用、高效的跨链体系。

## 特性
* 轻量级跨链协议
* 接入简单方便
* 同时支持同构链和异构链
* 支持跨链事务强一致性和最终一致性
* 资产跨链超级，同时支持资产跨链和任意信息跨链
* 跨链协议安全可靠，以密码学为基石
* 支持异构链协议范围广（BTC/ETH/NEO/Ontology/Cosmos）
* 中继链是开放式准入的公链

更多有关Poly Network的信息，请点击[此处](https://github.com/polynetwork/docs/blob/master/README_CN.md)查看。

# Neo跨链设计

## 概述

目前在Neo上有两种方式可以实现跨链操作：一种是通过节点如Neo-CLI和Neo-GUI等调用合约发送跨链交易，另一种是通过SDK构造跨链交易。

## 跨链的实现

以资产跨链为例，用户把资产在原链上锁定，之后在目标链上发行映射资产，同时可以在目标链上申请提现，最后在原链上解锁。要实现该过程，则需要目标链可以验证原链上发生的行为，即验证原链上确实锁定了一定数量的原资产。

非资产类的信息跨链也是如此，需要目标链验证原链上的信息变化，从而做出相应的变化。

这种验证过程目前都是通过merkle证明的形式实现的，即原链将其上发生的行为存储下来，并构造一棵merkle tree，然后将merkle tree的树根root和该行为的proof提交给目标链。目标链根据提交的merkle root验证proof的合法性，从而确定原链上发生的行为。

## 架构

<div align=center><img width="290" height="458" src="CrossChainStructure.png"/></div>

上图显示了Neo跨链生态的架构，从上到下分别是Neo链、Neo链的Relayer、中继链Poly、目标链的Relayer和目标链。简单来说，用户在Neo链上发出的跨链交易的证明会经由Relayer传递到Poly，再由目标链的Relayer传递到目标链，目标链验证Neo链上的交易证明并执行相应的交易。反之亦然。

生态中的角色定义如下：

- **中继链 Poly**：中继链是整个生态中的重要部分，每个节点由不同的个人或组织运行，有自己独特的治理模式和信任机制，它负责将各个链连接到一起和跨链信息的传递。
- **Relayer**：每条链都有自己的Relayer，它们负责把交易等跨链信息搬运到中继链或从中继链搬运到源链，并且它们会在这个过程中获取收益。
- **应用**：应用是指开发跨链业务的人或组织，任何人都可以部署跨链合约来构建跨链应用，然后把应用公开出去招揽用户。
- **用户**：对跨链生态来说，最重要的就是用户，通过调用具有跨链功能的应用，实现Neo到以太坊等链的跨链业务。

## Neo链和中继链之间的区块头同步

<div align=center><img src="HeaderSync.png"/></div>

上图展示了Neo链和中继链之间同步区块头的具体流程：

一方面，Neo链的共识节点由Neo持有者实时投票选举产生，理论上每个区块的验证者集合都可能变化。若验证者集合保持不变，则不需要同步该区块头，只需要同步关键区块头（即包含切换了验证者集合后的区块头），这样可以大大减少需要同步的区块头数量。同时由于Neo的merkle root不包含在区块头中，而是由共识节点独立签名并广播，所以当有跨链交易发生时，Relayer会把这些交易所在区块的merkle root和proof以及区块高度转发给中继链，中继链仅更新一下当前已同步的Neo链的高度并转发merkle root和proof。

另一方面，中继链区块头由Relayer同步到Neo链，Neo链的跨链管理合约会验证区块头的合法性，并存储区块头，该链的其它任何合约都可以从该合约中读取同步的区块头。

## Neo链和中继链之间的跨链交易

<div align=center><img src="Neo2Relay.png"/></div>

从Neo链到目标链的跨链流程：

1. 用户在Neo链上的业务合约中发起跨链交易；
2. 业务合约调用跨链管理合约的跨链接口，跨链管理合约处理跨链请求，分配唯一自增ID，发出跨链事件；
3. 共识节点会生成该跨链交易所在区块的merkle root和该交易的merkle proof；
4. Neo的Relayer捕获跨链事件后通过RPC调用获取上述merkle root和merkle proof以及区块高度并提交到中继链；
5. 中继链验证merkle proof的合法性，将跨链交易的信息以事件的形式广播，目标链的Relayer会监听这些事件并把发往自己链的交易捕获到，然后转发到对应目标链；
6. 目标链的跨链管理合约验证中继链merkle proof的合法性，验证通过则说明原链上的跨链信息合法，目标链的跨链管理合约会调用相应的业务合约，执行目标链上的业务逻辑。

<div align=center><img src="Relay2Neo.png"/></div>

从目标链返回到Neo链的跨链流程：

1. 用户在目标链上的业务合约中发起跨链交易；
2. 目标链的Relayer将跨链交易信息转发到中继链，调用中继链的跨链管理合约；
3. 跨链管理合约验证交易信息后，将验证后的信息存下来，并构造新的merkle tree，将树根写入中继链的区块头中，并生成跨链信息的merkle proof；
4. Neo的Relayer会监听中继链，将区块头和proof提交到Neo链；
5. Neo链的跨链管理合约会读取中继链区块头，验证中继链的proof，然后调用业务合约执行对应的逻辑。

## Neo跨链合约开发

合约的跨链对开发者来说只需要关注一个跨链接口，也就是跨链管理合约（CCMC）的`CrossChain`接口，该接口将链A上已经执行的业务存入Merkle tree，会有矿工生成该跨链交易的Merkle proof，并将其提交到链B的跨链管理合约中，该跨链管理合约会验证Merkle proof，并按照参数调用智能合约B中对应的方法。

`CrossChain`接口的实现如下：

```C#
public static bool CrossChain(BigInteger toChainID, byte[] toChainAddress, byte[] functionName, byte[] args)
```

**参数设置：**  

- toChainId: 目标链的链ID
- toChainAddress: 跨链需要调用的目标链上的合约
- functionName: 跨链资产数量
- args: 序列化之后的目标合约输入参数

### 跨链合约开发示例（新发行的NEP5资产）

要实现新发行的NEP5资产的跨链流通，需要在NEP5标准接口基础上增加`Lock`和`Unlock`接口。用户在源链调用`Lock`接口将资产锁定在智能合约A中，该接口同时调用跨链管理合约实现跨链调用智能合约B中的`Unlock`接口，并在目标链中将智能合约B中的资产释放给该用户。反之亦然。

`Lock`接口实现：

```C#
public static bool Lock(byte[] fromAddress, BigInteger toChainId, byte[] toAddress, BigInteger amount)
{
    // check parameters
    if (!IsAddress(fromAddress))
    {
        Runtime.Notify("The parameter fromAddress SHOULD be a legal address.");
        return false;
    }
    if (toAddress.Length == 0)
    {
        Runtime.Notify("The parameter toAddress SHOULD not be empty.");
        return false;
    }
    if (amount < 0)
    {
        Runtime.Notify("The parameter amount SHOULD not be less than 0.");
        return false;
    }
    // more checks
    if (!Runtime.CheckWitness(fromAddress))
    {
        Runtime.Notify("Authorization failed.");
        return false;
    }
    if (IsPaused())
    {
        Runtime.Notify("The contract is paused.");
        return false;
    }
    // lock asset
    StorageMap asset = Storage.CurrentContext.CreateMap(nameof(asset));
    var balance = asset.Get(fromAddress).AsBigInteger();
    if (balance < amount)
    {
        Runtime.Notify("Not enough balance to lock.");
        return false;
    }
    StorageMap contract = Storage.CurrentContext.CreateMap(nameof(contract));
    var totalSupply = contract.Get("totalSupply").AsBigInteger();
    if (totalSupply < amount)
    {
        Runtime.Notify("Not enough supply to lock.");
        return false;
    }
    asset.Put(fromAddress, balance - amount);
    contract.Put("totalSupply", totalSupply - amount);
    // construct args for the corresponding asset contract on target chain
    var inputBytes = SerializeArgs(toAddress, amount);
    var toContract = GetContractAddress(toChainId);
    // constrct params for CCMC
    var param = new object[] { toChainId, toContract, "unlock", inputBytes };
    // dynamic call CCMC
    var ccmc = (DynCall)CCMCScriptHash.ToDelegate();
    var success = (bool)ccmc("CrossChain", param);
    if (!success)
    {
        Runtime.Notify("Failed to call CCMC.");
        return false;
    }
    LockEvent(toChainId, fromAddress, toAddress, amount);
    return true;
}
```

**参数设置：**  

- fromAddress: 跨链调用发起地址，矿工费从该地址扣除
- toChainId: 目标链的链ID
- toAddress: 目标链上的用户地址
- amount: 跨链资产数量

在上面的代码中可以看到该接口先冻结用户在链A上的需要跨链的数量的资产，并减少该资产在链A上的流通量，然后调用Neo跨链管理合约的`CrossChain`接口。

在`Lock`接口的代码中可以看到该方法调用了目标合约的`Unlock`接口，`Unlock`接口实现如下：

```C#
private static bool Unlock(byte[] inputBytes, byte[] fromContract, BigInteger fromChainId, byte[] caller)
{
    //only allowed to be called by CCMC
    if (caller.AsBigInteger() != CCMCScriptHash.AsBigInteger())
    {
        Runtime.Notify("Only allowed to be called by CCMC");
        return false;
    }
    byte[] storedContract = GetContractAddress(fromChainId);
    // check the fromContract is stored, so we can trust it
    if (fromContract.AsBigInteger() != storedContract.AsBigInteger())
    {
        Runtime.Notify(fromContract);
        Runtime.Notify(fromChainId);
        Runtime.Notify(storedContract);
        Runtime.Notify("From contract address not found.");
        return false;
    }
    // parse the args bytes constructed in source chain proxy contract, passed by multi-chain
    object[] results = DeserializeArgs(inputBytes);
    var toAddress = (byte[])results[0];
    var amount = (BigInteger)results[1];
    if (!IsAddress(toAddress))
    {
        Runtime.Notify("ToChain Account address SHOULD be a legal address.");
        return false;
    }
    if (amount < 0)
    {
        Runtime.Notify("ToChain Amount SHOULD not be less than 0.");
        return false;
    }
    // unlock asset
    StorageMap asset = Storage.CurrentContext.CreateMap(nameof(asset));
    var balance = asset.Get(toAddress).AsBigInteger();
    StorageMap contract = Storage.CurrentContext.CreateMap(nameof(contract));
    var totalSupply = contract.Get("totalSupply").AsBigInteger();
    asset.Put(toAddress, balance + amount);
    contract.Put("totalSupply", totalSupply + amount);
    UnlockEvent(toAddress, amount);
    return true;
}
```

**参数设置：**  

- inputBytes: 序列化之后的合约参数
- fromContract: 发起跨链的源链上的合约
- fromChainId: 源链ID
- caller: 该方法的调用者，不用传入，合约自动补充该参数

该接口会先校验调用是不是来自跨链管理合约，然后校验源链上的合约地址是否可信，接着反序列化出所需参数，最后解锁链B上的资产给用户及增加该资产总量。


### 跨链合约开发示例（已发行的NEP5资产）

由于无法给已发行的NEP5资产添加`Lock`、`Unlock`方法，因此需要额外实现一个代理合约，相当于原先NEP5合约的补充功能，实现了跨链协议中的主要接口，也可以使用现有的代理合约实现跨链，避免重复开发。

以下为代理合约必须实现的接口：

#### BindProxyHash

```C#
public static bool BindProxyHash(BigInteger toChainId, byte[] targetProxyHash)
{
    if (!Runtime.CheckWitness(Operator)) return false;
    StorageMap proxyHash = Storage.CurrentContext.CreateMap(nameof(proxyHash));
    proxyHash.Put(toChainId.AsByteArray(), targetProxyHash);
    return true;
}
```

**参数设置：**  

- toChainId: 目标链的链ID
- targetProxyHash: 跨链需要调用的目标链上的代理合约

该接口绑定目标链上的代理合约hash，或者实现了跨链接口的合约hash，在NEP5向其他链发起跨链的时候会将该目标链代理合约作为中转站，进而将NEP5转到目标链的映射合约，当然在跨链开始前应该预先部署好映射合约。

#### BindAssetHash

```C#
public static bool BindAssetHash(byte[] fromAssetHash, BigInteger toChainId, byte[] toAssetHash, BigInteger initialAmount)
{
    if (!Runtime.CheckWitness(Operator)) return false;
    StorageMap assetHash = Storage.CurrentContext.CreateMap(nameof(assetHash));
    assetHash.Put(fromAssetHash.Concat(toChainId.AsByteArray()), toAssetHash);
    if (GetAssetBalance(fromAssetHash) != initialAmount)
    {
        Runtime.Notify("Initial amount incorrect.");
        return false;
    }
    StorageMap lockedAmount = Storage.CurrentContext.CreateMap(nameof(lockedAmount));
    lockedAmount.Put(fromAssetHash, initialAmount);
    BindAssetHashEvent(fromAssetHash, toChainId, toAssetHash, initialAmount);
    return true;
}
```

**参数设置：**  

- fromAssetHash: 源链上的资产合约hash
- toChainId: 目标链的链ID
- toAssetHash: 目标链上的资产合约hash
- initialAmount: 该资产初始锁定数量

该接口绑定NEP5资产合约地址和目标链映射合约，比如USDT地址，并设置初始锁定数量。

#### Lock

```C#
public static bool Lock(byte[] fromAssetHash, byte[] fromAddress, BigInteger toChainId, byte[] toAddress, BigInteger amount)
{
    // check parameters
    if (fromAssetHash.Length != 20)
    {
        Runtime.Notify("The parameter fromAssetHash SHOULD be 20-byte long.");
        return false;
    }
    if (fromAddress.Length != 20)
    {
        Runtime.Notify("The parameter fromAddress SHOULD be 20-byte long.");
        return false;
    }
    if (toAddress.Length == 0)
    {
        Runtime.Notify("The parameter toAddress SHOULD not be empty.");
        return false;
    }
    if (amount < 0)
    {
        Runtime.Notify("The parameter amount SHOULD not be less than 0.");
        return false;
    }
    // get the corresbonding asset on target chain
    var toAssetHash = GetAssetHash(fromAssetHash, toChainId);
    if (toAssetHash.Length == 0)
    {
        Runtime.Notify("Target chain asset hash not found.");
        return false;
    }
    // get the proxy contract on target chain
    var toContract = GetProxyHash(toChainId);
    if (toContract.Length == 0)
    {
        Runtime.Notify("Target chain proxy contract not found.");
        return false;
    }
    // transfer asset from fromAddress to proxy contract address, use dynamic call to call nep5 token's contract "transfer"
    byte[] currentHash = ExecutionEngine.ExecutingScriptHash; // this proxy contract hash
    var nep5Contract = (DynCall)fromAssetHash.ToDelegate();
    bool success = (bool)nep5Contract("transfer", new object[] { fromAddress, currentHash, amount });
    if (!success)
    {
        Runtime.Notify("Failed to transfer NEP5 token to proxy contract.");
        return false;
    }
    // construct args for proxy contract on target chain
    var inputBytes = SerializeArgs(toAssetHash, toAddress, amount);
    // constrct params for CCMC 
    var param = new object[] { toChainId, toContract, "unlock", inputBytes };
    // dynamic call CCMC
    var ccmc = (DynCall)CCMCScriptHash.ToDelegate();
    success = (bool)ccmc("CrossChain", param);
    if (!success)
    {
        Runtime.Notify("Failed to call CCMC.");
        return false;
    }
    // update locked amount
    StorageMap lockedAmount = Storage.CurrentContext.CreateMap(nameof(lockedAmount));
    BigInteger old = lockedAmount.Get(fromAssetHash).ToBigInteger();
    lockedAmount.Put(fromAssetHash, old + amount);
    LockEvent(fromAssetHash, toChainId, toAssetHash, fromAddress, toAddress, amount);
    return true;
}
```

**参数设置：**  

- fromAssetHash: 源链上的NEP5资产合约hash
- fromAddress: 跨链调用发起地址，矿工费从该地址扣除
- toChainId: 目标链的链ID
- toAddress: 目标链上的用户地址
- amount: 跨链资产数量

该接口会把NEP5资产锁定到代理合约账户中，并调用跨链管理合约的`CrossChain`接口从而发出跨链请求。

#### Unlock

```C#
private static bool Unlock(byte[] inputBytes, byte[] fromProxyContract, BigInteger fromChainId, byte[] caller)
{
    //only allowed to be called by CCMC
    if (caller.AsBigInteger() != CCMCScriptHash.AsBigInteger())
    {
        Runtime.Notify("Only allowed to be called by CCMC");
        return false;
    }
    byte[] storedProxy = GetProxyHash(fromChainId);
    // check the fromContract is stored, so we can trust it
    if (fromProxyContract.AsBigInteger() != storedProxy.AsBigInteger())
    {
        Runtime.Notify(fromProxyContract);
        Runtime.Notify(fromChainId);
        Runtime.Notify(storedProxy);
        Runtime.Notify("From proxy contract not found.");
        return false;
    }
    // parse the args bytes constructed in source chain proxy contract, passed by multi-chain
    object[] results = DeserializeArgs(inputBytes);
    var assetHash = (byte[])results[0];
    var toAddress = (byte[])results[1];
    var amount = (BigInteger)results[2];
    if (assetHash.Length != 20)
    {
        Runtime.Notify("ToChain Asset script hash SHOULD be 20-byte long.");
        return false;
    }
    if (toAddress.Length != 20)
    {
        Runtime.Notify("ToChain Account address SHOULD be 20-byte long.");
        return false;
    }
    if (amount < 0)
    {
        Runtime.Notify("ToChain Amount SHOULD not be less than 0.");
        return false;
    }
    // transfer asset from proxy contract to toAddress
    byte[] currentHash = ExecutionEngine.ExecutingScriptHash; // this proxy contract hash
    var nep5Contract = (DynCall)assetHash.ToDelegate();
    bool success = (bool)nep5Contract("transfer", new object[] { currentHash, toAddress, amount });
    if (!success)
    {
        Runtime.Notify("Failed to transfer NEP5 token to toAddress.");
        return false;
    }
    // update locked amount
    StorageMap lockedAmount = Storage.CurrentContext.CreateMap(nameof(lockedAmount));
    BigInteger old = lockedAmount.Get(assetHash).ToBigInteger();
    lockedAmount.Put(assetHash, old - amount);
    UnlockEvent(assetHash, toAddress, amount);
    return true;
}
```

**参数设置：**  

- inputBytes: 序列化之后的合约参数
- fromProxyContract: 发起跨链的源链上的代理合约hash
- fromChainId: 源链ID
- caller: 该方法的调用者，不用传入，合约自动补充该参数

该接口会被跨链管理合约调用，为用户释放原先锁定的NEP5资产，即从其他链返回的NEP5代币。

所以代理合约主要实现的是NEP5的注册、锁定与解锁，锁定就是用户将NEP5转到合约中，解锁只有跨链管理合约可以调用。代理合约可以自己部署，也可以使用现有的合约，多个NEP5资产可以使用一本代理合约。

### 如何将NEP5资产跨到其他链上

#### 收集资产在链上对应的合约地址

以某个NEP5资产为例，这种资产在链上的合约地址为：

-Neo链: 4da43995ee75be3931fd3abae06a3e6447d2febf (小端序)

-Ethereum链: 0x86B7Be11D77043E5aDe3858f2F694E4f89d4a941

### 执行跨链操作

#### 集成跨链功能的NEP5资产

如果该资产集成了跨链的Lock和Unlock接口，则可以直接调用该资产的Lock接口进行跨链资产转移：

```C#
public static bool Lock(byte[] fromAddress, BigInteger toChainId, byte[] toAddress, BigInteger amount)
```

**参数设置：**  

- fromAddress: 5392ebf880d9a7119b17b2f1adaebc5f71c28e75 (Neo链地址APPmjituYcgfNxjuQDy9vP73R2PmhFsYJR的script hash，小端序)  
- toChainId: 2 (Ethereum链的编码)  
- toAddress: 0x0B24aBDd39185055311aaa27082F9dEb294A7255 (Ethereum链地址)  
- amount: 10000 (跨链数量)

#### 未集成跨链功能的NEP5资产

如果该资产未集成跨链的Lock和Unlock接口，则需要调用该资产对应的代理合约的Lock接口：

```C#
public static bool Lock(byte[] fromAssetHash, byte[] fromAddress, BigInteger toChainId, byte[] toAddress, BigInteger amount)
```

**参数设置：**  

- fromAssetHash: 4da43995ee75be3931fd3abae06a3e6447d2febf (NEP5资产的script hash，小端序)  
- fromAddress: 5392ebf880d9a7119b17b2f1adaebc5f71c28e75 (Neo链地址APPmjituYcgfNxjuQDy9vP73R2PmhFsYJR的script hash，小端序)  
- toChainId: 2 (Ethereum链的编码)  
- toAddress: 0x0B24aBDd39185055311aaa27082F9dEb294A7255 (Ethereum链地址)  
- amount: 10000 (跨链数量)