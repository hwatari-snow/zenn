---
title: "Snowflake AIMを使ってRedshiftからSnowflake-managed Icebergテーブルへの移行を試してみた"
emoji: "❄️"
type: "tech"
topics: ["snowflake", "redshift", "iceberg", "aws", "datamigration"]
published: true
publication_name: "snowflakejp"
---

:::message
著者はSnowflakeに所属しておりますが、本記事は個人の見解であり、所属する組織の公式見解ではありません。
:::

## はじめに

こんにちは。Snowflakeでソリューションエンジニアをしている渡利です。

データウェアハウスの移行では、データをコピーするだけでは作業が終わりません。SQLの方言差分を直し、ビューやストアドプロシージャの依存関係を確認します。移行先で同じ結果が得られるかを検証する作業も必要です。

今回は、こうした移行作業を支援する **Snowflake AIM** を紹介します。後半では、CoCo DesktopからAmazon Redshiftの移行を試した画面を使い、操作の流れを追っていきます。移行先には、Snowflake storageを使用するIcebergテーブルを指定しました。

この記事は、移行を担当するエンジニア向けです。AIMの概要に続いて、接続設定、コード変換、データ移行の設定、移行後の確認を紹介します。

:::message
本記事の内容は、Snowflake World Tour Tokyoのセッション「**What's New: AI時代のアナリティクス最新情報**」でもお話ししています。オンデマンド配信でご覧いただけますので、動画で追いたい方は下記のリンクからご登録をいただきこちらもあわせてご覧ください！
:::

https://www.snowflake.com/ja/world-tour/tokyo/

:::message alert
製品仕様の説明は2026年9月18日時点の公開ドキュメントに基づきます。
:::

## Snowflake AIMとは

こちらが、Snowflakeへの移行を劇的にシンプルにする、「Snowflake AIM」の全体像です。
Snowflake AIMがコードやワークフロー、依存関係を自動で分析し、明確な移行プランを作成して実行してくれます。

![Snowflake AIMの全体像。仮想化、データウェアハウス移行、Sparkワークロードのモダナイゼーション。](/images/snowflake-aim-redshift-iceberg/01-aim-overview.jpg)


図には三つのブロックが並んでいますが、アプローチとしては大きく2つに分かれます。

1つ目がバーチャライゼーション（仮想化）です。Teradata専用のサービスで、接続先を切り替えるだけで、既存のアプリケーションやワークロードをそのままSnowflake上で実行できます。Snowflakeが買収したDatometry社の技術がベースです。

2つ目がモダナイゼーションです。データウェアハウス、ETLのプロセス、Sparkワークロードを、Snowflakeネイティブなアーキテクチャへ自動で移行・変換します。

今回のブログで扱うのは、2つ目の**Snowflake AIM Agent for Data Warehouses**です。Snowflake CoCo上で対話しながら、接続、コード抽出、変換、アセスメント、デプロイ、データ移行、検証を進めます。

対応する移行元は幅広く、工程ごとに対応範囲が定められています。コード抽出まで対応するのはSQL Server、Redshift、PostgreSQLです。決定論的なコード変換はこれらに加えてTeradata、Oracle、Azure Synapse、Google BigQuery、Greenplum、Netezza、Spark SQL、Databricks SQL、Vertica、Hive、IBM DB2などに対応します。

全体の進捗は標準で用意されているレポートやダッシュボードでも可視化され、人とAIが移行状態を見ながら進められます。

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
| 移行先DB | Snowflakeの`AIM_MIGRATION_DB` |
| テーブル形式 | Iceberg |
| 保存先の要件 | Snowflake storage |
| Orchestrator | Local |
| Worker | ローカルワーカー |
| 抽出方式 | Direct read（ODBC） |

:::message
ここでは、AIM Agentを利用できるCoCo Desktopと、移行元・移行先へ接続できる環境を前提にしています。新規導入からの全手順ではなく、移行操作の紹介です。
:::


## Snowflake AIMプラグインをCoCoに導入する

AIM Agentは、CoCo CLIにはバンドルされています。今回のようにCoCo Desktopから使う場合は、`snowflake-migration`プラグインを導入しておきます。

SnowConvertやODBCドライバなどの依存ツールは、初回実行時に自動でインストールされます。前提条件とトラブルシュートは次のドキュメントを参照してください。

