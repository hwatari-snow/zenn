---
title: "Snowflake AIMを使ってRedshiftからSnowflake managed Icebergテーブルへの移行を試してみた"
emoji: "❄️"
type: "tech"
topics: ["snowflake", "redshift", "iceberg", "aws"]
published: true
---

:::message
著者はSnowflakeに所属しておりますが、本記事は個人の見解であり、所属する組織の公式見解ではありません。
:::

## はじめに

こんにちは。Snowflakeでソリューションエンジニアをしている渡利です。

データウェアハウスの移行では、データをコピーするだけでは作業が終わりません。SQLの方言差分を直し、ビューやストアドプロシージャの依存関係を確認します。移行先で同じ結果が得られるかを検証する作業も必要です。

今回は、こうした移行作業を支援する **Snowflake AIM** を紹介します。後半では、CoCo DesktopからAmazon Redshiftの移行を試した画面を使い、操作の流れを追っていきます。移行先には、Snowflake storageを使用するIcebergテーブルを指定しました。

この記事は、移行を担当するエンジニア向けです。AIMの概要に続いて、接続設定、コード変換、データ移行の設定、移行後の確認を紹介します。

:::message alert
製品仕様の説明は2026年9月18日時点の公開ドキュメントに基づきます。実践パートはデモの画面記録です。画面では6テーブルの行数一致と、SnowsightでのIcebergテーブルの表示を確認しています。全行の値や業務ロジックの一致を検証した記事ではありません。
:::

## Snowflake AIMとは

Snowflake AIMは、Snowflakeへの移行とモダナイゼーションを支援するプラットフォームです。SnowConvert AIなどの移行技術を基盤としています。

![Snowflake AIMの全体像。仮想化、データウェアハウス移行、Sparkワークロードのモダナイゼーション。](/images/snowflake-aim-redshift-iceberg/01-aim-overview.jpg)

上の図には、Teradataの仮想化、データウェアハウスの移行、Sparkワークロードのモダナイゼーションが並んでいます。AIMには複数のアプローチがありますが、この記事ではデータウェアハウス移行を扱います。

利用するのは **Snowflake AIM Agent for Data Warehouses** です。Snowflake CoCo上で対話しながら、接続、コード抽出、変換、アセスメント、デプロイ、データ移行、検証を進めます。

:::message
対応する移行元や機能は、移行方式やバージョンによって異なります。図にある対応ソースすべてで、同じ工程を自動化できるという意味ではありません。利用時には公式ドキュメントを確認してください。
:::

https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/overview

### コード変換だけでなく、移行の状態を管理する

![AIMの移行オーケストレーション。人間とAIが、進捗や依存関係などの状態を共有しながら各工程を進める。](/images/snowflake-aim-redshift-iceberg/02-orchestration.jpg)

AIMでは、まずSnowConvertでコードを決定論的に変換します。そのうえでAIが残った問題の説明や修正、テスト作成を支援します。

単にSQLをLLMへ渡して書き直すだけではありません。オブジェクトの依存関係や進捗、課題を管理しながら作業を進める仕組みです。依存関係に基づく移行単位はWaveと呼ばれます。プロジェクトの状態を保存し、複数セッションで作業を継続できます。

図の上部に「人間 + AI」とある点も大切です。移行対象、実行先、受け入れ条件は人が確認します。変換が完了したことと、業務上の結果が一致することは別です。

仕組みの詳細は、AIM Migration Agentの技術解説を参照してください。

https://www.snowflake.com/en/blog/engineering/snowflake-aim-migration-agent/

## 今回試す構成

今回のデモでは、RedshiftからSnowflakeへ移行する際に、Icebergテーブルを移行先として指定しました。画面で選んでいる構成は次のとおりです。

| 項目 | デモでの指定・選択 |
| --- | --- |
| 移行元 | Amazon Redshiftの`dev`データベース |
| 移行先DB | `AIM_MIGRATION_DB` |
| テーブル形式 | Iceberg |
| 保存先の要件 | Snowflake storage |
| Orchestrator | Local |
| Worker | ローカルワーカー |
| 抽出方式 | Direct read（ODBC） |

:::message
ここでは、AIM Agentを利用できるCoCo Desktopと、移行元・移行先へ接続できる環境を前提にしています。新規導入からの全手順ではなく、移行操作の紹介です。画面に使用バージョンは写っていません。表示や選択肢は、手元の環境と異なる可能性があります。
:::

### Icebergのカタログと保存先を分けて考える

