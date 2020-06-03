<h1 align="center">How to Do Crosschain on Cosmos</h1>
<h4 align="center">Version 1.0 </h4>

English | [中文](how_to_cross_on_cosmos_CN.md)

This article mainly introduces how to import, set up and employ the provided modules in your cosmos application in order to do cross chain through [poly chain](https://github.com/polynetworks).

## How to add these cross chain modules for developers intending to integrate the cross chain functionality?
Add these created module keepers to the App in <code>app.go</code>.
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
		btcx.ModuleName:           {supply.Burner, supply.Minter},
		ft.ModuleName:             {supply.Burner, supply.Minter},
		lockproxy.ModuleName:      {supply.Minter},
	}
)
// create store key of the correlated module 
keys := sdk.NewKVStoreKeys(
    bam.MainStoreKey, auth.StoreKey, staking.StoreKey,
    supply.StoreKey, mint.StoreKey, distr.StoreKey, slashing.StoreKey,
    gov.StoreKey, params.StoreKey,
    headersync.StoreKey, ccm.StoreKey,
    btcx.StoreKey, ft.StoreKey, lockproxy.StoreKey,
)
// allocate subspace used for crosschain keeper
headerSyncSubspace := app.paramsKeeper.Subspace(headersync.DefaultParamspace) 
ccmSubspace := app.paramsKeeper.Subspace(ccm.DefaultParamspace)               
btcxSubspace := app.paramsKeeper.Subspace(btcx.DefaultParamspace)             
ftSubspace := app.paramsKeeper.Subspace(ft.DefaultParamspace)                 
lockproxySubspace := app.paramsKeeper.Subspace(lockproxy.DefaultParamspace)  

// create the necessary account keeper
app.accountKeeper = auth.NewAccountKeeper(app.cdc, keys[auth.StoreKey], authSubspace, auth.ProtoBaseAccount)
app.bankKeeper = bank.NewBaseKeeper(app.accountKeeper, bankSubspace, bank.DefaultCodespace, app.ModuleAccountAddrs())
app.supplyKeeper = supply.NewKeeper(app.cdc, keys[supply.StoreKey], app.accountKeeper, app.bankKeeper, maccPerms)

// create the crosschain keeper
// for syncing poly chain vital block header into this cosmos chain
app.headersyncKeeper = headersync.NewKeeper(app.cdc, keys[headersync.StoreKey], headerSyncSubspace)
// for processing crosschain msg from both cosmos chain and poly chain
app.ccmKeeper = ccm.NewKeeper(app.cdc, keys[ccm.StoreKey], ccmSubspace, app.headersyncKeeper, nil)
 // for some organizations to be able to provide proxy service for some assets
app.lockproxyKeeper = lockproxy.NewKeeper(app.cdc, keys[lockproxy.StoreKey], lockproxySubspace, app.accountKeeper, app.supplyKeeper, app.ccmKeeper)
// for non-virtual machine blockchain asset to be able to be crossed into current cosmos chain
app.btcxKeeper = btcx.NewKeeper(app.cdc, keys[btcx.StoreKey], btcxSubspace, app.accountKeeper, app.bankKeeper, app.supplyKeeper, app.ccmKeeper)
// for creating fungible token to be crossed into/from another chain 
app.ftKeeper = ft.NewKeeper(app.cdc, keys[ft.StoreKey], ftSubspace, app.accountKeeper, app.bankKeeper, app.supplyKeeper, app.ccmKeeper)
//  mount btcxKeeper, ftKeeper and lockproxyKeeper into ccm (cross chain manager) module 
app.ccmKeeper.MountUnlockKeeperMap(map[string]ccm.UnlockKeeper{
    btcx.StoreKey:      app.btcxKeeper,
    ft.StoreKey:        app.ftKeeper,
    lockproxy.StoreKey: app.lockproxyKeeper,
})
```
## What method, or interface or message can be used in each module?

### Message in <code>headersync</code> module
<code>MsgSyncGenesisParam</code> is for syncing genesis blockheader of <code>poly chain</code> or the blockheader containing <code>. The key header of poly chain means these exists valid <code>[NewChainConfig](https://github.com/skyinglyh1/gaia/blob/master/x/headersync/internal/keeper/keeper.go#L182)</code>.

Once the genesis header has been synced, the cosmos-relayer can be started to synchronize the key block headers, which means the poly chain block header contains valid <code>[CrossStateRoot](https://github.com/skyinglyh1/gaia/blob/master/x/headersync/poly-utils/core/types/header.go#L37)</code> that will be used to verify [proof](https://github.com/skyinglyh1/gaia/blob/master/x/ccm/internal/keeper/keeper.go#L125) submitted by cosmos-relayer and obtain the cross chain transaction message or [value](https://github.com/skyinglyh1/gaia/blob/master/x/ccm/internal/keeper/keeper.go#L174) 
```go
// Msg for syncing poly chain genesis header into current cosmos chain
type MsgSyncGenesisParam struct {
	Syncer        sdk.AccAddress // submitter of poly chain header
	GenesisHeader []byte         // header bytes of poly genesis header, encoding role is the same as poly chain Header.Serialization() method
}
// Msg for syncing poly chain header into current cosmos chain
type MsgSyncHeadersParam struct {
	Syncer  sdk.AccAddress      // submitter address
	Headers [][]byte            // multiple headers of poly chain block header
}