https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/troubleshooting

## CoCo Desktopから移行を試してみる

### 1. 移行したい内容を伝える

まず、CoCo Desktopで移行用のプロジェクトを開き、次のように依頼します。

```text:CoCo Desktopへのプロンプト
RedshiftからSnowflakeへ移行をお願いいたします。なお、Snowflake storageを使用したIcebergテーブルにしてください。
```

![CoCo DesktopでRedshiftからの移行と、Snowflake storageを使うIcebergテーブルを指定する。](/images/snowflake-aim-redshift-iceberg/03-start.jpg)

「Redshiftから移行したい」だけでなく、テーブル形式と保存先も最初に伝えています。通常のSnowflakeテーブルにするのか、Icebergにするのかで、後続のDDLやデータ移行設定が変わるためです。

:::message alert
データの書き込みやリソース作成は、対象と影響を確認してから進めてください。
:::

### 2. 接続設定は、AIMの質問に答えるだけで済む

AIMに依頼すると、環境を分析したうえで必要な作業を順に求めてきます。最初のステップは移行元Redshiftとの接続設定です。

![Redshiftの認証方式、ホスト、データベースを入力する画面。](/images/snowflake-aim-redshift-iceberg/04-source-connection.jpg)

ここで注目したいのは、設定ファイルの書式を調べる必要がない点です。必要な項目はAIMがポップアップで順に聞いてくるので、答えていくだけで設定が終わります。画面にはIAM認証とStandard Authが表示され、Standard Authを選んでいます。ホスト欄の`your-cluster.region.redshift.amazonaws.com`は入力例です。実際には、自分のRedshift環境の接続先を指定します。

対話で完結するとはいえ、人が用意すべき前提はあります。（移行対象を読み取る権限、Redshiftへのネットワーク到達性など）



### 3. 移行対象を対話で絞り込み、変換結果はファイルとして残る

接続できると、次は「Redshiftの何をどのように移行するか」をAIMが尋ねてきます。今回はデータベース1件、テーブル6件、プロシージャ2件、ビュー1件を対象に指定しました。

コード変換が終わると、結果と成果物の場所が表示されます。

![コード変換の完了メッセージ。変換済みコードとSnowConvertレポートの保存先が表示される。](/images/snowflake-aim-redshift-iceberg/05-conversion.jpg)

変換結果が会話の中で消えずにファイルとして残るのも良いところです。変換済みコードは`snowflake/`、レポートは`reports/SnowConvert/`に出力されています。生成されたSQLをそのまま開いてレビューでき、レポートでEWI（変換時のエラー・警告・課題）の内訳も確認できます。このデモではEWIは0件でした。


### 4. 進捗の可視化が最初から用意されている

AIMは、ブラウザで開けるアセスメントのダッシュボードを自動生成します。従来の移行プロジェクトでは、「今どこまで進んだのか」「どのオブジェクトが完了し、どこに課題があるのか」を集計する作業自体に手間がかかります。

:::message
**標準でアセスメントのダッシュボードが用意されていて、そのまま移行のタスク管理ができるのは最高です。** 進捗表を自作しなくても、オブジェクト単位の状況がひと目で分かります。
:::

![コード変換が13件中13件完了し、デプロイ以降は未完了となっているダッシュボード。](/images/snowflake-aim-redshift-iceberg/06-dashboard.jpg)

この時点では、Conversionが13件中13件で完了しています。内訳はデータベース1件、スキーマ1件、テーブル8件、ビュー1件、プロシージャ2件です。一方、DeploymentやData migrationはまだ0件です。

工程が分かれて表示されるため、「変換は終わったが、まだデータは移していない」という状態をそのまま読み取れます。会話の完了メッセージだけを見て全体が終わったと誤解せずに済む、という意味でも有用です。

さらに、オブジェクト単位で進捗と依存関係も追えます。

![salesテーブルの詳細画面。参照するビューとプロシージャがRequired byに表示される。](/images/snowflake-aim-redshift-iceberg/07-dependencies.jpg)

`dev.public.sales`の画面では、`Required by`に次の2オブジェクトが表示されています。

- `dev.public.v_event_sales`（ビュー）
- `dev.public.sp_refresh_sales_summary`（プロシージャ）

