---
title: "Aptos Move Note"
date: 2023-11-29
draft: false
subtitle:    ""
description: ""
author:      "Ashley Hsu"
image:       ""
tags:        ["Aptos"]
categories:  ["Blockchain"]
---

## Global Storage

- `move_to<T>(&signer,T)`
  - 只能 move_to 自己的 account，因為要傳入 `&signer`
  - 同一種 resource 只能放一個
- `borrow_global<T>(address): &T`
  - 可讀任何 account 的 resource 值，前提是 module 需要實作這個功能
- `borrow_global_mut<T>(address): &mut T`
  - 可讀寫 resource 值
- `move_from<T>(address): T`
- `exists<T>(address): bool`

### Notes
1. 都需要有 `key` 能力的 resource
2. resource 只能在被定義的 module 內更改，需實作一個修改 resource 的 function

## Acquires
- 定義所有這個 function 會用到的 resource
- 只需定義這個 module 的，就算會 call 到其他 moudule 可以不用管
- module 不能存取其他 module 的 resource
- 只要 function 有用到 Global Storage 的操作都需要寫

## Coin
- coin init 後會有 burn_cap, mint_cap, freeze_cap
- burn_cap 可以直接 burn 任意地址 coin，使用 `burn_from`
  -  https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/coin.move#L332

## 升級
可以看是v幾
```
aptos move list --url https://aptos-mainnet-rpc.allthatnode.com/v1 --account <account>
```

## Reference
- https://www.zellic.io/blog/top-10-aptos-move-bugs/#1-lack-of-generics-type-checking
- https://move-book.com/index.html
- https://move-language.github.io/move/