```
### Message in <code>ccm</code> module
Here, <code>ccm</code> means "cross chain manager" or "cross chain management". It has two functionality.
- 1. It provides interface for cosmos-relayer to submit cross chain transaction coming from poly chain. While cosmos-relayer monitoring the poly chain, once it detacts poly chain native event aiming for current cosmos chain, it will get the proof from ledger store of poly chain and construct <code>MsgProcessCrossChainTx</code> to request for the completion of cross chain transaction in cosmos chain side. Within <code>ccm</code> module, the submitted header (as we introduced before as key header) will be verified first. The <code>CrossStateRoot</code> within verified poly chain header will be used to verify the proof. Then the actual cross chain message or transaction can be obtained. The <code>ccm</code> module then checks which module should be responsible for processing the cross chain message and execute the "unlock" logic. Currently, there are three modules can process "unlock" logic, including <code>lockproxy</code>, <code>btcx</code> and <code>ft</code>, which will be introduced later.
- 2. It provides interface <code>CreateCrossChainTx</code> for <code>lockproxy</code>, <code>btcx</code> and <code>ft</code> modules to create cross chain transaction from current cosmos chain to another blockchain, say, Ethereum, BTCX, ONTology, or NEO. Within this interface, there will be a unique key and immutable value consisted of current cross chain transaction stored in the ledger to ensure the cross chain transaction existing in cosmos (from) side are completed successfully.

Now we can easily see, <code>MsgProcessCrossChainTx</code> is for cosmos-relayer to relay cross chain tx to cosmos chain. And <code>CreateCrossChainTx</code> function is for other modules to construct cross chain tx from cosmos chain. 

```go
// Msg for processing cross chain msg from poly chain to current cosmos chain
// this message is designed for the cosmos-relayer to invoke
type MsgProcessCrossChainTx struct {
    Submitter   sdk.AccAddress    // cosmos-relayer account address 
	FromChainId uint64            // which chain id this cross chain msg comes from
	Height      uint32            // which height the proof is generated at poly chain 
	Proof       string            // the proof of cross chain msg coming from poly chain for current cosmos chain to obtain the actual cross chain msg
	Header      []byte            // the block header of poly chain containing the CrossStateRoot to verify the provided proof
}
```

### Message in <code>btcx</code> module
```go
// Msg for some organizations or service provider to create btc asset contract for cross chain usage.
// `Createor` creates a kind of coin with name of `Denom` and redeem script of `RedeemScript`
// The initial supply if zero, make sure the coin with name of `Denom` has never been created before, otherwise, tx will fail
type MsgCreateCoin struct {
	Creator      sdk.AccAddress     // the btc asset contract creator
	Denom        string             // the name of the btc asset contract
    RedeemScript string             // the redeem script comes from the btc blockchain redeemscript of multiple accounts in order to 
                                    // process the cross chain transaction back to btc and ideneity the unique btc asset contract in cosmos chain 
}

