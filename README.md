# miyamankyushu Zenn

[H.Watari のZennアカウント](https://zenn.dev/miyamankyushu)向けの記事を管理するローカルリポジトリです。

## ローカルでの執筆

Node.js 22以降を利用します。初回は依存パッケージをインストールします。

```bash
npm ci
```

プレビューを起動します。

```bash
npm run preview
```

ブラウザで `http://127.0.0.1:8000` を開きます。

## 原稿

- `articles/snowflake-aim-redshift-iceberg.md` はAIM紹介記事の下書きです。
- `articles/cortex-code-cli-install-guide.md` は既存の原稿です。Zenn用Front Matterがなく、同期すると重複記事になる可能性があるため、`.gitignore` で同期対象から外しています。

AIM記事を文章チェックします。

```bash
npm run textlint
```

既存記事を含めた全記事のチェックも実行できます。

```bash
npm run textlint:all
```

## 公開状況

GitHubリポジトリは https://github.com/hwatari-snow/zenn （public）です。AIM記事は `published: true` でpush済みです。

Zennのデビュー画面でGitHub連携を有効にし、このリポジトリと `main` ブランチを選択すると公開されます。

- 掲載した図版には、デモのDB名、接続名 `coco_desktop`、ACCOUNTADMIN表示、承認省略設定が映っています。
- 未確認項目は、デモの使用バージョン、8テーブルと6テーブルの差分理由、最終DDLの保存先設定です。

## 公開前の確認

- 新規記事は `published: false` のまま内容を確認します。
- AIM記事は公式情報による概要説明と、12枚のデモ画像に基づく実践記事です。執筆作業では移行を実行していません。
- 画面で確認できる結果は、6テーブルの行数一致という報告と、SnowsightでのCATEGORYテーブルの表示です。全行・業務ロジックの検証完了は主張していません。
- 公開前に使用バージョン、変換対象8テーブルと結果6テーブルの差分、最終DDLのSnowflake storage設定を確認してください。図版の公開可否も確認対象です。
- 最終DDLの確認は未完了です。デモの`AIM_MIGRATION_DB`は別アカウントにあり、接続中のアカウントには存在しません。記事では、保存先の設定値を確認していないと明記しています。
- 既存のCLI記事にはZenn用Front Matterがありません。同期する場合は、公開済み記事のslugとの対応を確認してから`.gitignore`から外してください。名前を変えて同期すると別記事になる可能性があります。
- 認証情報、接続設定、実データ、顧客情報をコミットしないでください。`.gitignore` だけでは機密情報の混入を防げません。
- 公開先のPublicationは未設定です。必要な場合のみ確認後に設定します。

公開後に修正する場合は、原稿を編集してcommitとpushを実行するとZenn側に反映されます。

## AIM記事の画像

- 原本は `articles/Snowflake AIM/` に残し、誤って公開しないようGitの管理対象から除外しています。
- 掲載用のJPEG画像は `images/snowflake-aim-redshift-iceberg/` にあります。記事からは `/images/` 始まりのパスで参照します。
- `03-start.jpg` には承認省略設定が写っています。記事では、承認省略は利用の前提ではないことと、操作の対象・影響を確認する旨を明記しています。
- `12-snowsight.jpg` は無関係なデータベース名が並ぶ左側のサイドバーを除いています。テーブル名、Iceberg表示、データプレビューは残しています。
- `10-extraction.jpg` は、背景が暗く読みにくい画面から、注釈付きの明るい画面へ差し替えています。この画像の原本は `articles/Snowflake AIM/` にはありません。
- 残りの画像は形式変換のみです。掲載用画像を目視確認した範囲ではパスワードやトークンは見つかっていません。接続名、デモDB名、サンプルデータ、ACCOUNTADMIN表示は残っています。公開前の最終レビューは必要です。

## 参考

- [Zenn CLIで記事・本を管理する方法](https://zenn.dev/zenn/articles/zenn-cli-guide)