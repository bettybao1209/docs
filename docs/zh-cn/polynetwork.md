# Poly Network

Poly Network基于侧链/中继模式，采用双层结构设计，使用Poly中继链作为跨链协调器，多条异构链作为跨链交易执行器，Relayer作为跨链信息的搬运工，通过解决跨链信息的有效性、安全性和事务性等问题，实现了一套安全、易用、高效的跨链体系。

更多有关Poly Network的信息，请点击[此处](https://github.com/polynetwork/docs/blob/master/README_CN.md)查看。

# Neo链和中继链之间的跨链交易

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

# Neo跨链合约开发

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

## 跨链合约开发示例（新发行的NEP5资产）

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


## 跨链合约开发示例（已发行的NEP5资产）

由于无法给已发行的NEP5资产添加`Lock`、`Unlock`方法，因此需要额外实现一个代理合约，相当于原先NEP5合约的补充功能，实现了跨链协议中的主要接口，也可以使用现有的代理合约实现跨链，避免重复开发。

以下为代理合约必须实现的接口：

### BindProxyHash

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

### BindAssetHash

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

### Lock

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

### Unlock

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

## 如何将NEP5资产跨到其他链上

### 收集资产在链上对应的合约地址

以某个NEP5资产为例，这种资产在链上的合约地址为：

-Neo链: 4da43995ee75be3931fd3abae06a3e6447d2febf (小端序)

-Ethereum链: 0x86B7Be11D77043E5aDe3858f2F694E4f89d4a941

## 执行跨链操作

### 集成跨链功能的NEP5资产

如果该资产集成了跨链的Lock和Unlock接口，则可以直接调用该资产的Lock接口进行跨链资产转移：

```C#
public static bool Lock(byte[] fromAddress, BigInteger toChainId, byte[] toAddress, BigInteger amount)
```

**参数设置：**  

- fromAddress: 5392ebf880d9a7119b17b2f1adaebc5f71c28e75 (Neo链地址APPmjituYcgfNxjuQDy9vP73R2PmhFsYJR的script hash，小端序)  
- toChainId: 2 (Ethereum链的编码)  
- toAddress: 0x0B24aBDd39185055311aaa27082F9dEb294A7255 (Ethereum链地址)  
- amount: 10000 (跨链数量)

### 未集成跨链功能的NEP5资产

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

# 跨链资产列表

当前Neo上已有的跨链资产如下表所示：

类型 | 合约哈希 | 描述
---|---|---
ETHx | B: 0x17c76859c11bc14da5b3e9c88fa695513442c606 </br> L: 06c642345195a68fc8e9b3a54dc11bc15968c717 | Eth asset hash in Neo chain
ONTx | B: 0x271e1e4616158c7440ffd1d5ca51c0c12c792833 </br> L: 3328792cc1c051cad5d1ff40748c1516461e1e27 | ONT asset hash in Neo chain
pnWETH | B: 0x0df563008be710f3e0130208f8adc95ed7e5518d </br> L: 8d51e5d75ec9adf8080213e0f310e78b0063f50d | nWETH asset hash in Neo chain
nNEO | B: 0xf46719e2d16bf50cddcef9d4bbfece901f73cbb6 </br> L: b6cb731f90cefebbd4f9cedd0cf56bd1e21967f4 | NEP5 asset hash of NEO
pONT | B: 0xc277117879af3197fbef92c71e95800aa3b89d9a </br> L: 9a9db8a30a80951ec792effb9731af79781177c2 | ONTd asset hash in Neo chain
pnUSDT | B: 0x282e3340d5a1cd6a461d5f558d91bc1dbc02a07b </br> L: 7ba002bc1dbc918d555f1d466acda1d540332e28 | nUSDT asset hash in Neo chain
pnWBTC | B: 0x534dcac35b0dfadc7b2d716a7a73a7067c148b37 </br> L: 378b147c06a7737a6a712d7bdcfa0d5bc3ca4d53 | nWBTC asset hash in Neo chain
pnUNI_V2_ETH_WBTC | B: 0xc534d65c85c074887f58ed1f3bad7dfd739a525e </br> L: 5e529a73fd7dad3b1fed587f8874c0855cd634c5 | nUNI_V2_ETH_WBTC asset hash in Neo chain
FLM | B: 0x4d9eab13620fe3569ba3b0e56e2877739e4145e3 </br> L: e345419e7377286ee5b0a39b56e30f6213ab9e4d | Flamingo token

> Note: **`B`** 表示大端序, 可直接使用该哈希值在区块浏览器中查询合约交易历史。 **`L`** 表示小端序，通常在执行资产哈希绑定时使用。

# 路由与ChainId
类型 | 路由编码 | ChainId
:-:|:-:|:-:
Bitcoin | 1 | 1
Ethereum | 2 | 2
Ontology | 3 | 3
NEO | 4 | 4
Switcheo | 5 | 5