# haru-content

`haru` の書いたもの一式を管理するリポジトリ。

## ディレクトリ

- `articles/` — Zenn 記事。push すると [zenn.dev/haru0416](https://zenn.dev/haru0416) に自動公開される
- `books/` — Zenn 本。今は空（Zenn 連携が両ディレクトリの存在を要求するため `.keep` だけ）
- `blog/` — Portfolio サイト ([haru0416.dev](https://haru0416.dev)) 専用の記事。`/blog/<slug>` で配信
- `images/` — Zenn 記事の画像置き場

## 連携

- **Zenn**: 個人アカウント `haru0416` の GitHub 連携先として登録。`articles/*.md` の frontmatter に `published: true` を入れて push すると公開される
- **Portfolio**: ビルド時に `blog/*.md` を fetch、`/blog/<slug>` でレンダリング。`draft: true` の記事はスキップ

## frontmatter

### `articles/` (Zenn 公式テンプレ)

```yaml
---
title: タイトル
emoji: 🧠
type: tech  # tech or idea
topics: [rust, ai]
published: true
published_at: "2026-05-14 11:00"
---
```

### `blog/` (Portfolio)

```yaml
---
title: タイトル
date: 2026-05-14
description: 一行要約（OG description にも使用、省略可）
draft: false
---
```
