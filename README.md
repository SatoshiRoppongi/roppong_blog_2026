# roppong.com

ろっぽんのブログ。Hugo + PaperMod テーマで構築し、GitHub Pages でホスティング。

## 記事の作成・更新フロー

### 1. 新規記事の作成

```bash
hugo new posts/記事名/index.md
```

例:
```bash
hugo new posts/my-first-post/index.md
```

作成されるファイル: `content/posts/my-first-post/index.md`

### 2. 記事の編集

生成されたファイルを開き、frontmatter と本文を編集:

```yaml
---
title: "記事タイトル"
date: 2026-02-06
lastmod: 2026-02-06
draft: true              # 下書き状態
description: "記事の説明"
summary: "一覧に表示される要約"
tags: ["Hugo", "ブログ"]
categories: ["技術"]
cover:
  image: "images/cover.jpg"
  alt: "カバー画像の説明"
  relative: true
showToc: true
---

ここに本文を書く...
```

### 3. 画像の追加

記事フォルダ内に `images/` ディレクトリを作成し、画像を配置:

```
content/posts/my-first-post/
├── index.md
└── images/
    ├── cover.jpg
    └── photo1.png
```

記事内での参照:
```markdown
![画像の説明](images/photo1.png)
```

### 4. ローカルでプレビュー

```bash
hugo server -D
```

ブラウザで http://localhost:1313/ を開いて確認。

`-D` オプションで下書き（`draft: true`）の記事も表示されます。

### 5. 公開

#### 5.1 下書きを公開状態に変更

```yaml
draft: false
```

#### 5.2 コミット & プッシュ

```bash
git add .
git commit -m "Add post: 記事タイトル"
git push origin main
```

GitHub Actions が自動でビルド・デプロイします（1〜2分）。

### 6. 公開確認

https://roppong.com/ で記事が公開されていることを確認。

---

## 記事の状態管理

| 状態 | 設定 |
|------|------|
| 公開 | `draft: false` |
| 下書き（非公開） | `draft: true` |
| 限定公開（検索除外） | `draft: false`, `robotsNoIndex: true`, `searchHidden: true` |

---

## ディレクトリ構成

```
.
├── archetypes/          # 記事テンプレート
├── assets/css/extended/ # カスタム CSS
├── content/
│   ├── posts/           # ブログ記事
│   ├── about.md         # プロフィール
│   ├── contact.md       # お問い合わせ
│   ├── archives.md      # 記事一覧
│   └── search.md        # 検索
├── layouts/
│   ├── shortcodes/      # カスタムショートコード
│   └── partials/        # カスタムパーシャル
├── static/              # 静的ファイル（favicon, 画像等）
├── themes/PaperMod/     # テーマ（git submodule）
└── hugo.yaml            # サイト設定
```

---

## ショートコード

### YouTube 埋め込み（プライバシーモード）

```
{{</* youtube-enhanced "VIDEO_ID" */>}}
```

### Google Form 埋め込み

```
{{</* google-form "FORM_ID" */>}}
```

### 画像（WebP 変換 + lazy load）

```
{{</* img src="images/photo.jpg" alt="説明" caption="キャプション" */>}}
```

---

## 開発コマンド

```bash
# ローカルサーバー起動（下書き含む）
hugo server -D

# ビルド
hugo --gc --minify

# 新規記事作成
hugo new posts/記事名/index.md
```

---

## リンク

- 本番サイト: https://roppong.com/
- 旧ブログ: https://roppong-blog.web.app/
