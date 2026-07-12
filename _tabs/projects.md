---
title: 作品集
icon: fas fa-code
order: 4
---

精選的個人專案，持續更新中。

## kernel-hash — Linux 核心雜湊函式取捨分析

比較 `jhash2()`、`hsiphash()`、`siphash()` 三種雜湊函式用於 Linux 核心雜湊表的安全與效能取捨，以 Open vSwitch（OVS）kernel datapath flow table 為目標場景：

- 自建核心雜湊函式微基準測試（microbenchmark）
- OVS datapath 測試環境與整合成本量測
- Hash-flooding 攻擊與碰撞、bucket 分布分析

**使用技術**：C（kernel module）、Python、Shell、OVS/OVN

[:fab fa-github: GitHub Repo](https://github.com/yhsindev/kernel-hash){: target="_blank" }

<!-- 新專案照上面的格式加一段即可 -->