Apache Icebergは、データファイルをテーブルとして管理するためのオープンなテーブルフォーマットです。スキーマやスナップショットなどのメタデータも扱います。Parquetファイルを置くだけでIcebergテーブルになるわけではありません。

Snowflakeがカタログを管理するIcebergテーブルでも、ファイルの保存先には選択肢があります。利用者が管理するクラウドストレージを使う方法と、Snowflake storageを使う方法です。

今回依頼したのは後者です。Snowflake storageでは、Snowflakeがデータとメタデータのファイルを管理します。Icebergの保存先として、自分でS3バケットやExternal Volumeを作成する必要はありません。

公式ドキュメントでは、この構成を`CATALOG = SNOWFLAKE`と`EXTERNAL_VOLUME = SNOWFLAKE_MANAGED`で指定します。

:::details Snowflake storageを使うIcebergテーブルのDDL例
```sql
CREATE OR REPLACE ICEBERG TABLE my_iceberg_table (
  catid    INT,
  catgroup STRING,
  catname  STRING,
  catdesc  STRING
)
  CATALOG = SNOWFLAKE
  EXTERNAL_VOLUME = SNOWFLAKE_MANAGED;
```

`SNOWFLAKE_MANAGED`は予約値であり、自分で作成するExternal Volumeの名前ではありません。
:::

利用可能なクラウドや制約は、次のドキュメントを確認してください。以下の画面記録には最終DDLが含まれないため、保存先の設定値自体の確認結果は掲載していません。

https://docs.snowflake.com/en/user-guide/tables-iceberg-internal-storage

## CoCo Desktopから移行を試してみる

### 1. 移行したい内容を伝える

まず、CoCo Desktopで移行用のプロジェクトを開き、次のように依頼します。

```text:CoCo Desktopへのプロンプト
RedshiftからSnowflakeへ移行をお願いいたします。なお、Snowflake storageを使用したIcebergテーブルにしてください。
```

![CoCo DesktopでRedshiftからの移行と、Snowflake storageを使うIcebergテーブルを指定する。](/images/snowflake-aim-redshift-iceberg/03-start.jpg)

「Redshiftから移行したい」だけでなく、テーブル形式と保存先も最初に伝えています。通常のSnowflakeテーブルにするのか、Icebergにするのかで、後続のDDLやデータ移行設定が変わるためです。

:::message alert
承認を省略する設定はAIM利用の前提ではありません。データの書き込みやリソース作成は、対象と影響を確認してから進めてください。
:::

### 2. Redshiftへの接続を設定する

続いて、移行元への接続情報を設定します。

![Redshiftの認証方式、ホスト、データベースを入力する画面。](/images/snowflake-aim-redshift-iceberg/04-source-connection.jpg)

画面にはIAM認証とStandard Authが表示され、Standard Authを選んでいます。ホスト欄の`your-cluster.region.redshift.amazonaws.com`は入力例です。実際には、自分のRedshift環境の接続先を指定します。

移行対象を読み取る権限と、ネットワーク到達性も確認します。Workerの実行環境には、Redshift用ODBCドライバーも必要です。公開ドキュメントでは、後述するODBC抽出とUNLOAD抽出の両方で前提条件になっています。

:::message alert
パスワードやトークンなどの認証情報は、記事やGitリポジトリへ記載しません。
:::

### 3. コード変換の結果を見る

コード変換が終わると、CoCo上に変換結果と成果物の場所が表示されます。

![コード変換の完了メッセージ。変換済みコードとSnowConvertレポートの保存先が表示される。](/images/snowflake-aim-redshift-iceberg/05-conversion.jpg)

この画面では、変換済みコードは`snowflake/`、レポートは`reports/SnowConvert/`に出力したと報告されています。EWIは0件と表示されました。EWIは、変換時のエラー・警告・課題を示すものです。

:::message
ここで確認しているのは、変換段階の結果です。EWIが0件でも、データ移行やプロシージャの動作確認が終わったわけではありません。画面に表示された処理時間も、このデモの報告値であり、移行全体の所要時間ではありません。
:::

### 4. ダッシュボードで進捗と依存関係を確認する

CoCoの会話だけでなく、Migration Dashboardでも状況を確認できます。

![コード変換が13件中13件完了し、デプロイ以降は未完了となっているダッシュボード。](/images/snowflake-aim-redshift-iceberg/06-dashboard.jpg)

