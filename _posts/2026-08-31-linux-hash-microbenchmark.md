---
title: "keyed hash 一定比較慢嗎？三種核心雜湊的 microbenchmark"
description: 把 jhash2、hsiphash、siphash 單獨抓進核心重複跑，量 cycles per hash，得到一個跟直覺相反的結果。
date: 2026-08-31 20:00:00 +0800
categories: [Linux 核心, 雜湊函數]
tags: [linux-kernel, hash, benchmark, perf, kernel-module]
math: true
---

> Linux 核心雜湊函式系列的第二篇。上一篇談雜湊常數背後的數學，這篇開始做實測，先量三個函式本身的成本。其他篇：① 黃金比例與 Three-gap 定理、③ 接進 Open vSwitch 資料路徑、④ HashDoS 攻擊與防禦。
{: .prompt-info }

雜湊表能做到平均 O(1) 的查詢，前提是資料被均勻打散。但如果雜湊函式是公開又固定的，攻擊者可以反算出一堆會撞進同一格的輸入，把查詢拖成 O(n) 進而拖垮服務，這種手法叫 HashDoS。防禦方式之一是改用帶密鑰（keyed）的雜湊，等於加一把只有系統知道的鑰匙，讓攻擊者猜不到結果，代價是可能變慢。

Linux 核心正好面對這個取捨。網路元件 Open vSwitch（在虛擬網路裡把封包轉來轉去的軟體交換器）的封包查表，用的是快但不帶密鑰的 jhash，換成帶密鑰的 siphash 會比較安全，但會慢多少？動系統之前，得先知道這三個候選函式本身有多貴。這篇就是最底層的一步：把函式單獨抓出來，在核心裡重複跑上百萬次，量它的成本。

上一篇談 `hash_32()` 的黃金比例常數，跟這篇是同個系列，但這篇獨立看也沒問題。

## 三個受測對象

| 函式 | 輸出 | 定位 |
| --- | --- | --- |
| `jhash2()` | 32-bit | 非加密雜湊，主打低延遲 |
| `hsiphash()` | 32-bit | 帶密鑰、回合數精簡的版本，用來擋 HashDoS |
| `siphash()` | 64-bit | 完整的帶密鑰雜湊，安全性最高、成本也最高 |

jhash 有 `jhash()` 和 `jhash2()` 兩個版本。`jhash()` 收 `const void *`，以 byte 為單位，會透過 `__get_unaligned_cpu32` 處理未對齊記憶體；`jhash2()` 收 `const u32 *`，向編譯器保證輸入已對齊，直接以陣列存取。我選 `jhash2()`，因為它比較適合當對齊輸入的基準，也是 OVS `flow_hash()` 實際用的版本。

## 第一個坑：輸出長度不一樣，怎麼比才公平

`siphash()` 輸出 64-bit，另外兩個是 32-bit。如果直接拿去比，siphash 會佔便宜，因為更大的輸出空間本來就比較不容易碰撞，這樣比等於在比不同的東西。而且 OVS 最後只要一個 `u32`。

所以我把 siphash 的 64-bit 輸出摺疊（fold）成 32-bit，用高低位 XOR 合併，並把這步的成本也算進計時範圍，讓量測貼近它在 OVS 裡真正被使用的樣子。

```c
/* Fold SipHash's 64-bit output to the 32-bit hash used by OVS. */
u64 h = siphash(buf, len, &skey);
u32 folded = (u32)h ^ (u32)(h >> 32);
```

## 為什麼用 kernel module 而不是 KUnit

KUnit 是核心的單元測試框架，結果以 KTAP 格式出現在 kernel log，適合測邏輯正確性。但我要的是重複量測，需要自己控制迴圈次數、輸出格式，還要搭配 `perf`。所以我選了 out-of-tree 的核心模組，用 `module_param` 把三個變因開出來，一份模組就能測所有組合。

```bash
# hash_type: 0=jhash2, 1=hsiphash, 2=siphash
sudo insmod hash_microbench.ko hash_type=0 input_len=64 iterations=10000000
dmesg | tail -20
```

模組裡有幾個要點：先配置一塊固定的 pseudo-random buffer，避免每次雜湊前重新產生資料；在迴圈裡用 `rdtsc_ordered()` 讀 cycle 數（這個指令直接讀 CPU 的週期計數器）；結果寫進一個 `volatile` sink，免得編譯器發現輸出沒被用到就把整個迴圈最佳化掉；最後印出 total cycles 與 cycles per hash。

## 輸入長度怎麼選

長度不是亂挑的，要對應 OVS flow key（識別一條連線的那組欄位，例如來源／目的 IP、port、協定）的實際大小。`jhash2()` 以 `u32` word 為單位，`hsiphash()` 和 `siphash()` 收 byte buffer，而 OVS 實際拿去雜湊的資料長度會隨情況改變。我測五種長度，涵蓋短、中、長 key 到壓力測試：

| 輸入長度 | 意義 |
| ---: | --- |
| 16 bytes | 小型 key，接近 5-tuple 片段 |
| 32 bytes | 常見短 flow key |
| 64 bytes | 中型 key |
| 128 bytes | 較完整的 flow key 區段 |
| 256 bytes | 壓力測試，觀察長輸入成本 |

