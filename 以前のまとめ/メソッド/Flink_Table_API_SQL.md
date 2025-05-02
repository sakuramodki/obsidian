---
tags: メソッド, ApacheFlink, API, ストリーム処理, SQL, 宣言的プログラミング
aliases: Flink Table API, Flink SQL
---

# メソッド: Flink Table API / SQL

## 概要
Flink Table APIおよびSQLは、[[Apache_Flink]]における高レベルな宣言的APIです。データストリームやバッチデータをリレーショナルデータベースのテーブルのように扱い、SQLライクな式や標準SQLクエリを用いて処理を記述できます。

## 前提となるモデル/理論(L1)とコンテキスト
* **L1モデル:** [[ストリーム処理(データ)]], [[ステートフルコンピューティング]], [[Boundedストリーム vs Unboundedストリーム]], リレーショナルモデル
* **コンテキスト:**
    * **目的:** データストリームやバッチデータに対して、結合、集計、フィルタリング、ウィンドウ処理などを、SQLやそれに近い直感的な方法で記述したい。
    * **状況:** 開発者がSQLに慣れている。データ分析的なタスクが多い。[[Flink_DataStream_API]]ほどの低レベルな制御は不要で、より簡潔に処理を記述したい。異種データソース間の結合などを容易に行いたい。
    * **技術:** Java, Scala, Pythonバインディング、またはSQL Clientを通じて利用可能。

## 具体的な手順・考慮事項
1.  **`TableEnvironment`の取得:** Table API/SQLの操作の起点となる実行環境オブジェクトを取得します (StreamExecutionEnvironmentから作成可能)。
2.  **テーブルソース(Source Table)の登録:** 外部システム（Kafka, DB, ファイルなど）を仮想的なテーブルとして`TableEnvironment`に登録します。CREATE TABLE DDL (SQL) やコネクタAPI (Table API) を使用します。
3.  **クエリの実行:**
    * **Table API:** `Table`オブジェクトに対して`select()`, `filter()`, `groupBy()`, `window()`, `join()`などのメソッドをチェーンさせてクエリを構築します。
    * **SQL:** `tableEnvironment.sqlQuery()`や`tableEnvironment.executeSql()`メソッドを用いて、標準SQL (拡張を含む) でクエリを記述します。
4.  **テーブルシンク(Sink Table)の登録と出力:** クエリ結果（`Table`オブジェクト）を外部システムに出力するために、シンクテーブルを登録し、`executeInsert()` (Table API/SQL) などで書き込みます。
5.  **DataStreamとの相互変換:** 必要に応じて`TableEnvironment.toDataStream()`や`fromDataStream()`メソッドを用いて、[[Flink_DataStream_API]]とTable API/SQLの間でデータを変換できます。
6.  **考慮事項:**
    * **動的テーブル (Dynamic Tables):** ストリームデータを表現するために、時間とともに変化する「動的テーブル」という概念を内部的に使用します。クエリ結果も動的テーブルとなります。
    * **時間属性:** [[ストリーム処理における時間概念]] (イベント時間/処理時間) をテーブルスキーマ内で定義し、時間ベースの操作（ウィンドウ集計など）に利用します。
    * **組み込み関数とUDF:** 豊富な組み込み関数に加え、カスタムロジックを実装するためのユーザー定義関数（UDF, UDAF, UDTF）を作成・登録できます。
    * **最適化:** Flinkは宣言的なクエリを解析し、内部的に最適化された実行プラン（DataStreamジョブ）を生成します。
    * **SQLの標準準拠と拡張:** 標準SQLに近い構文ですが、ストリーム処理特有の機能（ウィンドウTVFなど）に関する拡張が含まれます。