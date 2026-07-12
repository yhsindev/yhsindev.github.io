---
title: 用 Chirpy + GitHub Pages 架起我的部落格
date: 2026-07-12 00:30:00 +0800
categories: [網站]
tags: [jekyll, chirpy, github-pages]
---

這是本站的第一篇文章，內容就是這個網站本身是怎麼架起來的。

## 為什麼要有自己的部落格

身為研究生，平常讀 paper、trace code、做實驗，學到的東西如果不寫下來，過幾個月就還給網路了。筆記軟體我也用，但公開寫作有幾個好處：

- 寫給別人看，會逼自己把東西真正想清楚
- 累積下來就是可以放在履歷上的個人網站
- 找工作、遞名片時，能給出一個代表自己的網址

## 技術選擇：Jekyll + Chirpy + GitHub Pages

需求很簡單：**免費、快速上線、之後維護越省事越好**。

最後選了 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 這個 Jekyll 佈景主題，理由：

- 外觀好看，搜尋、深色模式、分類、標籤、歸檔、RSS 全部內建
- 官方提供 [chirpy-starter](https://github.com/cotes2020/chirpy-starter) 範本，主題以 gem 形式引入，升級只要改版本號，自己的客製不會被蓋掉
- 建置交給 GitHub Actions，本機不需要裝 Ruby 也能發文

## 架設步驟

1. 用 chirpy-starter 範本建立名為 `<username>.github.io` 的 repo
2. 修改 `_config.yml`：語言（`zh-TW`）、時區、標題、GitHub 帳號、頭像
3. 到 repo 的 **Settings → Pages**，把 Source 設成 **GitHub Actions**
4. `git push`，等 Actions 跑完，網站就上線了

之後發文的日常流程只有一件事：

```bash
# 在 _posts/ 新增一個 Markdown 檔，檔名格式固定
_posts/2026-07-20-my-new-post.md
git add . && git commit -m "post: my new post" && git push
```

幾分鐘後文章自動上線。

## 客製的部分

在預設的部落格功能之外，我加了一個「作品集」分頁（`_tabs/projects.md`），手動精選要展示的專案，只放有在維護、拿得出手的，其他舊 repo 不會出現在這裡。

## 之後想做的事

- [ ] 掛上 giscus 留言系統（用 GitHub Discussions 當後端）
- [ ] 微調樣式，加一點個人風格
- [ ] 名片放上網站 QR code，研討會直接遞

如果你也想架一個，照上面的步驟大概一個下午就能完成。