この時点では、Conversionが13件中13件で完了しています。内訳はデータベース1件、スキーマ1件、テーブル8件、ビュー1件、プロシージャ2件です。一方、DeploymentやData migrationはまだ0件です。

工程ごとの状態が分かれているため、「変換できたが、まだデータは移していない」という状況を読み取れます。

オブジェクトを開くと、そのオブジェクトの進捗や依存関係も表示されます。

![salesテーブルの詳細画面。参照するビューとプロシージャがRequired byに表示される。](/images/snowflake-aim-redshift-iceberg/07-dependencies.jpg)

`dev.public.sales`の画面では、`Required by`に次の2オブジェクトが表示されています。

- `dev.public.v_event_sales`（ビュー）
- `dev.public.sp_refresh_sales_summary`（プロシージャ）

テーブルだけを移して終わりではなく、それを参照するSQLも移行対象として追えることが分かります。移行順序の検討や、テーブル定義を修正した際の確認に使える情報です。

### 5. 移行先とデータ移行の実行環境を選ぶ

次に、移行先のSnowflake接続とデータベースを指定します。

![Snowflake接続名と移行先データベースAIM_MIGRATION_DBを指定する画面。](/images/snowflake-aim-redshift-iceberg/08-target-connection.jpg)

このデモでは、接続名に`coco_desktop`、ターゲットDBに`AIM_MIGRATION_DB`を指定しています。これらはデモ環境の値です。自分の環境では、意図したアカウントとデータベースへ接続しているかを確認します。

続いて、OrchestratorとWorkerの実行場所を選びます。

![OrchestratorをLocal、Workerをローカルワーカーに設定する画面。](/images/snowflake-aim-redshift-iceberg/09-runtime.jpg)

Orchestratorはデータ移行・検証ワークフローを管理する役割です。Workerは移行元への接続やデータ抽出などを担当します。ここでは、どちらもローカルで動かす選択になっています。

画面にはSnowpark Container Services（SPCS）も表示されています。必ずSPCSを用意するというわけではなく、規模やネットワーク構成に応じて配置先を決めます。詳しくは次のドキュメントを参照してください。

https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/overview

### 6. ODBC抽出とIcebergを選ぶ

抽出方式と、ターゲットのテーブル形式を選びます。

![抽出方式でDirect read（ODBC）、テーブルタイプでIceberg（Redshift only）を選択したデータ移行の計画画面。](/images/snowflake-aim-redshift-iceberg/10-extraction.jpg)

この画面では、抽出方式に`Direct read (ODBC)`、テーブルタイプに`Iceberg (Redshift only)`を選んでいます。ODBC抽出は、WorkerがRedshiftから結果セットを取得する方式です。公開ドキュメントでは`regular`という抽出方式に対応します。

:::message alert
テーブルタイプの説明には「Snowflake管理のIcebergテーブル（Redshiftソースのみ）」と書かれています。Icebergを移行先に選べる範囲は移行元によって異なる点に注意してください。
:::

もう一方の`UNLOAD to S3`は、RedshiftがS3へファイルを書き出し、Snowflakeが外部ステージ経由でロードする方式です。画面の説明にもS3バケットとIAMロールが必要と書かれています。今回選んでいるのはUNLOADではありません。

抽出経路とIcebergの保存先は、別の設定です。UNLOADを使う場合でも、UNLOAD用S3とIcebergの保存先を同じものとして扱わないようにします。次のドキュメントでは、前提を整えられる場合はUNLOADを推奨しています。ODBC抽出は少量データなどに向く選択肢です。

https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/migrate-redshift

### 7. 移行結果の行数を確認する

データ移行後の画面には、Icebergテーブルの行数確認結果が表示されました。

![6テーブルの行数がソースと一致したと報告されている結果画面。](/images/snowflake-aim-redshift-iceberg/11-row-counts.jpg)

画面に表示された結果をまとめると、次のとおりです。

| テーブル | 行数 | 画面上のステータス |
| --- | ---: | --- |
| CATEGORY | 11 | 一致 |
| DATE | 365 | 一致 |
| EVENT | 541 | 一致 |
| SALES | 998 | 一致 |
| SALES_SUMMARY | 541 | 一致 |
| VENUE | 202 | 一致 |

6テーブルについて、ソースと行数が一致したと報告されています。

:::message alert
行数が同じでも、個々の値や重複の状態まで一致するとは限りません。この画面を、全行比較が完了した証拠としては扱いません。
:::

