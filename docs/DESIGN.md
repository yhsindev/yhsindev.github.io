# 網站設計文件（2026-07-12）

## 目標

以部落格為核心（B）＋作品集頁面（C）的個人網站；名片 QR code 指向本站。
非學術取向。快速上線、低維護成本、可長期持續更新。

## 技術決策

- **Jekyll + Chirpy**（chirpy-starter 範本，主題以 gem 引入，升級不蓋客製）
- **GitHub Pages + GitHub Actions** 自動建置部署（push 即上線，本機免 Ruby）
- 介面語言 `zh-TW`、時區 `Asia/Taipei`、文章以繁體中文撰寫

## 結構

| 頁面 | 來源 | 說明 |
|---|---|---|
| 首頁 | Chirpy 內建 | 文章列表（搜尋／深色模式／分類／標籤／歸檔內建） |
| 作品集 `/projects/` | `_tabs/projects.md` | 手寫精選，不自動抓 GitHub；首發只放 kernel-hash |
| 關於 `/about/` | `_tabs/about.md` | 簡短自介；名片 QR 的落點之一 |
| 文章 | `_posts/*.md` | 檔名 `YYYY-MM-DD-slug.md`，push 後自動上線 |

## 維護流程

- 發文：`_posts/` 加 `.md` → push
- 主題升級：`Gemfile` 改 `jekyll-theme-chirpy` 版本號 → push
- 建置失敗：GitHub Actions 頁面看紅叉 log（內建 html 檢查）

## 之後再做（不擋上線）

名片設計（QR→本站）、giscus 留言、自訂樣式、真實姓名／頭像替換。
