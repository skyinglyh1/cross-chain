<h1 align="center">How to Deploy Crosschain on Cosmos</h1>
<h4 align="center">Version 1.0 </h4>

[English](how_to_deploy_crosschain_on_cosmos.md) | 中文

本文档介绍如何在Cosmos子链中添加、设置及使用<code>crosschain</code>模块。包括如何在Cosmos子链的App中初始化<code>crosschain</code>的Keeper，及设置当前链已有资产的跨链配置或设置当前链初始时不存在的资产配置。其中跨链资产可分为源链支持虚拟机的资产如Ether, ONT等，和源链不含有虚拟机的资产如BTC, Atom。

## 在App中添加<code>crosschain</code>模块

```go
// make sure crosschain keepr has the permission to change supply
var (
    maccPerms = map[string][]string{
		auth.FeeCollectorName:     nil,
		distr.ModuleName:          nil,
		mint.ModuleName:           {supply.Minter},
		staking.BondedPoolName:    {supply.Burner, supply.Staking},
		staking.NotBondedPoolName: {supply.Burner, supply.Staking},
		gov.ModuleName:            {supply.Burner},
		crosschain.ModuleName:     {supply.Burner, supply.Minter},
	}
)
// create store key of the crosschain module 
keys := sdk.NewKVStoreKeys(
		bam.MainStoreKey, auth.StoreKey, staking.StoreKey,
		supply.StoreKey, mint.StoreKey, distr.StoreKey, slashing.StoreKey,
		gov.StoreKey, params.StoreKey,
		crosschain.StoreKey,
    )
// allocate subspace used for crosschain keeper
crosschainSubspace := app.paramsKeeper.Subspace(crosschain.DefaultParamspace)

// create the necessary account keeper
app.accountKeeper = auth.NewAccountKeeper(app.cdc, keys[auth.StoreKey], authSubspace, auth.ProtoBaseAccount)
app.bankKeeper = bank.NewBaseKeeper(app.accountKeeper, bankSubspace, bank.DefaultCodespace, app.ModuleAccountAddrs())
app.supplyKeeper = supply.NewKeeper(app.cdc, keys[supply.StoreKey], app.accountKeeper, app.bankKeeper, maccPerms)

// create the crosschain keeper
app.ccKeeper = crosschain.NewKeeper(app.cdc, keys[crosschain.StoreKey], crosschainSubspace, app.accountKeeper, app.supplyKeeper)
```

## <code>crosschain</code>模块中函数的设置
```go
type Keeper interface {
	HeaderSyncKeeper
	LockProxyKeeper
	GetModuleAccount(ctx sdk.Context) exported.ModuleAccountI
	CreateCoins(ctx sdk.Context, creator sdk.AccAddress, coins sdk.Coins) sdk.Error
	SetRedeemScript(ctx sdk.Context, denom string, redeemKey []byte, redeemScript []byte)
}

type HeaderSyncKeeper interface {
	SyncGenesisHeader(ctx sdk.Context, genesisHeader []byte) sdk.Error
	SyncBlockHeaders(ctx sdk.Context, headers [][]byte) sdk.Error
	ProcessHeader(ctx sdk.Context, header *mctype.Header) sdk.Error
	HeaderSyncViewKeeper
}

type HeaderSyncViewKeeper interface {
	GetHeaderByHeight(ctx sdk.Context, chainId uint64, height uint32) (*mctype.Header, sdk.Error)
	GetHeaderByHash(ctx sdk.Context, chainId uint64, hash mcc.Uint256) (*mctype.Header, sdk.Error)
	GetCurrentHeight(ctx sdk.Context, chainId uint64) (uint32, sdk.Error)
	GetConsensusPeers(ctx sdk.Context, chainId uint64, height uint32) (*types.ConsensusPeers, sdk.Error)
	GetKeyHeights(ctx sdk.Context, chainId uint64) *types.KeyHeights
}

type LockProxyKeeper interface {
	SetOperator(ctx sdk.Context, operator types.Operator)
	BindProxyHash(ctx sdk.Context, targetChainId uint64, targetProxyHash []byte)
	BindAssetHash(ctx sdk.Context, sourceAssetDenom string, targetChainId uint64, targetAssetHash []byte, initialAmt sdk.Int) sdk.Error
	Lock(ctx sdk.Context, fromAddress sdk.AccAddress, sourceAssetDenom string, toChainId uint64, toAddressBs []byte, value sdk.Int) sdk.Error
	ProcessCrossChainTx(ctx sdk.Context, fromChainId uint64, height uint32, proofStr string, headerBs []byte) sdk.Error
	LockProxyViewKeeper
}
type LockProxyViewKeeper interface {
	GetOperator(ctx sdk.Context) (operator types.Operator)
	GetProxyHash(ctx sdk.Context, toChainId uint64) []byte
	GetAssetHash(ctx sdk.Context, sourceAssetDenom string, toChainId uint64) []byte
	GetLockedAmt(ctx sdk.Context, sourceAssetDenom string) sdk.Int
}

```

<code>crosschain</code>模块的<code>Keeper</code>主要包换两个部分。
- <code>HeaderSyncKeeper</code>主要完成中继链的区块头信息同步到Cosmos子链的工作；
- <code>LockProxyKeeper</code>部分主要完成
  - 目标链代理合约与当前链的<code>crosschain</code>模块的映射(<code>BindProxyHash</code>)，
  - 当前链的新资产发行(<code>CreateCoins</code>)，
  - 目标链资产哈希与当前链新增资产的映射(<code>BindAssetHash</code>)，
  - 处理跨链请求(<code>Lock</code>)，
  - 处理中继链(<code>ProcessCrossChainTx</code>)跨链到当前Cosmos子链的交易。