テーブルを移すだけでなく、それを参照するSQLも追跡対象になっています。AIMはこの依存関係をもとに移行順序（Wave）を組むため、大規模で複雑なDBでも「ビューより先に参照先テーブル」という順序を人が手で並べ替える必要がありません。テーブル定義を直したときに影響範囲を確認する用途にも使えます。

### 5. 実行環境はローカルとSPCSから選べる

次に、移行先のSnowflake接続とデータベースを指定します。

![Snowflake接続名と移行先データベースAIM_MIGRATION_DBを指定する画面。](/images/snowflake-aim-redshift-iceberg/08-target-connection.jpg)

このデモでは、接続名に`coco_desktop`、ターゲットDBに`AIM_MIGRATION_DB`を指定しています。これらはデモ環境の値です。自分の環境では、意図したアカウントとデータベースへ接続しているかを確認します。

続いて、OrchestratorとWorkerの実行場所を選びます。

![OrchestratorをLocal、Workerをローカルワーカーに設定する画面。](/images/snowflake-aim-redshift-iceberg/09-runtime.jpg)

Orchestratorはデータ移行・検証ワークフローを管理する役割です。Workerは移行元への接続やデータ抽出などを担当します。今回はデモ用でデータ量が小さいため、どちらもローカルを選びました。

注目したいのは、データ量が増えても実行環境を差し替えられることです。画面にはSnowpark Container Services（SPCS）も表示されており、大規模データではSnowflakeのマネージドなコンテナ上でOrchestratorとWorkerを動かせます。ローカルで小さく試し、本番規模ではSPCSへ寄せるという進め方が取れます。規模やネットワーク構成に応じて配置先を決めてください。詳しくは次のドキュメントを参照してください。

https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/overview

### 6. 抽出経路とテーブル形式を決める

抽出方式と、移行先のテーブル形式を選びます。

![抽出方式でDirect read（ODBC）、テーブルタイプでIceberg（Redshift only）を選択したデータ移行の計画画面。](/images/snowflake-aim-redshift-iceberg/10-extraction.jpg)

今回は抽出方式に`Direct read (ODBC)`、テーブルタイプに`Iceberg`を選びました。S3を経由せず、WorkerがODBC接続でRedshiftから結果セットを直接取得する方式です。公開ドキュメントでは`regular`という抽出方式に対応します。

この設定を入れると、AIMは裏側で次を全テーブルに対して自動実行します。

1. 移行先にIcebergテーブルを作成する
2. ODBC接続でRedshiftからデータを取得する
3. 取得したデータをIcebergテーブルへ投入する

テーブル定義の作成からデータ投入までを一括で面倒を見てくれるため、ネイティブテーブルではなくIcebergで移行したい場合も、選択肢を切り替えるだけで済みます。


今回はODBC抽出を選んでいるため、S3は経由していません。もう一方の`UNLOAD to S3`は、RedshiftがS3へファイルを書き出し、Snowflakeが外部ステージ経由でロードする抽出方式です。S3バケットとIAMロールを用意する必要がありますが、公開ドキュメントでは前提を整えられる場合はUNLOADが推奨されています。ODBC抽出は少量データなどに向く選択肢です。

ここで押さえておきたいのは、**抽出経路（ODBC / UNLOAD）とIcebergの保存先（Snowflake storage / 外部ボリューム）は別の設定**だという点です。UNLOAD用のS3バケットと、Icebergのデータファイルを置く場所は同じものではありません。

保存先に外部ボリュームを使いたい場合も、CoCoはAWS CLIを実行できるので、IAMロールとS3バケットの作成からExternal Volumeの定義まで、同じセッションの中で進められます。詳細は下記をご参考にしてください。

https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/migrate-redshift

### 7. データ移行と同時に検証まで走る

データ移行が終わると、行数の確認結果まで続けて表示されます。移行プロジェクトで怖いのは「データが欠損していないか」なので、移行と検証が同じフローに載っているのは実務上ありがたい設計です。

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
:::message
データが問題なく移行できているか、件数だけでなく欠損や型の不一致による不備がないか細かく検証してくれる点も最高です。
:::


### 8. 移行先の画面でもテーブル形式を確かめる

最後に、念のためSnowsightのデータベースエクスプローラーから移行先を開いて確認します。

