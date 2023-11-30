---
title: "Vyper notes"
date: 2023-09-26
draft: false
subtitle:    ""
description: ""
author:      "Ashley Hsu"
image:       ""
tags:        ["Vyper"]
categories:  ["Blockchain"]
---
## Install Vyper
```
pip3 install vyper
```

## Tools
### Vyper intepreter - [titanoboa](https://github.com/vyperlang/titanoboa#titanoboa)

```python
# simple.vy
@external
def foo() -> uint256:
    x: uint256 = 1
    return x + 7
```
```python
>>> import boa

>>> simple = boa.load("examples/simple.vy")
>>> simple.foo()
    8
>>> simple.foo()._vyper_type
    uint256
```

## Compiler bugs
https://github.com/vyperlang/vyper/security?page=1

