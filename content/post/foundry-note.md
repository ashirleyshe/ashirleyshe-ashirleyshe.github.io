---
title: "Foundry Notes"
subtitle:    ""
description: ""
date: 2023-09-28
author:      "Ashley Hsu"
image:       ""
tags:        ["foundry"]
categories:  ["Solidity" ]
---

## Install
```
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

## Init
```bash
forge init hello
cd hello
```

## build

```bash
forge build
```

## test

```bash
forge test

forge test -vv

# unit tests 
forge test -m "testExample()"
```

## Openzeppelin lib
```
forge install openzeppelin/openzeppelin-contracts
```
### Remapping
Add remappings to `foundry.toml`.
```toml
remappings = ["@openzeppelin/=lib/openzeppelin-contracts/"]
```
We can import openzeppelin contracts like this:
```solidity
import "@openzeppelin/contracts/access/Ownable.sol";
```

## Script
```
forge script script/Contract.s.sol:ContractScript --rpc-url ${RPC_URL}  --private-key ${PRIVATE_KEY} --legacy -vvvv --broadcast
```

## Makefile
```makefile
-include .env

all: build

build:; forge build

test :; forge test 
 
deploy :; forge script script/Solve.s.sol:Solve --rpc-url ${RPC_URL}  --private-key ${PRIVATE_KEY} --legacy -vvvv --broadcast
```
Set the `.env` file:

```
RPC_URL=http://10.132.61.18:10033/3695cc9d-4573-4888-91f1-d9ffe61187d0
PRIVATE_KEY=0xf6f246042debbc0c5ddd200e72a3d12e03038d651a5d0c5e744ac30a94e5c3d7
CHAL_ADDR=
``` 
Remember to load the variables in the `.env` file.
```
source .env
```
## `foundry.toml` settings
```
rpc_endpoints = { mainnet = "https://rpc.ankr.com/eth", optimism = "https://rpc.ankr.com/optimism" , fantom = "https://rpc.ankr.com/fantom", arbitrum = "https://rpc.ankr.com/arbitrum" , bsc = "http://10.132.83.97:8000/bsc" }
```

## Cheat code
```solidity
// fork bsc at block 21157028
vm.createSelectFork("bsc", 21157028);

// set address(this) 0 ether
vm.deal(address(this), 0);

vm.startPrank(attacker);
vm.stopPrank();

// load address from env
chal = Challenge(vm.envAddress("CHAL_ADDR"));
// load private key from env
privKey = uint256(vm.envBytes32("PRIV_KEY"));
```

## Example script for ctf

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

// import { Challenge } from "../src/Challenge.sol";
import { CommonBase } from "forge-std/Base.sol";
import "forge-std/console.sol";

contract Pwn {
    constructor() payable {
        // do something...
        console.log("print something...");
    }
}

contract Solve is CommonBase {
    Challenge chal;
    uint256 privKey;

    constructor() {
        chal = Challenge(vm.envAddress("CHAL_ADDR"));
        privKey = uint256(vm.envBytes32("PRIV_KEY"));
    }

    function run() public {
        vm.startBroadcast(privKey);
        Pwn p = new Pwn();
        vm.stopBroadcast();
    }
}
```

## Reference
- [Foundry Book](https://book.getfoundry.sh/)


![](https://stickershop.line-scdn.net/stickershop/v1/product/1976/LINEStorePC/main.png)