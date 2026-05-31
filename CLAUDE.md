# Zenn ブログリポジトリ

Zenn の技術記事を管理するリポジトリです。

## 記事執筆フロー

1. ブランチを切る（`new/<記事スラッグ>`）
2. `make new-arti TITLE="タイトル"` で記事ファイルを生成
3. 本文を執筆（`articles/<hash>.md`）
4. PR を作成してレビュー・修正
5. `published: true` に変更してマージ → Zenn に公開

## 重要ルール

- **編集前に必ずブランチを切ること**
- `published: false` のまま執筆し、公開直前に `true` に変える
- `node_modules/` の変更はコミットしない（記事ファイルのみステージする）

## 利用可能なスキル

| スキル | 説明 |
|--------|------|
| `/blog-new` | 新しい記事を作成する（ブランチ作成〜執筆） |
| `/blog-log` | 執筆セッションのログを `.claude/session/` に保存する |

## Makefile コマンド

| コマンド | 説明 |
|---|---|
| `make new-arti TITLE="タイトル"` | 新記事ファイルを生成（`articles/<hash>.md`） |
| `make preview` | ローカルプレビューサーバーを起動 |

## 記事 frontmatter テンプレート

```yaml
---
title: "記事タイトル"
emoji: "🔥"
type: "tech"
topics: ["topic1", "topic2"]
published: false
---
```

## ディレクトリ構成

```
articles/   # 記事 Markdown ファイル
images/     # 記事内画像
books/      # Zenn 本
.claude/
├── skills/ # カスタムスキル
└── session/ # 執筆セッションログ（gitignore 済み）
```
