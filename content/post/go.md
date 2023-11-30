---
title: "[Go] Notes"
date: 2022-10-11T16:21:49+08:00
draft: true
subtitle:    ""
description: ""
author:      "Ashley Hsu"
image:       ""
tags:        ["go"]
categories:  ["Tech" ]
---
## Method | Function Receiver
在看 ival 的時候發現 Go 的 function 怎麼長得奇奇怪怪，原來是因為 Go 不是物件導向，沒有 class，所以用 `receiver` 加上 `Type` 來做出類似的效果。

Method 就是一個有 receiver 的 function

```go
type Cat struct {
    name string
    age int
}
func (c Cat) getInfo() string {
    return c.name
}
func main() {
    c := Person{name: "EE", age: 3}
    fmt.Println(p.getInfo())  // EE
}
```