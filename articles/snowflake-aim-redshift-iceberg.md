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
製品仕様の説明は2026年9月18日時点の公開ドキュメントに基づきます。
:::

## Snowflake AIMとは

こちらが、Snowflakeへの移行を劇的にシンプルにする、「Snowflake AIM」の全体像です。
Snowflake AIMがコードやワークフロー、依存関係を自動で分析し、明確な移行プランを作成して実行してくれます。

![Snowflake AIMの全体像。仮想化、データウェアハウス移行、Sparkワークロードのモダナイゼーション。](/images/snowflake-aim-redshift-iceberg/01-aim-overview.jpg)


AIMは2つのアプローチをサポートしています。

1つ目が、バーチャライゼーション、仮想化です。こちらはTeradata専用のサービスになっておりますが接続先を切り替えるだけで、既存のアプリケーションやワークロードをそのままSnowflake上で実行できます。Snowflakeが買収したDatometry社の技術がベースです。

2つ目が、モダナイゼーションです。データウェアハウス、ETLのプロセス、Sparkワークロードを、Snowflakeネイティブなアーキテクチャへ自動で移行・変換します。

今回のブログで扱うのは この２つ目の**Snowflake AIM Agent for Data Warehouses** です。Snowflake CoCo上で対話しながら、接続、コード抽出、変換、アセスメント、デプロイ、データ移行、検証を進めます。

対応可能なサービスは、SQL Server, Redshift, Teradata, Oracle, Azure Synapse, Google BigQuery, Greenplum, Netezza, Spark SQL, Databricks SQL, Vertica, Hive, IBM DB2など多様です。
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
ここでは、AIM Agentを利用できるCoCo Desktopと、移行元・移行先へ接続できる環境を前提にしています。新規導入からの全手順ではなく、移行操作の紹介です。画面に使用バージョンは写っていません。表示や選択肢は、手元の環境と異なる可能性があります。
:::

なお、Snowflake StrageのIceebrgテーブルに関しては、こちらのブログを参考にしてください。
https://zenn.dev/snowflakejp/articles/snowflake_iceberg_open_sharing?redirected=1

## Snowflake AIMプラグインをCoCoに導入する

AIM Agentは、CoCo CLIにはバンドルされています。今回のようにCoCo Desktopから使う場合は、`snowflake-migration`プラグインを導入しておきます（このデモ環境では`Snowflake-Labs/cortex-code-migrations`から導入したものを使用しています）。

SnowConvertやODBCドライバなどの依存ツールは、初回実行時に自動でインストールされます。前提条件とトラブルシュートは次のドキュメントを参照してください。

https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/troubleshooting

## CoCo Desktopから移行を試してみる

### Snowflake AIMのインストール
前提として、こちらの手順に従って、Snowflake AIMのプラグインをこの手順でCoCoにインストールしてください。


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

### 2. 接続設定は、AIMの質問に答えるだけで済む

AIMに依頼すると、環境を分析したうえで必要な作業を順に求めてきます。最初のステップは移行元Redshiftとの接続設定です。

![Redshiftの認証方式、ホスト、データベースを入力する画面。](/images/snowflake-aim-redshift-iceberg/04-source-connection.jpg)

ここで注目したいのは、設定ファイルの書式を調べる必要がない点です。何が必要かをAIMがポップアップで聞いてくるので、こちらは答えていくだけで完了します。画面にはIAM認証とStandard Authが表示され、Standard Authを選んでいます。ホスト欄の`your-cluster.region.redshift.amazonaws.com`は入力例です。実際には、自分のRedshift環境の接続先を指定します。

対話で完結するとはいえ、人が用意すべき前提はあります。移行対象を読み取る権限、Redshiftへのネットワーク到達性、そしてWorker実行環境のRedshift用ODBCドライバーです。公開ドキュメントでは、後述するODBC抽出とUNLOAD抽出の両方でODBCドライバーが前提条件になっています。

:::message alert
パスワードやトークンなどの認証情報は、記事やGitリポジトリへ記載しません。
:::

### 3. 移行対象を対話で絞り込み、変換結果はファイルとして残る

