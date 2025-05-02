---
tags: メソッド, ApacheFlink, API, ストリーム処理, プログラミング
aliases: Flink DataStream API
---

# メソッド: Flink DataStream API

## 概要
Flink DataStream APIは、[[Apache_Flink]]におけるコアAPIの一つで、データストリームに対する[[ステートフルコンピューティング]]をプログラム的に（命令的に）記述するための低レベル寄りなAPIです。JavaやScalaで利用できます。

## 前提となるモデル/理論(L1)とコンテキスト
* **L1モデル:** [[ストリーム処理(データ)]], [[ステートフルコンピューティング]], [[Boundedストリーム vs Unboundedストリーム]]
* **コンテキスト:**
    * **目的:** データストリームに対して、map, filter, join, aggregateなどの基本的な操作から、[[Flink_状態管理]]や[[Flink_ウィンドウ処理]]、[[ストリーム処理における時間概念]]に基づいた処理など、複雑で柔軟な処理ロジックを実装したい。
    * **状況:** [[Flink_Table_API_SQL]]のような宣言的なAPIでは表現しきれない、より細かい制御やカスタムロジックが必要。開発者がJavaやScalaプログラミングに慣れている。

## 具体的な手順・考慮事項
1.  **`StreamExecutionEnvironment`の取得:** Flinkジョブのエントリーポイントとなる実行環境オブジェクトを取得します。
2.  **Sourceの追加:** `addSource()`メソッドなどでデータソース（Kafka, ファイルなど）から`DataStream<T>`オブジェクトを生成します。
3.  **Transformationの適用:** `DataStream`オブジェクトに対して、各種変換オペレータ（`map()`, `filter()`, `keyBy()`, `window()`, `apply()`, `process()`など）をチェーンさせて処理パイプラインを構築します。
    * `keyBy()`: ストリームをキーに基づいてパーティショニングし、[[Flink_状態管理]] (Keyed State) やキーごとの集計を可能にします。
    * `window()`: [[Flink_ウィンドウ処理]]を適用します。
    * `process()` (ProcessFunction): 最も低レベルなオペレータの一つで、イベント時間処理、状態アクセス、タイマー登録など、高度な制御が可能です。
4.  **Sinkの追加:** `addSink()`メソッドなどで処理結果の`DataStream`を外部システム（Kafka, DBなど）に出力します。
5.  **ジョブの実行:** `env.execute()`を呼び出してジョブを実行します。
6.  **考慮事項:**
    * **データ型:** ストリーム内のデータ型（`<T>`）を意識し、シリアライゼーションが適切に行われるようにする（POJO, Tuple, またはカスタムシリアライザ）。
    * **状態アクセス:** `ProcessFunction`や`RichFunction`内で`ValueState`, `ListState`, `MapState`などのManaged State APIを使用して[[Flink_状態管理]]を行う。
    * **時間処理:** イベント時間を扱う場合は、`assignTimestampsAndWatermarks`を設定し、`ProcessFunction`などでタイマーを利用する。
    * **関数の実装:** ラムダ式、匿名クラス、または`RichFunction`を継承したクラス（`open()`, `close()`メソッドで初期化・後処理が可能）としてオペレータロジックを実装する。
    * **表現力 vs 抽象度:** Table API/SQLに比べて自由度が高い反面、コード量が増え、最適化をFlinkに任せきれない側面もある。