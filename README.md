# zenn-contents

Zenn記事管理用リポジトリ。GitHub連携でZennに自動反映される。

## 構成

- `articles/` — 記事本体(Markdown)
- `books/` — 本(今のところ未使用)

## ローカルプレビュー

```
npm install
npx zenn preview
```

## 新規記事の作成

```
npx zenn new:article --slug 記事のslug --title "タイトル" --type tech --emoji "🕵️"
```

* [📘 Zenn CLIの使い方](https://zenn.dev/zenn/articles/zenn-cli-guide)