![AIM_MIGRATION_DB.PUBLIC.CATEGORYがIcebergテーブルとして表示され、11行のデータを確認できる。](/images/snowflake-aim-redshift-iceberg/12-snowsight.jpg)

`AIM_MIGRATION_DB.PUBLIC.CATEGORY`には「Icebergテーブル」と表示されています。データプレビューには11行が表示され、`CATID`、`CATGROUP`、`CATNAME`、`CATDESC`の値を確認できます。

CoCoの完了メッセージだけでなく、移行先の画面でもテーブル形式とデータを確認できました。


## 試してみて良かったポイント

### 対話で指定し、ダッシュボードで把握できる

接続、移行対象の選定、抽出方式、テーブル形式といった判断はすべてCoCoとの対話で指定でき、進捗と依存関係はダッシュボードで俯瞰できます。判断は対話、状況確認はダッシュボードと役割が分かれているため、進捗表を別途作らずに移行作業を進められます。

### 移行先にIcebergを選べる

抽出方式とテーブル形式が独立した選択肢になっているので、「RedshiftからODBCで抽出し、Snowflake storageのIcebergテーブルへ入れる」という組み合わせを選択だけで実現できました。テーブル作成からデータ投入までAIMがまとめて面倒を見てくれるため、オープンなレイクハウス構成を前提とした移行も検討しやすくなります。

### 検証が段階的に用意されている

行数一致は自動で表示されましたが、AIMの検証はそれだけではありません。次の3段階が用意されています。

| レベル | 内容 |
| --- | --- |
| L1 | スキーマ比較 |
| L2 | 件数や集計指標の比較 |
| L3 | 行レベル比較 |

「まずL1とL2で全体を確認し、重要なテーブルだけL3で突き合わせる」といった使い分けができます。移行で一番気になるデータの整合性確認を、ツール側の仕組みに載せられるのは大きな安心材料です。対応する型の範囲は次のドキュメントで確認できます。

https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/validate-redshift

### プロシージャとビューまで含めて回せる

AIMは依存関係を解析して移行順序（Wave）を組むため、テーブルとそれを参照するビューやプロシージャを正しい順番でまとめて移行できます。プロシージャや関数は移行元の出力をベースラインとして突き合わせるテストの仕組みもあり、一度行った修正は再利用可能なルールとしてプロジェクト全体へ展開できます。オブジェクト数が多い移行ほど後半が楽になる設計です。

## まとめ

Snowflake AIMは、コード変換だけでなく、依存関係や進捗を管理しながら移行を進めるための仕組みです。

今回はCoCo DesktopからRedshiftの移行を依頼し、Icebergを移行先として指定する流れを紹介しました。画面では、ローカルのOrchestratorとWorker、ODBC抽出を選んでいます。その後、6テーブルの行数一致という報告と、SnowsightでのCATEGORYテーブルのデータ表示を確認できました。

移行できるのはテーブルだけではありません。今回の対象にもストアドプロシージャとビューが含まれており、AIMはオブジェクト間の依存関係を踏まえて移行順序を組みます。大規模で複雑なデータベースほど、この依存関係の把握と進捗の可視化が効いてくる部分です。

これから試す場合は、小さな範囲で接続、変換、ロード、検証を一巡させると、確認すべき項目を整理しやすくなります。AIMが示す結果を見ながら、移行先の定義とデータも確認して進めてみてください！

## 参考資料

- [Snowflake AIMの発表記事](https://www.snowflake.com/en/blog/snowflake-aim-enterprise-migration-modernization/)
- [Snowflake AIM Agent for Data Warehouses](https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/overview)
- [AIM Migration Agentの技術解説](https://www.snowflake.com/en/blog/engineering/snowflake-aim-migration-agent/)
- [Redshiftからのデータ移行](https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/migrate-redshift)
- [Data Migration & Validationの概要](https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/overview)
- [Redshiftからのデータ検証](https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/validate-redshift)
- [Snowflake storage for Apache Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-internal-storage)
- [Snowflake World Tour Tokyo（セッションのオンデマンド配信）](https://www.snowflake.com/ja/world-tour/tokyo/)

## 関連記事

Snowflake storageを使うIcebergテーブル自体については、こちらの庄司さんの記事で詳しく扱っています！併せて最高の機能ですので読んでいただけると幸いです。

https://zenn.dev/snowflakejp/articles/snowflake_iceberg_open_sharing