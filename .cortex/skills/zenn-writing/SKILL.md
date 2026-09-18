---
name: zenn-writing
description: "Zennの技術記事・スクラップ・本を執筆する。記事作成、アイデア出し、構成案作成、Markdown出力をサポート。トリガー: 記事を書く, Zenn, 技術記事, スクラップ, book, 執筆, ブログ"
---

# Zenn執筆スキル

## 概要このスキルを
Zenn（zenn.dev）向けの技術記事・スクラップ・本をMarkdown形式で作成するスキル。

## Zennコンテンツタイプ

| タイプ | 用途 | 出力先 |
|--------|------|--------|
| **Article** | 技術記事（tech）/ アイデア記事（idea） | `articles/{slug}.md` |
| **Scrap** | メモ・議論・TIL | `scraps/{slug}.md` |
| **Book** | 複数章の技術書 | `books/{slug}/` |

## ワークフロー

```
ユーザーリクエスト
      ↓
Step 1: コンテンツタイプ確認
      ↓
Step 2: テーマ・構成決定
      ↓
Step 3: Markdownファイル生成
      ↓
Step 4: ユーザーに出力確認
```

### Step 1: コンテンツタイプ確認

**ユーザーに確認:**

```
どのタイプのコンテンツを作成しますか？

1. 技術記事（Article - tech）- コード、チュートリアル、技術解説
2. アイデア記事（Article - idea）- 考察、ポエム、エッセイ
3. スクラップ（Scrap）- メモ、TIL、議論
4. 本（Book）- 複数章の技術書
```

### Step 2: テーマ・構成決定

**ユーザーから収集する情報:**

- タイトル
- 対象読者
- 主要なポイント（3-5個）
- 含めたいコード例やサンプル

**構成案を提示してから執筆に進む。**

### Step 3: Markdownファイル生成

コンテンツタイプに応じたフォーマットでファイルを生成。

### Step 4: 出力確認

**⚠️ MANDATORY STOPPING POINT**

生成したMarkdownをユーザーに提示し、修正が必要か確認する。

---

## 出力フォーマット

### Article（記事）

ファイル: `articles/{slug}.md`

```markdown
---
title: "記事タイトル"
emoji: "🎉"
type: "tech" # tech or idea
topics: ["topic1", "topic2", "topic3"]
published: false
---

## はじめに

[導入文]

## 本文セクション1

[内容]

## 本文セクション2

[内容]

## まとめ

[結論]
```

**フロントマター規則:**
- `title`: 60文字以内推奨
- `emoji`: 記事を表す絵文字1つ
- `type`: `tech`（技術）または `idea`（アイデア）
- `topics`: 1-5個のトピック（小文字、ハイフン区切り）
- `published`: `true`で公開、`false`で下書き

### Scrap（スクラップ）

ファイル: `scraps/{slug}.md`

```markdown
---
title: "スクラップタイトル"
emoji: "📝"
type: "scrap"
topics: ["topic1"]
closed: false
---

## 最初の投稿

[内容]
```

### Book（本）

ディレクトリ構造:

```
books/{slug}/
├── config.yaml
├── cover.png (optional)
├── chapter1.md
├── chapter2.md
└── ...
```

**config.yaml:**

```yaml
title: "本のタイトル"
summary: "本の概要説明"
topics: ["topic1", "topic2"]
published: false
price: 0 # 0=無料, 有料は200-5000
chapters:
  - chapter1
  - chapter2
```

**各チャプター:**

```markdown
---
title: "チャプタータイトル"
free: true
---

[チャプター内容]
```

---

## Zenn Markdown記法

### 基本記法

| 記法 | 用途 |
|------|------|
| `## 見出し` | セクション見出し |
| `**太字**` | 強調 |
| `[リンク](url)` | リンク |
| `` `code` `` | インラインコード |

### コードブロック

````markdown
```言語名:ファイル名
コード
```
````

例:
````markdown
```python:main.py
def hello():
    print("Hello, Zenn!")
```
````

### Zenn独自記法

**メッセージボックス:**

```markdown
:::message
通常のメッセージ
:::

:::message alert
警告メッセージ
:::
```

**アコーディオン:**

```markdown
:::details タイトル
折りたたみコンテンツ
:::
```

**数式（KaTeX）:**

```markdown
$$
e^{i\pi} + 1 = 0
$$
```

**外部コンテンツ埋め込み:**

```markdown
@[tweet](ツイートURL)
@[youtube](動画ID)
@[github](リポジトリURL)
@[gist](GistのURL)
@[codepen](ペンのURL)
@[slideshare](スライドキー)
@[speakerdeck](スライドID)
```

---

## 記事構成テンプレート

### チュートリアル記事

```markdown
## はじめに
- 何を作るか
- 対象読者
- 前提知識

## 環境構築
- 必要なツール
- インストール手順

## 実装
### ステップ1: [タスク]
### ステップ2: [タスク]
### ステップ3: [タスク]

## 動作確認

## まとめ
- 学んだこと
- 次のステップ
```

### 技術解説記事

```markdown
## はじめに
- なぜこのトピックか
- 対象読者

## 背景・基本概念
- 用語説明
- 従来のアプローチ

## 詳細解説
### ポイント1
### ポイント2
### ポイント3

## 実践例

## まとめ
- キーポイント
- 参考リソース
```

### トラブルシューティング記事

```markdown
## 問題の概要
- 発生した問題
- エラーメッセージ

## 原因調査
- 調査プロセス
- 発見した原因

## 解決方法
- 解決手順
- コード修正

## 教訓・予防策
```

---

## Stopping Points

- ✋ Step 1後: コンテンツタイプ確認
- ✋ Step 2後: 構成案の承認
- ✋ Step 4: 生成ファイルの確認

---

## Output

生成物は以下のいずれか:

1. **Article**: `articles/{slug}.md` - 単一のMarkdownファイル
2. **Scrap**: `scraps/{slug}.md` - 単一のMarkdownファイル
3. **Book**: `books/{slug}/` - config.yaml + 複数のチャプターファイル

**ファイル命名規則:**
- slug: 小文字英数字とハイフンのみ
- 12-50文字程度
- 内容を表す簡潔な名前