接続できると、次は「Redshiftの何をどのように移行するか」をAIMが尋ねてきます。今回はデータベース1件、テーブル6件、プロシージャ2件、ビュー1件を対象に指定しました。

コード変換が終わると、結果と成果物の場所が表示されます。

![コード変換の完了メッセージ。変換済みコードとSnowConvertレポートの保存先が表示される。](/images/snowflake-aim-redshift-iceberg/05-conversion.jpg)

ここでのポイントは、変換結果が会話の中で消えずにファイルとして残ることです。変換済みコードは`snowflake/`、レポートは`reports/SnowConvert/`に出力されています。生成されたSQLをそのまま開いてレビューでき、レポートでEWI（変換時のエラー・警告・課題）の内訳も確認できます。このデモではEWIは0件でした。

:::message
ここで確認しているのは、変換段階の結果です。EWIが0件でも、データ移行やプロシージャの動作確認が終わったわけではありません。画面に表示された処理時間も、このデモの報告値であり、移行全体の所要時間ではありません。
:::

### 4. 進捗の可視化が最初から用意されている

AIMは、ブラウザで開けるダッシュボードを自動生成します。個人的にはここが一番うれしいポイントでした。従来の移行プロジェクトでは、「今どこまで進んだのか」「どのオブジェクトが完了し、どこに課題があるのか」を集計する作業自体に手間がかかります。AIMはそれを成果物として自動で出してくれるので、進捗管理の負荷が下がります。

![コード変換が13件中13件完了し、デプロイ以降は未完了となっているダッシュボード。](/images/snowflake-aim-redshift-iceberg/06-dashboard.jpg)

この時点では、Conversionが13件中13件で完了しています。内訳はデータベース1件、スキーマ1件、テーブル8件、ビュー1件、プロシージャ2件です。一方、DeploymentやData migrationはまだ0件です。

工程が分かれて表示されるため、「変換は終わったが、まだデータは移していない」という状態をそのまま読み取れます。会話の完了メッセージだけを見て全体が終わったと誤解せずに済む、という意味でも有用です。

さらに、オブジェクト単位で進捗と依存関係も追えます。

![salesテーブルの詳細画面。参照するビューとプロシージャがRequired byに表示される。](/images/snowflake-aim-redshift-iceberg/07-dependencies.jpg)

`dev.public.sales`の画面では、`Required by`に次の2オブジェクトが表示されています。

- `dev.public.v_event_sales`（ビュー）
- `dev.public.sp_refresh_sales_summary`（プロシージャ）

テーブルを移すだけでなく、それを参照するSQLも追跡対象になっている点がポイントです。AIMはこの依存関係をもとに移行順序（Wave）を組むため、大規模で複雑なDBでも「ビューより先に参照先テーブル」という順序を人が手で並べ替える必要がありません。テーブル定義を直したときに影響範囲を確認する用途にも使えます。

### 5. 実行環境はローカルとSPCSから選べる

次に、移行先のSnowflake接続とデータベースを指定します。

![Snowflake接続名と移行先データベースAIM_MIGRATION_DBを指定する画面。](/images/snowflake-aim-redshift-iceberg/08-target-connection.jpg)

このデモでは、接続名に`coco_desktop`、ターゲットDBに`AIM_MIGRATION_DB`を指定しています。これらはデモ環境の値です。自分の環境では、意図したアカウントとデータベースへ接続しているかを確認します。

続いて、OrchestratorとWorkerの実行場所を選びます。

![OrchestratorをLocal、Workerをローカルワーカーに設定する画面。](/images/snowflake-aim-redshift-iceberg/09-runtime.jpg)

Orchestratorはデータ移行・検証ワークフローを管理する役割です。Workerは移行元への接続やデータ抽出などを担当します。今回はデモ用でデータ量が小さいため、どちらもローカルを選びました。

ここでのポイントは、データ量が増えても実行環境を差し替えられることです。画面にはSnowpark Container Services（SPCS）も表示されており、大規模データではSnowflakeのマネージドなコンテナ上でOrchestratorとWorkerを動かせます。ローカルで小さく試し、本番規模ではSPCSへ寄せるという進め方が取れます。規模やネットワーク構成に応じて配置先を決めてください。詳しくは次のドキュメントを参照してください。

