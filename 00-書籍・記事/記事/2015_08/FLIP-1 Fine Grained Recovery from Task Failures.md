---
title: FLIP-1 Fine Grained Recovery from Task Failures
url: https://cwiki.apache.org/confluence/display/FLINK/FLIP-1%3A+Fine+Grained+Recovery+from+Task+Failures
status: Accepted (as of document)
proposer: Stephan Ewen
date: 2015-08-27 (Created)
tags: FLIP, Flink, 分散システム, 耐障害性, リカバリ戦略, 技術文書
---

# FLIP-1: Fine Grained Recovery from Task Failures - 要約

## 概要

このFLIP (Flink Improvement Proposal) 文書は、Apache Flinkにおけるタスク障害からの回復メカニズムを改善するための提案です。従来の[[Flink 粗粒度リカバリモデル]]（All-or-nothing）では、単一のタスクが失敗した場合でもジョブ全体の全タスクを再起動する必要があり、特に大規模なジョブや高並列度のジョブ、とりわけバッチ処理において非効率であるという問題点を指摘しています。

この問題を解決するために、FLIP-1では **[[Flink 細粒度リカバリモデル]] (Fine-grained recovery)** を提案しています。このモデルは、障害の影響を局所化し、失敗したタスクとその依存関係にあるタスクのみを再起動することで、リカバリコストを大幅に削減することを目的としています。

## 提案内容の主要なモデルとメソッド

* **モデル (L1):**
    * [[Flink 粗粒度リカバリモデル]]: 既存のリカバリ戦略とその課題。
    * [[Flink 細粒度リカバリモデル]]: FLIP-1で提案する新しいリカバリ戦略の概念。障害の影響範囲を限定する。

* **メソッド (L2):** 細粒度リカバリを実現するための具体的な技術要素とアプローチ。
    * [[Flink リージョンベースリカバリ]]: 実行グラフを「リージョン」に分割し、リージョン単位でリカバリを行う中心的な手法。
    * [[Flink 実行グラフコンポーネント化]]: 細粒度リカバリの実装を可能にするための、実行グラフとその構成要素のリファクタリング。
    * [[Flink 結果パーティション管理(細粒度リカバリ)]]: 失敗しなかった上流タスクの結果を再利用可能にするための、中間結果のライフサイクル管理。
    * [[Flink フォールバックリカバリ(細粒度)]]: 細粒度リカバリが頻繁に失敗する場合に、従来の粗粒度リカバリへ切り替えるための戦略。

## 目的と効果

* 大規模・高並列度ジョブにおけるタスク失敗時のリカバリ時間を短縮する。
* 特にバッチ処理ジョブの効率を向上させる。
* クラスタリソースの浪費を削減する。

このFLIPは承認され、Flinkのコア機能として実装されました。

## 関連リンク

* [[Flink 粗粒度リカバリモデル]]
* [[Flink 細粒度リカバリモデル]]
* [[Flink リージョンベースリカバリ]]
* [[Flink 実行グラフコンポーネント化]]
* [[Flink 結果パーティション管理(細粒度リカバリ)]]
* [[Flink フォールバックリカバリ(細粒度)]]