## 結果：一個跟直覺相反的發現

![三種雜湊在五種輸入長度下的 cycles per hash](/assets/img/posts/linux-hash-microbenchmark/cycles-per-hash.png)
_cycles per hash：短輸入 jhash2 最低，64 bytes 以上 hsiphash 反超_

先講符合預期的部分：短輸入時 `jhash2` 成本最低，符合它偏低延遲的設計；`siphash` 在所有長度都最貴，因為 keyed hash 需要額外的攪拌回合。

不符合預期的是：當輸入長到 64 bytes 以上，`hsiphash` 的 cycles per hash 反而比 `jhash2` 低了大約 4 個 cycle。帶密鑰、比較安全的那一個，在長輸入時居然比較快。

摘錄三個長度的實測值（左為 cycles per hash，右為 instructions per hash）：

| 輸入 | jhash2 | hsiphash | siphash | | jhash2 | hsiphash | siphash |
| ---: | ---: | ---: | ---: | --- | ---: | ---: | ---: |
| 16 | 16.7 | 28.6 | 45.4 | | 83.4 | 148.4 | 207.4 |
| 64 | 61.4 | 57.2 | 101.9 | | 235.5 | 274.4 | 423.5 |
| 256 | 249.8 | 171.0 | 324.9 | | 843.8 | 778.6 | 1287.8 |

![三種雜湊在五種輸入長度下的 instructions per hash](/assets/img/posts/linux-hash-microbenchmark/instructions-per-hash.png)
_instructions per hash：趨勢與 cycles 相近，但要到 256 bytes hsiphash 才低於 jhash2_

關鍵在固定成本和變動成本的差別。`jhash2` 的固定成本低，所以短輸入贏；但 `hsiphash` 每多一個 byte 的邊際成本比較低。從 16 到 256 bytes，`jhash2` 平均每個 byte 增加約 0.97 cycle，`hsiphash` 只增加約 0.59 cycle。輸入夠長時，較低的斜率就追過、甚至超越了較低的截距。

這件事對後面的 OVS 決策很重要：換成 keyed hash 到底划不划算，答案取決於 `flow_hash()` 真實的輸入長度落在哪一段。這個伏筆留到第三篇。

至於 branch-misses 和 cache-misses，三種函式的差異都很小（每次雜湊分別約 0.002 和 0.01 的量級），所以效能差異不是來自分支預測失敗或快取，而是來自演算法本身的運算結構。

## 差在運算結構

`jhash2` 的核心是 `__jhash_mix`，用 add、sub、xor 和 `rol32`（位元循環左移）把三個狀態值快速攪散，這類操作便宜。

`siphash` 系列則是 ARX 結構（Add-Rotate-XOR），核心是 `SIPROUND`：

```c
#define SIPROUND \
    do { \
        v0 += v1; v1 = rol64(v1, 13); v1 ^= v0; v0 = rol64(v0, 32); \
        v2 += v3; v3 = rol64(v3, 16); v3 ^= v2; \
        v0 += v3; v3 = rol64(v3, 21); v3 ^= v0; \
        v2 += v1; v1 = rol64(v1, 17); v1 ^= v2; v2 = rol64(v2, 32); \
    } while (0)
```

SipHash-2-4 每 8 bytes 要做 2 輪 SIPROUND，收尾還要再做 4 輪。那 4 輪收尾是固定成本，對短輸入特別不划算，這解釋了為什麼 siphash 在短輸入吃虧。但 SIPROUND 裡 `v0` 到 `v3` 有很強的資料相依性，CPU 反而在長輸入時能穩定地把管線填滿，邊際成本低。

## perf 的一個坑：counter multiplexing

量硬體計數器時踩到一個坑，值得記下來。初版我一次量六個 event（cycles、instructions、branches、branch-misses、cache-references、cache-misses），但檢查 raw output 發現有些 event 的 counted percentage 只有 66% 到 83%。這代表發生了 hardware counter multiplexing：實體計數器不夠用，核心以分時多工共用，量到的值含有估算成分。

解法是把 event 分成三組分開量，每組的計數器數量都在硬體上限內：

```text
core   = cycles,instructions
branch = branches,branch-misses
cache  = cache-references,cache-misses
```

分組後再看 raw log，counted percentage 回到 100%，數字才可信。這種「工具本身的量測誤差」如果沒抓出來，後面的分析全都會偏掉。

## 小結

三個函式本身的成本模型有了：短輸入 `jhash2` 最快，長輸入 `hsiphash` 反而可能最快，`siphash` 最貴但沒貴太多。keyed hash 不一定比較慢，這個結論會直接影響 OVS 該不該換。

但函式在 microbenchmark 裡的成本，跟它放進真實系統的成本是兩回事。OVS 每秒查表可能數百萬次，還牽涉 cache locality、函式呼叫開銷、以及雜湊只佔整條 lookup path 的一小段。下一篇我把這三個函式接進 Open vSwitch 的 `flow_hash()`，看在真實 datapath 上會變成什麼樣子。

---

*完整的模組原始碼、perf 腳本與數據在 [kernel-hash](https://github.com/yhsindev/kernel-hash)。*
