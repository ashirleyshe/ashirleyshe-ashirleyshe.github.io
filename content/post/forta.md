---
title: "Forta Bot Notes"
date: 2023-07-10
draft: true
subtitle:    ""
description: ""
author:      "Ashley Hsu"
image:       ""
tags:        ["Forta"]
categories:  ["Blockchain"]
---

## forta cli
```
npm run tx 0xf9c43e15ef2abfec163ec3b1165f18a5119ba119b6e059fc924903e5251e3543

npm run block 12821978

npm run start
```



## debug_traceTransaction
https://docs.chainstack.com/reference/ethereum-tracetransaction

```
curl --request POST \
     --url https://nd-422-757-666.p2pify.com/0a9d79d93fb2f4a4b1e04695da2b77a7/ \
     --header 'accept: application/json' \
     --header 'content-type: application/json' \
     --data '
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "debug_traceTransaction",
  "params": [
    "0x4fc2005859dccab5d9c73c543f533899fe50e25e8d6365c9c335f267d6d12541",
    {
      "tracer": "unigramTracer"
    }
  ]
}
'
```
```
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "PUSH1": 21,
    "MSTORE": 5,
    "CALLDATASIZE": 4,
    "PUSH2": 19,
    "JUMPI": 9,
    "JUMPDEST": 12,
    "JUMP": 8,
    "PUSH32": 2,
    "SLOAD": 2,
    "PUSH20": 1,
    "AND": 1,
    "SWAP1": 12,
    "POP": 5,
    "DUP1": 14,
    "CALLDATACOPY": 1,
    "DUP5": 2,
    "GAS": 1,
    "DELEGATECALL": 1,
    "LT": 1,
    "CALLDATALOAD": 1,
    "SHR": 1,
    "PUSH4": 5,
    "GT": 3,
    "EQ": 3,
    "CALLER": 2,
    "DUP2": 3,
    "KECCAK256": 1,
    "CALLVALUE": 2,
    "SWAP3": 2,
    "DUP3": 2,
    "ADD": 2,
    "ISZERO": 2,
    "SWAP2": 3,
    "SSTORE": 1,
    "MLOAD": 2,
    "SUB": 1,
    "LOG3": 1,
    "STOP": 1,
    "RETURNDATASIZE": 2,
    "RETURNDATACOPY": 1,
    "RETURN": 1
  }
}
```