// this interface is designed for the denom creator to bind asset of `sourceAssetDenom` with another asset contract with hash of `ToAssetHash` in another blockchain with chain id `ToChainId`
// only the `SourceAssetDenom` creator can make the transaction successfully
type MsgBindAssetHash struct {
	Creator          sdk.AccAddress // this parameter should be the same as `Creator` in `MsgCreateCoin` struct, indicates that only the creator can make bind asset hash transaction successfully
	SourceAssetDenom string         // this parameter indicates which denom in current cosmos chain will be bonded 
	ToChainId        uint64         // this parameter indicates the `SourceAssetDenom` will be bonded with another asset hash in which blockchain 
	ToAssetHash      []byte         // this parameter indicates the `SourceAssetDenom` will be bonded with which asset hash in `ToChainId` blockchain
}

// this interface is designed for the user to do "lock" operation, meaning user (`FromAddress`) will lock `Value` amount of `SourceAssetDenom` coins into btcx module, 
// so that the address `ToAddressBs` in another blockchain with chainId (`ToChainId`) can receive the same amount of the same asset (yet in different chain).
// before user invokes this method, he/she should make sure the service provider/organization is reliable in terms of their reputation, 
// correctness of paramters (say, `ToChainId` and the related `ToAssetHash` in previous message)
type MsgLock struct {
	FromAddress      sdk.AccAddress  // the user (locker/sender) who wants to do cross chain transaction from current cosmos chain to another blockchain
	SourceAssetDenom string          // the kind of denom (coin) the user wants to send `Value` amount of `SourceAssetDenom` to "btcx" module
	ToChainId        uint64          // into which chainId, the user wants to do cross chain transaction form current cosmos chain
	ToAddressBs      []byte          // in `ToChainId`, the receiver address is `ToAddressBs` in bytes format
    Value            *sdk.Int        // the amount of `SourceAssetDenom` the user intends to lock 
                                     //   (cross from current cosmos chain to another chain with `ToChainId`)
}
```

### Message in <code>ft</code> module
Here, <code>ft</code> means the <code>fungible token</code>, like the asset follows [OEP4 protocol](https://github.com/ontio/OEPs/blob/master/OEPS/OEP-4.mediawiki) or [ERC20 protocol](https://eips.ethereum.org/EIPS/eip-20), yet it just creates coins and sends the coins to the designated account. As for the rich functionalities like "balanceOf", "approve", "allowance", "transferFrom", they are not implemented considering the built-in module <code>bank</code> in cosmos-sdk.

Besides, this module provides other message types or interfaces for the cross chain lockproxy service providers, traditional users or dev team behind the Dapp. 

As for the cross chain lockproxy service provider, there are two approaches to employ the <code>ft</cod> module.
- 1. Cross chain service providers can construct, sign and send the message of <code>MsgCreateAndDelegateCoinToProxy</code> to create coins and delegate all the coins to the reliable <code>lockproxy</code> (also can be what we created before). The steps or flow to use <code>MsgCreateAndDelegateCoinToProxy</code> and how to integrate with <code>lockproxy</code> module are as follows.
  - step 1. The coin creator can ask the reliable <code>lockproxy</code> creator (also can be himself) to help  do <code>BindAssetHash</code> operation, before which, the coin creator should make sure the reliable operation of <code>BindProxy</code> has already been done within the reliable <code>lockproxy</code> contract. Here, by "reliable", we mean any <code>lockproxy</code> contract, in any other blockchains connected to polychain and related to this <code>lockproxy</code> contract, should be verified and operated by the reliable operator.
  - step 2. Coin creator asks the reliable <code>lockproxy</code> operator to do <code>BindAssetHash</code> operation to map current asset/coin to other asset hash deployed in other blockchain.
  - step 3. After the setup done correctly, any user can receive correlated asset or coin sending from <code>lockproxy</coce> when the cross chain request coming from <code>poly chain</code> are processed completely. Also the user can use <code>bank</code> module of cosmos-sdk to send coins to other account within cosmos chain. Meanwhile, any user holding this type of coin, can construct <code>MsgLock</code> message of <code>lockproxy</code>, sign, and broadcast the message in order to request for cross chain transaction to other blockchain, like Ethereum, Ontology or Neo.

- 2. Cross chain service providers can also use <code>MsgCreateDenom</code> to create coins with zero initial supply, so that they can realize the cross chain functionalities without the help of <code>lockproxy</code> (or without trusting any other <code>lockproxy</code> contract).
  - step 1. The coin creator can bind current denom or asset hash with the same asset contract hash in another blockchain by using <code>MsgBindAssetHash</code> provided within <code>ft</code> module. Normally speaking, in another blockchain, the same asset contract hash is not mapped to current denom or asset hash through <code>lockproxy</code> contract.
  - step 2. After the setup done correctly, any user can receive correlated asset or coin minted after the cross chain request coming from <code>poly chain</code> are processed completely. Also the user can use <code>bank</code> module of cosmos-sdk to send coins to other account within cosmos chain. Meanwhile, any user holding this type of coin, can construct <code>MsgLock</code> message of <code>ft</code>, sign, and broadcast the message in order to request for cross chain transaction to other blockchain, like Ethereum, Ontology or Neo.

In addition, any team intending to issue new coin/token can create coins with non-zero supply with the help of <code>MsgCreateCoins</code>, the initial coins are in the account of creator. 

```go
// this message is for some service provider/organizations to create some fungible token with non-zero initial supply 
// and delegate all the coins to `lockproxy` module, which will be introduced later. This coin is prepared for the target chain asset in cosmos chain.
// Actually, there is no much difference speaking of who is the coin creator since the critical part is that the `lockproxy` operator should do "BindProxyHash" and "BindAssetHash" correctly.
// for some service provider who wishes to do cross chain transaction through the `lockproxy` module below, you should invoke this transaction to create your token/coin.
type MsgCreateAndDelegateCoinToProxy struct {
	Creator sdk.AccAddress          // coin creator
	Coin    sdk.Coin                // coin should have the format of "100000MySimpleToken" with supply of 100000 and name of "MySimpleToken"
}