<code>crosschain</code>模块的正常运行需要<code>operator</code>对参数设置进行管理，如资产的映射、代理合约的映射、新资产的发行来确保参数的正确性与安全性。

### 有虚拟机的源链资产跨链设置

以当前Cosmos子链与以太坊链之间的Ether资产跨链为例，
- 首先需要<code>operator</code>在Cosmos子链上通过<code>CreateCoins</code>发行Ether通证，这些通证在被创建后全部在模块帐户中。
- 然后<code>opeator</code>需要调用<code>BindProxyHash</code>完成当前Cosmos子链的<code>crosschain</code>模块与以太坊链上可信代理合约的映射，
- 然后</code>operator</code>需要调用<code>BindAssetHash</code>完成当前Cosmos子链的Ether通证与以太坊链上的Ether资产进行映射。

在以太坊端的相关合约中进行相似的操作映射后，即可通过调用以太坊端的代理合约的跨链请求接口发送跨链请求到当前Cosmos子链，Cosmos Relayer在接收到中继广播的交易证明后，将调用<code>crosschain</code>模块的<code>ProcessCrossChainTx</code>函数，在验证以太坊上确实锁定了一定量的Ether资产后，<code>crosschain</code>将会从自身模块帐户中解锁对应量的Ether通证到对应的Cosmos子链的用户帐户地址中。

### 无虚拟机的源链资产跨链设置
以当前Cosmos子链与比特币网络之间的BTC资产跨链为例，
- 首先需要<code>operator</code>在Cosmos子链上通过<code>CreateCoins</code>发行BTC通证，这些通证在被创建后全部在模块帐户中，
- <code>operator</code>在Cosmos子链上通过调用<code>SetRedeemScript</code>将多签的锁定脚本的[哈希](https://github.com/btcsuite/btcutil/blob/v1.0.2/hash160.go#L21)与锁定脚本进行映射，
- 然后</code>operator</code>需要调用<code>BindAssetHash</code>完成当前Cosmos子链的BTC通证与比特币链上的该锁定脚本对就多签地址的BTC资产进行映射。


## 发送跨链交易

以Atom为例，
- Cosmos子链端：因为当前Cosmos子链已有Atom资产，所以<code>operator</code>不用调用<code>CreateCoins</code>来创建新的通证，但<code>operator</code>仍需要调用<code>BindProxyHash</code>、<code>BindAssetHash</code>来进行代理和资产的映射。
- 目标链端：需要具有AtomX资产合约及其信任的代理合约<code>LockProxy</code>，如不是很清楚该如何在目标链上进行设置，请移步至[跨链之以太坊文档](./../eth/how_to_deploy_crosschain_on_ethereum_CN.md)或[跨链之本体链文档](./../ont/How_to_new_cross_chain_asset_cn.md)。

### 从当前Cosmos子链到其他目标链的跨链交易
用户构造<code>MsgLock</code>结构体，并放到<cod>auth</code>模块中的<code>StdSignMsg</code>的<code>Msgs</code>进行签名并发送交易，即完成跨链请求
```go
type MsgLock struct {
	FromAddress      sdk.AccAddress // Cosmos子链中请求跨链的发送方
	SourceAssetDenom string         // 请求进行跨链的资产在Cosmos子链上的名称
	ToChainId        uint64         // 目标链的ChainId
	ToAddressBs      []byte         // 目标链上接收地址的字节数组
	Value            *sdk.Int       // 请求进行跨链资产的数量
}
```
具体如下，其中SendMsg可参考[这里](https://github.com/skyinglyh1/gaia/blob/master/x/test/crosschain_test.go#L290)。
```go
import (
    rpchttp "github.com/tendermint/tendermint/rpc/client"
)
// new http client
client := rpchttp.NewHTTP("tcp://cosmoms_ip:cosmos_port", "/websocket")
// register cdc
appCdc := app.MakeCodec()
// obtain your cosmos account from wallet and password
fromPriv, fromAddr, err := GetCosmosPrivateKey(user0Wallet, []byte(operatorPwd))
// parse ont address from base58 format to byte array
toOntAddr, _ := common.AddressFromBase58("AQf4Mzu1YJrhz9f3aRkkwSm9n3qhXGSh4p")
toOntAmount = sdk.NewInt(100)
// construct the MsgLock of crosschain module
msg := crosschain.NewMsgLock(fromCosmosAddr, "umuon", 3, toOntAddr, toOntAmount)
// broadcast the msg to the network
if err := sendMsg(client, fromAddr, fromPriv, appCdc, msg); err != nil {
    fmt.Errorf("sendMsg error:%v", err)
}

```

### 从目标链到当前Cosmos子链的跨链交易
从目标链发送交易的细节可以参考本体链上的发送跨链交易的请求和以太坊链上的发送跨链交易请求。该目标链上的跨链交易在目标链上执行成功后，将由目标链上的Relayer将跨链信息同步到中继链，由中继链进行验证、生成证明并全网广播。Cosmos子链的Relayer监听到中继链上的发向该Cosmos子链的交易证明后，将跨链信息通过<code>crosschain</code>模块里的<code>ProcessCrosschainTx</code>函数提交到Cosmos子链，由<code>ProcessCrossChainTx</code>来验证Relayer提交的交易信息的有效性，从而将锁定的资产从<code>crosschain</code>模块帐户转出到接收地址。


## 许可证

Ontology遵守GNU Lesser General Public License, 版本3.0。 详细信息请查看项目根目录下的LICENSE文件。
