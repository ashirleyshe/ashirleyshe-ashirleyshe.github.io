---
title: "[Defi Hack] 2022-10-02 TransitSwap 安全事件分析"
date: 2022-10-12
draft: true
subtitle:    ""
description: ""
author:      "Ashley Hsu"
image:       ""
tags:        ["Defi Hack"]
categories:  ["Blockchain" ]
---

- 日期: 2022/10/02
- 金額：＄21M
- 攻擊者：0xcc3d51c137e2360a34d58840e65a501cfc33ae7d
- 攻擊交易：<https://bscscan.com/tx/0x4d988ccde0caa266d88b459b710a7fbb6a44d5d2bccf91ecf4b79ba24564877c>
- Root cause: No data validation

{{< tweet user="BlockSecTeam" id="1576428812514250753" >}}

## Transit Swap
[Transit Swap](http://swap.transit.finance/#/) 是一個跨鏈聚合閃兌平台，各大主流公鏈幾乎都有支援，雖然網頁上掛著由 PeckShield 提供安全审计，不過 PeckShield 回應並無審計這個部分。

## 兌換流程
執行 swap 時，主要牽涉到3個合約：