// this message is for some service provider/organizations to create some fungible token with zero initial supply and is designed to 
// perform cross chain transaction within `ft` module with the following two message: `MsgBindAssetHash` and `MsgLock`. 
type MsgCreateDenom struct {
	Creator sdk.AccAddress          // `Denom` creator 
	Denom   string                  // coin name being created with initial zero supply
}

// this interface is designed for the denom creator to bind asset of `sourceAssetDenom` with another asset contract with hash of `ToAssetHash` in another blockchain with chain id `ToChainId`
// only the `SourceAssetDenom` creator can make the transaction successfully
type MsgBindAssetHash struct {
	Creator          sdk.AccAddress // this parameter should be the same as `Creator` in the above `MsgCreateDenom` struct, indicates that only the creator can make bind asset hash transaction successfully
	SourceAssetDenom string         // this parameter indicates which denom in current cosmos chain will be bonded 
	ToChainId        uint64         // this parameter indicates the `SourceAssetDenom` will be bonded with another asset hash in which blockchain 
	ToAssetHash      []byte         // this parameter indicates the `SourceAssetDenom` will be bonded with which asset hash in `ToChainId` blockchain
}

// this interface is designed for the user to do "lock" operation, meaning user (`FromAddress`) will burn `Value` amount of `SourceAssetDenom` coins to air, 
// so that the address `ToAddressBs` in another blockchain with chainId (`ToChainId`) can receive the same amount of the same asset (yet in different chain).
// before user invokes this method, he/she should make sure the service provider/organization is reliable in terms of their reputation, 
// correctness of paramters (say, `ToChainId` and the related `ToAssetHash` in previous message)
type MsgLock struct {
	FromAddress      sdk.AccAddress  // the user (locker/msg sender) who wants to do cross chain transaction from current cosmos chain to another blockchain
	SourceAssetDenom string          // the kind of denom (coin) the user wants to burn `Value` amount of `SourceAssetDenom`
	ToChainId        uint64          // into which chainId, the user wants to do cross chain transaction form current cosmos chain
	ToAddressBs      []byte          // in `ToChainId`, the receiver address is `ToAddressBs` in bytes format
    Value            *sdk.Int        // the amount of `SourceAssetDenom` the user intends to do cross chain
                                     //   (from current cosmos chain to another chain with `ToChainId`)
}

// this interface is designed for everyone to create multiple coins, which are available for both `bank` and `supply` modules in cosmos-sdk. The coins should have valid format 
// in terms of coin.Denom and coin.Amount. All the initialized coins are distributed to the `Creator`.
type MsgCreateCoins struct {
	Creator sdk.AccAddress           // coins' creator
	Coins   string                   // Coins should be the format of "100000MySimpleToken,2000000MST"
}

