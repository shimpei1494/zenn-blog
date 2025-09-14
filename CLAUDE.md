# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

このリポジトリは Zenn CLI を使用した技術ブログプロジェクトです。技術記事を日本語で執筆・管理しています。

## 基本コマンド

### 開発・プレビュー
```bash
npx zenn preview        # ローカルプレビューサーバー起動
npx zenn new:article    # 新しい記事作成
npx zenn list:articles  # 記事一覧表示
```

### 依存関係管理
```bash
npm install             # 依存関係インストール
npm update              # 依存関係更新
```

## アーキテクチャとディレクトリ構造

```
├── articles/           # 記事ファイル（Markdown + YAML frontmatter）
├── images/            # 記事用画像（記事slugごとにディレクトリ分割）
├── books/             # 本形式コンテンツ（現在未使用）
└── package.json       # Zenn CLI設定
```

## Zenn記事執筆の規約

### フロントマター
```yaml
---
title: "記事タイトル"
emoji: "🧠"
type: "tech"           # "tech" または "idea"
topics: [ai, llm, deeplearning, 初心者]
published: true
published_at: "2024-04-27 12:54"  # 任意
---
```

### 記事構造パターン
- **導入:** `## はじめに` または `## 概要` で開始
- **メッセージボックス:** `:::message` で補足説明
- **詳細セクション:** `:::details タイトル` で詳細内容
- **画像:** `/images/{記事slug}/` に格納、サイズ指定 `=500x` 使用可能

### コードブロック
```markdown
```python:filename.py
コード内容
```
```

### 記事のトーン
- 丁寧で読みやすい日本語
- 専門用語の説明を含む
- 実践的かつ学習過程を共有するスタイル

## 画像管理
- ローカル画像は `/images/{記事slug}/` に格納
- Zennアップローダーとローカル管理を併用
- 記事内では相対パス `/images/...` で参照