:::details 変換時は8テーブル、ここでは6テーブルになっている理由
変換時のダッシュボードには8テーブル、行数確認には6テーブルが表示されています。変換対象の一覧には内部テーブルも含まれていましたが、この画面だけでは差分の理由をすべて特定できません。初期対象すべての移行完了ではなく、ここに示された6テーブルの結果として紹介します。
:::

### 8. SnowsightでIcebergテーブルとデータを見る

最後に、Snowsightのデータベースエクスプローラーから移行先を開きます。

![AIM_MIGRATION_DB.PUBLIC.CATEGORYがIcebergテーブルとして表示され、11行のデータを確認できる。](/images/snowflake-aim-redshift-iceberg/12-snowsight.jpg)

`AIM_MIGRATION_DB.PUBLIC.CATEGORY`には「Icebergテーブル」と表示されています。データプレビューには11行が表示され、`CATID`、`CATGROUP`、`CATNAME`、`CATDESC`の値を確認できます。

CoCoの完了メッセージだけでなく、移行先の画面でもテーブル形式とデータを確認できました。

:::message
この画面にはストレージ設定は表示されていません。Snowflake storageを使っていることは、別途テーブルのDDLやメタデータで確認する項目です。`SHOW ICEBERG TABLES`や`GET_DDL`で確認します。
:::

## 試してみて分かったことと、本番で追加する確認

今回は、対話とダッシュボードを使い分けて作業を進めました。CoCoで接続や移行方式を指定し、ダッシュボードではオブジェクト単位の進捗と依存関係を確認できました。

一方、このデモで確認した範囲と、本番の移行完了条件は分ける必要があります。

### DDLとデータ移行設定の両方を確認する

Icebergを選ぶときは、作成されたテーブル定義とデータ移行設定が一致しているかを確認します。

:::message alert
データ移行設定だけをIcebergにしても、既存の通常テーブルが自動で置き換わるとは考えないでください。
:::

Snowflake storageを指定した場合は、最終的なカタログと保存先の設定も確認します。通常テーブルとIcebergの型マッピングを比較し、変換後の列型も確認します。

### 行数だけでなく、データと業務ロジックを検証する

AIMには、次の3段階の検証レベルがあります。

| レベル | 内容 |
| --- | --- |
| L1 | スキーマ比較 |
| L2 | 件数や集計指標の比較 |
| L3 | 行レベル比較 |

今回掲載した行数確認だけで、これらすべての検証を実施したとはいえません。

本番移行では、重要なテーブルの値、NULL、重複、集計結果などを確認します。ビューやプロシージャも、業務上の入力条件を使って結果を比較します。テーブルのデータ検証と、業務ロジックの動作検証は別の作業です。

また、データ移行で対応する型と、行レベル検証で対応する型は同じとは限りません。次のドキュメントで対象範囲を確認してください。

https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/validate-redshift

### 比較時点と切り替え条件を決める

データを抽出した後にRedshift側が更新されると、比較時点の違いによって不一致が生じます。検証が終わるまでは移行元への接続を維持し、比較するデータの時点をそろえます。

本番切り替えには、ETLやBIツールの接続先変更、最終差分の反映、切り戻し手順も必要です。移行用の権限やリソースを見直し、不要になった実行環境を停止するところまで計画します。

## まとめ

Snowflake AIMは、コード変換だけでなく、依存関係や進捗を管理しながら移行を進めるための仕組みです。

今回はCoCo DesktopからRedshiftの移行を依頼し、Icebergを移行先として指定する流れを紹介しました。画面では、ローカルのOrchestratorとWorker、ODBC抽出を選んでいます。その後、6テーブルの行数一致という報告と、SnowsightでのCATEGORYテーブルのデータ表示を確認できました。

これから試す場合は、小さな範囲で接続、変換、ロード、検証を一巡させると、確認すべき項目を整理しやすくなります。AIMが示す結果を見ながら、移行先の定義とデータも確認して進めてみてください。

## 参考資料

- [Snowflake AIMの発表記事](https://www.snowflake.com/en/blog/snowflake-aim-enterprise-migration-modernization/)
- [Snowflake AIM Agent for Data Warehouses](https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/overview)
- [AIM Migration Agentの技術解説](https://www.snowflake.com/en/blog/engineering/snowflake-aim-migration-agent/)
- [Redshiftからのデータ移行](https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/migrate-redshift)
- [Data Migration & Validationの概要](https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/overview)
- [Redshiftからのデータ検証](https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/validate-redshift)
- [Snowflake storage for Apache Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-internal-storage)