```

### <code>lockproxy</code>模块中函数或消息

This module contains common interface for cross chain request through <code>lockproxy</code> module. There are two types of invokers separated by the roles.

As the service provider or lockproxy provider, the instruction to use <code>lockproxy</code> module are the followings.
  1. First, the lockproxy creator invoke <code>MsgCreateLockProxy</code> to create the unique `MsgCreateLockProxy` lockproxy contract. The `lockproxy` contract is identified by the creator account address and managed by the creator, also knowns as operator.
  2. The operator invokes `MsgBindProxyHash` to bind current <code>lockproxy</code> contract with another lockproxy contract deployed in another blockchain. Note that the operator should do bind proxy hash in other sides of blockchains.
  3. The service provider or operator sets up the map from current asset hash (coin name or denom) to the same asset hash deployed in another blockchain. Note that the operator should do bind asset hash in other sides of blockchains.

As the users of <code>lockproxy</code>, we can invoke <code>MsgLock</code> directly to send cross chain request. The precondition is that cross chain requester should make sure `MsgLock.FromAddress` have enough balance of coins.

Next we explain each part of the struct in detail.

```go
// this message is for the cross chain service provider to invoke in order to create a new lockproxy contract.
// the lockproxy contract address is identified by the creator's account address. One account can only create one lockproxy
type MsgCreateLockProxy struct {
	Creator sdk.AccAddress
}

// this message is for lockproxy creator to bind current lockproxy contract with another lockproxy contract in another chain.
type MsgBindProxyHash struct {
	Operator         sdk.AccAddress     // `Operator` should be the lockproxy creator
	ToChainId        uint64             // `ToChainId` indicates another chain the current lockproxy is able to reach
    ToChainProxyHash []byte             // `ToChainProxyHash` indicates the lockproxy contract hash in another chain that
                                        //  can reach the current lockproxy contract.
}

// this message is for lockproxy creator to bind current asset denom `SourceAssetDenom` with the same asset 
// with hash of `ToAssetHash` yet in another chain with `ToChainId`. The initial amount prepared for cross chain transaction 
// is `InitialAmt`, meaning `InitialAmt` of `SourceAssetDenom` should already in `lockproxy` module account.
// Now we can see, this message should be invoked after `MsgCreateAndDelegateCoinToProxy` of `ft` module is invoked.
type MsgBindAssetHash struct {
	Operator         sdk.AccAddress  // `Operator` should be the lockproxy creator
	SourceAssetDenom string          // `SourceAssetDenom` is the coin kind correlated with the asset of `ToAssetHash` in `ToChainId` blockchain
	ToChainId        uint64          // another chain with `ToChainId` that holds the `ToAssetHash`, and are supported by the poly chain
	ToAssetHash      []byte          // the asset hash existing in `ToChainId` chain 
	InitialAmt       sdk.Int         // the initial amount of `SourceAssetDenom` that the `lockproxy` module account holds
}



// this interface is designed for the user to do "lock" operation, meaning user (`FromAddress`) will burn `Value` amount of `SourceAssetDenom` coins to air, 
// so that the address `ToAddressBs` in another blockchain with chainId (`ToChainId`) can receive the same amount of the same asset (yet in different chain).
// before user invokes this method, he/she should make sure the service provider/organization is reliable in terms of their reputation, 
// correctness of paramters (say, `ToChainId` and the related `ToAssetHash` in previous message)
type MsgLock struct {
    LockProxyHash    []byte          // current lockproxy the FromAddress wants to use to do cross chain transaction
	FromAddress      sdk.AccAddress  // the user (locker/msg sender) who wants to do cross chain transaction from current cosmos chain to another blockchain
	SourceAssetDenom string          // the kind of denom (coin) the user wants to burn `Value` amount of `SourceAssetDenom`
	ToChainId        uint64          // into which chainId, the user wants to do cross chain transaction form current cosmos chain
	ToAddressBs      []byte          // in `ToChainId`, the receiver address is `ToAddressBs` in bytes format
    Value            *sdk.Int        // the amount of `SourceAssetDenom` the user intends to do cross chain
                                     //   (from current cosmos chain to another chain with `ToChainId`)
}


```





## 许可证

The Ontology library is licensed under the GNU Lesser General Public License v3.0. Please refer to the LICENSE file in the root directory of the project for details.