https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/overview

### 6. 抽出経路とテーブル形式を決める

ここが今回のデモで一番重要な設定です。抽出方式と、移行先のテーブル形式を選びます。

![抽出方式でDirect read（ODBC）、テーブルタイプでIceberg（Redshift only）を選択したデータ移行の計画画面。](/images/snowflake-aim-redshift-iceberg/10-extraction.jpg)

今回は抽出方式に`Direct read (ODBC)`、テーブルタイプに`Iceberg (Redshift only)`を選びました。S3を経由せず、WorkerがODBC接続でRedshiftから結果セットを直接取得する方式です。公開ドキュメントでは`regular`という抽出方式に対応します。

この設定を入れると、AIMは裏側で次を全テーブルに対して自動実行します。

1. 移行先にIcebergテーブルを作成する
2. ODBC接続でRedshiftからデータを取得する
3. 取得したデータをIcebergテーブルへ投入する

テーブル定義の作成からデータ投入までを一括で面倒を見てくれるため、ネイティブテーブルではなくIcebergで移行したい場合も、選択肢を切り替えるだけで済みます。

:::message alert
テーブルタイプの説明には「Snowflake管理のIcebergテーブル（Redshiftソースのみ）」と書かれています。Icebergを移行先に選べる範囲は移行元によって異なる点に注意してください。
:::

もう一方の`UNLOAD to S3`は、RedshiftがS3へファイルを書き出し、Snowflakeが外部ステージ経由でロードする方式です。画面の説明にもS3バケットとIAMロールが必要と書かれています。今回選んでいるのはUNLOADではありません。

抽出経路とIcebergの保存先は、別の設定です。UNLOADを使う場合でも、UNLOAD用S3とIcebergの保存先を同じものとして扱わないようにします。次のドキュメントでは、前提を整えられる場合はUNLOADを推奨しています。ODBC抽出は少量データなどに向く選択肢です。

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

:::message alert
ただし、自動で走るのは検証の一部です。行数が同じでも、個々の値や重複の状態まで一致するとは限りません。この画面を、全行比較が完了した証拠としては扱いません。後述する検証レベルのどこまでを実施するかは、人が決める必要があります。
:::

:::details 変換時は8テーブル、ここでは6テーブルになっている理由
移行対象として指定したのはテーブル6件ですが、変換時のダッシュボードには8テーブルが表示されています。変換対象の一覧には内部テーブルも含まれていたためです。この画面だけでは差分の理由をすべて特定できないため、ここに示された6テーブルの結果として紹介します。
:::

### 8. 移行先の画面でもテーブル形式を確かめる

最後に、AIMの完了メッセージを鵜呑みにせず、Snowsightのデータベースエクスプローラーから移行先を開いて確認します。

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

移行できるのはテーブルだけではありません。今回の対象にもストアドプロシージャとビューが含まれており、AIMはオブジェクト間の依存関係を踏まえて移行順序を組みます。大規模で複雑なデータベースほど、この依存関係の把握と進捗の可視化が効いてくる部分です。

これから試す場合は、小さな範囲で接続、変換、ロード、検証を一巡させると、確認すべき項目を整理しやすくなります。AIMが示す結果を見ながら、移行先の定義とデータも確認して進めてみてください。

## 参考資料

- [Snowflake AIMの発表記事](https://www.snowflake.com/en/blog/snowflake-aim-enterprise-migration-modernization/)
- [Snowflake AIM Agent for Data Warehouses](https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/overview)
- [AIM Migration Agentの技術解説](https://www.snowflake.com/en/blog/engineering/snowflake-aim-migration-agent/)
- [Redshiftからのデータ移行](https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/migrate-redshift)
- [Data Migration & Validationの概要](https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/overview)
- [Redshiftからのデータ検証](https://docs.snowflake.com/en/migrations/aim-for-datawarehouses/data-migration-validation/validate-redshift)
- [Snowflake storage for Apache Iceberg tables](https://docs.snowflake.com/en/user-guide/tables-iceberg-internal-storage)