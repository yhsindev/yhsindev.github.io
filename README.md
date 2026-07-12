# yhsindev.github.io

個人網站：技術部落格＋作品集。
Jekyll + [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)（經 [chirpy-starter](https://github.com/cotes2020/chirpy-starter)），由 GitHub Actions 自動建置部署。

網址：<https://yhsindev.github.io>

## 發新文章

```bash
# 1. 在 _posts/ 新增檔案，檔名格式固定：YYYY-MM-DD-slug.md
# 2. 開頭加 front matter：
#    ---
#    title: 文章標題
#    date: 2026-07-20 12:00:00 +0800
#    categories: [分類]
#    tags: [標籤1, 標籤2]
#    ---
# 3. 推上去即自動部署
git add . && git commit -m "post: 文章標題" && git push
```

## 常用位置

| 要改什麼 | 檔案 |
|---|---|
| 網站標題／副標／頭像 | `_config.yml` |
| 關於頁 | `_tabs/about.md` |
| 作品集 | `_tabs/projects.md` |
| 設計文件 | `docs/DESIGN.md` |

## 主題升級

Chirpy 出新版時：改 `Gemfile` 裡 `jekyll-theme-chirpy` 的版本號 → push。
