---
title: "Elasticsearchのベクトル検索におけるフィルター使用時のパフォーマンス改善 | メルカリエンジニアリング"
source: "https://engineering.mercari.com/blog/entry/20250718-03e1dbe909/"
author:
  - "[[mrkm4ntr]]"
published: 2025-07-18
created: 2025-07-28
description: "Search Infra Teamのmrkm4ntrです。画像検索にElasticsearchのベクトル検索(kNN検索)を活用しています。しかし、従来のキーワード検索と比較して、同等のリソースで処理できるQPS（Queries Per S"
tags:
  - "clippings"
---
![Elasticsearchのベクトル検索におけるフィルター使用時のパフォーマンス改善](https://storage.googleapis.com/prd-engineering-asset/2025/06/d232f046-eyecatch.png)

Search Infra Teamのmrkm4ntrです。

画像検索にElasticsearchのベクトル検索(kNN検索)を活用しています。しかし、従来のキーワード検索と比較して、同等のリソースで処理できるQPS（Queries Per Second）が大幅に低いという課題がありました。そこで、Elasticsearch 8を基に、kNN検索のパフォーマンスをどこまで改善できるのかを調査しました。

## kNN検索の構成と課題

今回の検証で使用したkNN検索のクエリ構成は以下の通りです。

```json
{
  "size": 100,
  "query": {
    "knn": {
      "image_embedding": {
        "vector": [
          0.1,
          0.2,
          ... (128次元のベクトル)
        ],
        "k": 100,
        "num_candidates": 100,
        "filter": {
          "term": {
            "status": "on_sale"
          }
        }
      }
    }
  }
}
```

このクエリは、「status」フィールドが「on\_sale」に一致するドキュメントの中から、与えられたベクトル（image\_embedding）に類似した上位100件のドキュメントを検索するものです。ベクトルの次元数は128です。

検証当初はElasticsearch 8.12.1を使用しており、async-profilerを用いてCPUプロファイルを取得した結果、以下の箇所がボトルネックとなっていることが判明しました。

![](https://storage.googleapis.com/prd-engineering-asset/2025/07/2cb0ad58-8.12.1.png)

特に目立つのは、赤枠で囲まれた `jint_disjoint_arraycopy` と `jlong_disjoint_arraycopy` です。これらのメソッドは、 `OffHeapQuantizedByteVectorValues.vectorValue` から呼び出されており、JVMのヒープ外（つまりファイルシステムキャッシュ上）に格納されているベクトルデータをJVMのヒープ内にコピーする処理を行っています。

Luceneでは高速なkNN検索を実現するためにPanama Vector APIを活用しています。このAPIはベクトル計算でプロセッサのSIMD命令（AVX命令）を使用し、演算効率を向上させるものです。しかし、ベクターデータを一度JVMヒープ内にコピーしてからPanama Vector APIに渡す無駄が発生しているため、パフォーマンスが大きく制約されています。

このボトルネックを軽減する可能性がある修正がLuceneに施されていることが分かりました。具体的にはhttps://github.com/apache/lucene/pull/13339 の変更で、JVMヒープ外メモリから直接Panama Vector APIに渡す実装に改善されています。

## Elasticsearch 8.17.1へのアップデートとパフォーマンスの変化

上記の改善が既に含まれるElasticsearch 8.17.1にアップデートし、パフォーマンスを検証しました。

検証環境

- Elasticsearch Version: 8.17.1
- k: 100
- num\_candidates: 100
- ベクトルの次元数: 128
- リアルタイムなドキュメントの追加/更新/削除

フィルタリングの有無と度合いがパフォーマンスに与える影響を評価するため、以下の3つのケースで同一のスペックで捌けるQPSをパフォーマンスとして計測しました。

- フィルタなし: フィルタリングなし
- 緩いフィルタ（loose filter）: 約50%のドキュメントがフィルタ条件に一致
- 絞り込みフィルタ（selective filter）: 約10%のドキュメントがフィルタ条件に一致

フィルタなしのケースにおいて、async-profilerでCPUプロファイルを再取得したところ、flame graphからPanama Vector APIの痕跡が消え、代わりに赤枠で囲んだ `dot7u` というメソッドが表示されるようになりました。

![](https://storage.googleapis.com/prd-engineering-asset/2025/07/a551abd1-8.17.1-no-filter.png)

これは、Elasticsearch 8.15から、LuceneのPanama Vector APIではなく、より効率的なElasticsearch独自のベクトル化モジュールが使用されるようになったためです ([https://www.elastic.co/search-labs/blog/vector-similarity-computations-ludicrous-speed](https://www.elastic.co/search-labs/blog/vector-similarity-computations-ludicrous-speed)) 。この独自実装では、不要なデータコピーはそもそも発生しません。

8.12.1と8.17.1のパフォーマンスの比較は以下です。

|  | 8.12.1 | 8.17.1 |
| --- | --- | --- |
| フィルタなし | 350 | 450 |
| 緩いフィルタ（loose filter） | 50 | 70 |
| 絞り込みフィルタ（selective filter） | 30 | 40 |

Elasticsearch 8.17.1にアップデート後、全体的にパフォーマンスが改善しましたが、フィルタありのケースでは依然としてフィルタ無しのケースと比べるとパフォーマンスが低い状態です。

## 絞り込みフィルタリング時のHNSWグラフ探索の最適化

絞り込みフィルタ（selective filter）を適用した場合のCPUプロファイルを確認したところ、赤枠で囲まれた部分の `HNSWGraphSearcher.search` の処理時間が大幅に増加していることがわかりました。

![](https://storage.googleapis.com/prd-engineering-asset/2025/07/6ad279c8-8.17.1-selective-filter.png)

これは近傍のノードをチェックする際に、類似度の計算を行わず、諦めて次をチェックする回数が増加していることを意味します。 つまり、HNSWグラフの構造上、フィルタ条件に合致するドキュメントが少ない場合、グラフのリンクを効率的に辿ることができないために発生する問題です。類似度が高い方向にグラフを探索しても、フィルタ条件に合致するドキュメントが見つからない場合、大きく迂回したり、戻ったりする必要が生じ、探索効率が低下します。

この問題を解決するために、ACORNと呼ばれるアルゴリズム([https://arxiv.org/pdf/2403.04871](https://arxiv.org/pdf/2403.04871)) が提案されており、様々なベクトル検索エンジンが採用しています。Luceneのupstreamでも、ACORN風のアルゴリズムが実装されているため、このPR ([https://github.com/apache/lucene/pull/14160](https://github.com/apache/lucene/pull/14160)) をcherry-pickして試したところ、特に絞り込みフィルタの場合に大きなパフォーマンス改善が見られました。

|  | 8.17.1 | 8.17.1 + ACORN |
| --- | --- | --- |
| フィルタなし | 450 | 450 |
| 緩いフィルタ（loose filter） | 70 | 80 |
| 絞り込みフィルタ（selective filter） | 40 | 120 |

とはいえ、緩いフィルタ（loose filter）の場合はあまり改善されていません。

## 緩いフィルタリング時のBitSet合成の最適化

緩いフィルタ（loose filter）を適用した場合のCPUプロファイルを確認したところ、赤枠で囲んだ部分である `AbstractKnnVectorQuery.createBitSet` の処理時間が大部分を占めていることがわかりました。

![](https://storage.googleapis.com/prd-engineering-asset/2025/07/faf66afc-8.17.1-acorn-loose-filter.png)

フィルタが固定で、kNN検索に指定するベクトル値のみを変更している場合、フィルタの結果のBitSetはクエリキャッシュに保存されるため、フィルタ自体のコストはほぼ無視できるはずです。

コードを解析した結果、createBitSetメソッド内で、フィルタのBitSetとliveDocsのBitSetを合成した新しいBitSetを作成していることが判明しました。LuceneのSegmentはimmutableであるため、削除されたドキュメントを管理するために別のデータ構造（liveDocs）が必要になります。liveDocsもBitSetで表現されており、フィルタのBitSetとliveDocsのBitSetを合成する際に、BitSetの中身をiterateしていました。緩いフィルタの場合、フィルタ条件に合致するドキュメントが多いため、この処理に大きなコストがかかっていました。

しかし、グラフを辿る際に、見つけたドキュメントがフィルタに合致するかをチェックするだけであれば、BitSetを合成する必要はありません。また、カーディナリティ（BitSet内で1が立っているビット数）を計算する場合も、iterateして合成する必要はなく、FixedBitSetのintersectionCountメソッドを使用することで高速に計算できます。

これらの点を修正した結果、特に緩いフィルタを使用した場合のパフォーマンスが大幅に改善しました。

|  | 8.17.1 + ACORN | 8.17.1 + ACORN + BitSet合成最適化 |
| --- | --- | --- |
| フィルタなし | 450 | 450 |
| 緩いフィルタ（loose filter） | 80 | 200 |
| 絞り込みフィルタ（selective filter） | 120 | 170 |

この修正はLuceneのupstreamにPRとして送りました([https://github.com/apache/lucene/pull/14771](https://github.com/apache/lucene/pull/14771)) 。しかし、その少し前に追加されていた修正 ([https://github.com/apache/lucene/pull/14674](https://github.com/apache/lucene/pull/14674)) にて改善されていたためこのPRは不要でした。正確には、このPRそのものはcherry pickして試していましたが、 `applyMask` の非default実装は別PRで対応されていたため、それを見落としていました。

## FieldExistQueryによるパフォーマンス低下の解消

改善後のCPUプロファイルを確認したところ、依然としてcreateBitSet内の `Lucene99ScalarQuantizedVectorsReader$QuantizedVectorValues.advance` の処理時間が残っていました。

![](https://storage.googleapis.com/prd-engineering-asset/2025/07/5fe9d775-8.17.1-acorn-bitset-loose-filter.png)

これは、フィルタにLucene内部で追加されるFieldExistQueryによるものであることがわかりました。インデックス内のすべてのドキュメントにkNNの対象となるベクトルフィールドが存在するわけではない場合、その存在をチェックする追加の処理が必要になります。今回のケースでは、埋め込み処理でエラーが発生した場合にベクトルが存在しないドキュメントが存在していました。

これらのドキュメントをインデックスに含めないように修正したところ、パフォーマンスがさらに改善しました。

|  | 8.17.1 + ACORN + BitSet合成最適化 | 8.17.1 + ACORN + BitSet合成最適化 + FieldExistQueryの除外 |
| --- | --- | --- |
| フィルタなし | 450 | 450 |
| 緩いフィルタ（loose filter） | 200 | 250 |
| 絞り込みフィルタ（selective filter） | 170 | 200 |

一般にFieldExistQueryは対象のドキュメントのfieldを全てチェックする必要があるため高コストです。FieldExistQueryと今回のフィルタはQuery Cacheの対象のため全てのドキュメントをチェックしているわけではないはずですが、Query Cacheの対象とならないようなサイズのセグメントのみが対象だったとしても、緩いフィルタの場合は高コストであったと考えられます。

## さいごに

Elasticsearch 8のkNN検索において、フィルタリング時のパフォーマンスを改善するために、以下の施策を実施しました。

1. Elasticsearch 8.17.1へのアップデート: Elasticsearch独自のベクトル化モジュールを使用することで、フィルタなしのケースにおけるパフォーマンスを向上させました。
2. ACORN風アルゴリズムの導入: 絞り込みフィルタ適用時のHNSWグラフ探索を最適化しました。
3. BitSet合成の最適化: 緩いフィルタ適用時のBitSet合成処理を効率化しました。
4. ベクトルフィールドの存在チェックの排除: ベクトルフィールドが存在しないドキュメントをインデックスから排除することで、不要な存在チェックを削減しました。

これらの改善により、フィルタリングの有無に関わらず、kNN検索のパフォーマンスを大幅に向上させることができました。以下が最終的な結果です。

|  | 改善前 | 改善後 |
| --- | --- | --- |
| フィルタなし | 350 | 450 |
| 緩いフィルタ（loose filter） | 50 | 250 |
| 絞り込みフィルタ（selective filter） | 30 | 200 |

ベクトルフィールドの存在チェック以外の改善は将来のElasticsearchのリリース(9.0.4以降?)に含まれる予定です。

これらの改善は、ElasticsearchのkNN検索をHybrid Searchなどに活用し、より高度な検索サービスの提供に繋がるものと考えています。