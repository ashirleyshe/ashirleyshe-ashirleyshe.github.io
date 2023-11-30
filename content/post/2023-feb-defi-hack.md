---
title: "2023 Feb Defi Hack Recap & Analysis"
date: 2023-03-08
draft: false
subtitle:    ""
description: ""
author:      "Ashley Hsu"
image:       "https://i.imgur.com/JbOqEij.jpg"
tags:        ["Defi"]
categories:  ["Blockchain"]
---

## Overview
2023 年 2 月，攻擊者竊取了約 3550 萬美元的加密貨幣。

Hopeless...

### Defi 安全事件
- 2 月 3 日，bonqdao 遭受價格操控攻擊，損失約 1.2 億美元，但攻擊者只拿走不到 200 萬美元。
- 2 月 3 日，USDs 遭受攻擊，損失約 30 萬美元。
- 2 月 4 日，orion protocol 遭受閃電貸重入攻擊，損失約 300 萬美元。
- 2 月 10 日，dForce 遭受攻擊，又是 read-only reentrancy，損失約 370 萬美元。
- 2 月 17 日，Avalanche 鏈上的 Platypus 項目遭受閃電貸攻擊，損失約 850 萬美元。
- 2 月 17 日，Dexible 項目因合約中函數邏輯漏洞遭受攻擊，損失約 154 萬美元。
- 2 月 24 日，Shata Capital 合約在升級後遭受攻擊，損失約 510 萬美元。
- 2 月 27 日，SwapX 合約因關鍵函數缺乏訪問控制而遭到攻擊，此合約並未開源，多個授權給該合約的代幣遭受了損失，涉及金額約 90 萬美元。

### 其他安全事件
- 2 月 21 日，Arbitrum 鏈上 Hope Finance 發生 rug pull，提取了 180 萬美元並轉入 Tornado Cash。
- 2 月 25 日，Wormhole 遭駭的 12 萬顆 eth 被 Jump Crypto 及 Oasis 回收。

## Platypus
> 攻擊者存入 4400 萬抵押品，借入 4200 萬，然後使用 emergencyWithdraw()，它很高興地向攻擊者返還了全部存入的抵押品——此時攻擊者擁有的是 4400 萬抵押品加上借入的 4200 萬。

- 攻擊交易：
  - https://snowtrace.io/tx/0x1266a937c2ccd970e5d7929021eed3ec593a95c68a99b4920c2efa226679b430 
- 攻擊者地址：
  - https://snowtrace.io/address/0xeff003d64046a6f521ba31f39405cb720e953958

### 漏洞關鍵點：emergencyWithdraw()
```solidity
/// @notice Withdraw without caring about rewards. EMERGENCY ONLY.
/// @param _pid the pool id
function emergencyWithdraw(uint256 _pid) public nonReentrant {
    PoolInfo storage pool = poolInfo[_pid];
    UserInfo storage user = userInfo[_pid][msg.sender];

    if (address(platypusTreasure) != address(0x00)) {
        (bool isSolvent, ) = platypusTreasure.isSolvent(msg.sender, address(poolInfo[_pid].lpToken), true);
        require(isSolvent, 'remaining amount exceeds collateral factor');
    }

    // reset rewarder before we update lpSupply and sumOfFactors
    IBoostedMultiRewarder rewarder = pool.rewarder;
    if (address(rewarder) != address(0)) {
        rewarder.onPtpReward(msg.sender, user.amount, 0, user.factor, 0);
    }

    // SafeERC20 is not needed as Asset will revert if transfer fails
    pool.lpToken.transfer(address(msg.sender), user.amount);

    // update non-dialuting factor
    pool.sumOfFactors -= user.factor;

    user.amount = 0;
    user.factor = 0;
    user.rewardDebt = 0;

    emit EmergencyWithdraw(msg.sender, _pid, user.amount);
}
```

`isSolvent()` 主要檢查的是抵押品的價值是否大於借款的價值。
`emergencyWithdraw()` 並沒有除了 `isSolvent()` 的其他檢查。


### 利用步驟
1. 攻擊者閃電貸貸出 44M USDC。
2. 將 USDC 存入 Platypus 的池子中並拿到 LP token。
3. 將 LP token 透過 MasterPlatypusV4 進行 deposit()。
4. 透過 PlatypusTreasure 進行借款 borrow()，借出 USP。
5. 調用 emergencyWithdraw() 領出所有抵押品。因為借貸的額度沒有超過最高限制， isSolvent 函數返回為 true，攻擊者便可以提取抵押品。
6. 將 USP 換成其他穩定幣。
7. 償還閃電貸獲利。

### 後續
- 資金依然留存在攻擊合約中，因為攻擊者沒寫 `withdraw()`
  - https://snowtrace.io/tokenholdings?a=0x67afdd6489d40a01dae65f709367e1b1d18a5322
- Tether 將攻擊者地址列入黑名單

{{< tweet user="zachxbt" id="1626335278972510208">}}

- ZachXBT 追溯到攻擊者的推特帳戶

{{< tweet user="zachxbt" id="1626434265260118021">}}

- 2/18 Blocksec 取回被盜資金

{{< tweet user="BlockSecTeam" id="1626626794966368258">}}

- 2/26 法国警方已逮捕涉嫌攻击 Platypus Finance 的两名黑客
  - https://foresightnews.pro/news/detail/18357

### Blocksec Reverse Hack
- 交易：
  - https://snowtrace.io/tx/0x5e3eb070c772631d599367521b886793e13cf0bc150bd588357c589395d2d5c3

{{< tweet user="danielvf" id="1626641254531448833">}}


![](https://pbs.twimg.com/media/FpL9YFNXsAAm6oh?format=jpg&name=4096x4096)

Blocksec 直接復用攻擊合約的 `executeOperation()`，達成一次精彩的資金救援。

攻擊合約的 `executeOperation()` 並未檢查是否為 AAVE flashloan 的 callback，導致任何人都可以調用此函數。

項目方升級合約，升級成攻擊期間竊取攻擊者的資金給項目方。順序涉及 approve USDC 給 Platypus 項目方合約，deposit USDC 到 Platypus pool 中，但攻擊合約不會拿到 LP token。
![](https://i.imgur.com/I6HM5cY.png)

### Reference
- https://twitter.com/danielvf/status/1626340324103663617
- https://twitter.com/danielvf/status/1626641254531448833

## Dexible

> selfSwap 函數存在調用 fill 函數的邏輯缺陷，調用攻擊者的自定義數據，並無檢查此自定義數據。攻擊者在數據中構造 transferfrom 函數，傳入其他用戶和自己的攻擊地址，允許合約認可的代幣轉出。

- 攻擊交易：
  - https://etherscan.io/tx/0x138daa4cbeaa3db42eefcec26e234fc2c89a4aa17d6b1870fc460b2856fd11a6 
- 攻擊者地址：
  - https://etherscan.io/address/0x684083f312ac50f538cc4b634d85a2feafaab77a

### 漏洞關鍵點：selfSwap() & fill()

`selfSwap()` 允許用戶自己定義交易路徑，並且可以指定交易的代幣、DEX，也就是說要指定調用什麼 DEX 以及向什麼 DEX 發送什麼 data 來進行交易。
並沒有檢查地址是不是真的是一個 DEX，也沒有一個白名單。

```solidity
function selfSwap(SwapTypes.SelfSwap calldata request) external notPaused {
    //we create a swap request that has no affiliate attached and thus no
    //automatic discount.
    SwapTypes.SwapRequest memory swapReq = SwapTypes.SwapRequest({
        executionRequest: ExecutionTypes.ExecutionRequest({
            fee: ExecutionTypes.FeeDetails({
                feeToken: request.feeToken,
                affiliate: address(0),
                affiliatePortion: 0
            }),
            requester: msg.sender
        }),
        tokenIn: request.tokenIn,
        tokenOut: request.tokenOut,
        routes: request.routes
    });
    SwapMeta memory details = SwapMeta({
        feeIsInput: false,
        isSelfSwap: true,
        startGas: 0,
        preSwapVault: address(DexibleStorage.load().communityVault),
        bpsAmount: 0,
        gasAmount: 0,
        nativeGasAmount: 0,
        toProtocol: 0,
        toRevshare: 0,
        outToTrader: 0,
        preDXBLBalance: 0,
        outAmount: 0,
        inputAmountDue: 0
    });
    details = this.fill(swapReq, details);
    postFill(swapReq, details, true);
}
```

```solidity
function fill(SwapTypes.SwapRequest calldata request, SwapMeta memory meta) external onlySelf returns (SwapMeta memory)  {

    preCheck(request, meta);
    meta.outAmount = request.tokenOut.token.balanceOf(address(this));
    
    for(uint i=0;i<request.routes.length;++i) {
        SwapTypes.RouterRequest calldata rr = request.routes[i];
        IERC20(rr.routeAmount.token).safeApprove(rr.spender, rr.routeAmount.amount);
        (bool s, ) = rr.router.call(rr.routerData);

        if(!s) {
            revert("Failed to swap");
        }
    }
    ...
    return meta;
}
```

`rr.routerData` 可自行構造，攻擊者在此構造了一個 `transferFrom` 函數，傳入其他用戶和自己的攻擊地址，允許合約認可的代幣轉出。
```json
{
    routerData:"0x23b872dd00000000000000000000000058f5f0684c381fcfc203d77b2bba468ebb29b098000000000000000000000000684083f312ac50f538cc4b634d85a2feafaab77a00000000000000000000000000000000000000000000000000066189a9f3b980"
}
```

`0x23b872dd` 即為 `transferFrom(address,address,uint256)` 的 function selector。

### 利用步驟
1. 先找一個有 approve token 給 Dexible 合約的受害者。
2. 調用 `selfSwap()`，並構造指定 calldata。將受害者的代幣轉移到攻擊者的地址。

### 後續
- 攻擊者將資金轉移到 Tornado Cash
- 項目方已暫停所有合約功能

### Reference
- https://rekt.news/zh/dexible-rekt/


## Shata Capital

> EFVault 合約升級後關鍵變數未正確配置，導致可以領出比實際存入的代幣更多的代幣。

- 攻擊交易：
  - https://etherscan.io/tx/0x1fe5a53405d00ce2f3e15b214c7486c69cbc5bf165cf9596e86f797f62e81914
- 攻擊者地址：
  - https://etherscan.io/address/0xa0959536560776ef8627da14c6e8c91e2c743a0a

### 漏洞關鍵點：EFVault 合約升級，storage collision

在新版本中新增了幾個變數，未考慮舊版本 impl 的 storage。讀取 `assetDecimal` 時其實是讀到舊版本的 `maxDeposit`，他們在同一個 storage slot。

新版本 EFVault 的 impl 中的 `initialize()` 不能被調用，因為 proxy 已經初始化過不能再次初始化，使得新增的變數不能進行初始化。

{{< tweet user="punk3155" id="1630753890038861825">}}

`setMaxDeposit` 是用來設定 `maxDeposit`，新的值被設為了 5000000000000，表示 assetDecimal=5000000000000，遠大於預期的值。

升級的合約新增了 `redeem()` 函數。讀取到的 storage 導致用戶可以獲得比他們實際應該獲得的更多資金。
```solidity
function redeem(uint256 shares, address receiver)
    public
    virtual
    nonReentrant
    unPaused
    onlyAllowed
    returns (uint256 assets)
{
    require(shares > 0, "ZERO_SHARES");
    require(shares <= balanceOf(msg.sender), "EXCEED_TOTAL_BALANCE");

    assets = (shares * assetsPerShare()) / 1e24;  // HERE!!!

    require(assets <= maxWithdraw, "EXCEED_ONE_TIME_MAX_WITHDRAW");

    // Withdraw asset
    _withdraw(assets, shares, receiver);
}
```
```solidity
function assetsPerShare() internal view returns (uint256) {
    return (IController(controller).totalAssets(false) * assetDecimal * 1e18) / totalSupply();
}
```

### 利用步驟
1. 攻擊者在事件發生前 27 天有調用 deposit()向 EFVault 存入 0.1 ETH，獲得一定數量的 shares。
2. 項目方升級合約。
3. 升級後沒過幾個 block，攻擊者調用 `redeem()` 獲利。

### 後續
- 攻擊者將資金轉移到 Tornado Cash
- Shata Capital 表示他們掌握了有關攻擊者身份的線索，並給他們 24 小時的時間來返還資金。（懷疑是自己人，畢竟怎麼可能升級之後馬上知道要展開攻擊）

{{< tweet user="CertiKAlert" id="1630509179960979457">}}

### Refernece
- https://twitter.com/BeosinAlert/status/1630884733671579653
- https://twitter.com/peckshield/status/1630490333716029440

## Jump Crypto & Oasis Counter Exploit

### 前情提要
2022 年 2 月，跨鏈橋 Wormhole 遭到攻擊，損失了 12 萬個 ETH，當時總價值約 3.25 億美元。

2023 年 1 月份，攻擊者在 MakerDao 旗下的 Oasis 上存入了 2.18 億美元的抵押品，並開設了兩個 vault，循環抵押 wstETH、rETH 借貸 DAI 做多 ETH。

2023 年 2 月 16 日白帽聯繫了 Oasis 團隊，向其披露了一個漏洞，能夠將任何 vault 中的資產與債務轉移。

2023 年 2 月 21 日，在英格蘭及威爾斯高等法院的命令下，Oasis 團隊利用可升級的 Oasis 合約，從 Wormhole Exploiter 的 vault 中轉移了抵押品與債務。


### Vault 30100

Oasis 提供 Automation Services（AutomationBot 合約），能夠創建 stop-loss trigger 自動幫用戶的 vault 賣抵押品還貸款等。

只要在 vault 中加入 automation trigger，AutomationBot 合約就能獲得對 vault 的訪問權限。然而，Oasis 合約可升級。

- 歷史數據：
  - https://oasis.app/30100#history

### 利用步驟
- 攻擊交易：
  - https://etherscan.io/tx/0x4f4117317a9f69915cbd060dc649c91bdc2963ea326ede46b14a2d8ef9007617

1. 設定 upgrade delay 到 0
2. 部署 Authorizer 及 Executor 合約
3. 通過 ServiceRegistry 替換合約，McdView -> Authorizer，McdView 用來控制 vault 的 ratio，使 vault 可以觸發 stop-loss。MultiProxyActions -> Executor，替換了 CloseCommand delegatecall 的一個合約（stop-loss的邏輯合約），不執行 stop-loss 操作而是創建一個新的 vault，將攻擊者的 vault 資產及債務轉移到新的 vault（CDP_MANAGER shift操作）。
4. 轉移 vault 的所有權到 Oasis Multisig
5. 關閉 vault
6. 建立一個新的 vault 並轉移抵押品與債務
7. 將 proxy contract 回復到原地址


*Oasis 違背了去中心化的精神...
未來會不會有更多類似的事情發生？*



### Reference
- https://www.blockworksresearch.com/research/we-do-a-little-counter-exploit