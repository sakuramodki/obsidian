---
title: 機能部分をwebと分離することによるE2Eテストの削減
url: https://blog.cybozu.io/entry/2025/05/20/113000
author: kintone
published_date: 2025-05-20
tags:
  - ブログ
  - ユニットテスト
  - 単体テスト
  - 結合試験
  - 関心の分離
---
# 要約

## kintoneにおけるE2Eテスト削減に向けた取り組みのまとめ

### はじめに

- kintoneチームでは、新規機能開発時にユーザーストーリーを担保するためE2Eテストを追加していた。
- これにより品質を担保できたが、E2Eテスト数が膨大になり大きな負担となり、非効率さが目立つようになった。
- この状況を改善するため、機能に関わる処理をwebから分離するという内部設計の改善により、E2Eテスト削減を目指している。

### E2Eテストの種類とその問題

- kintoneのE2Eテストには**E2E-uiテスト**と**E2E-apiテスト**の二種類がある。
- **E2E-uiテスト**はSeleniumを使用し、ユーザー操作を模擬するテスト。
- **E2E-apiテスト**はサーバーサイドAPIの振る舞いを検査するテストで、主にREST APIに利用されるが内部APIのテストにも使われる。
- どちらのテストも開発環境に本番と同等の状態（ユーザー管理システム、Elasticsearch、ジョブキューなど）をデプロイしてテストされる。
- それぞれのテストケース数が**数千ケースを超え**ており、開発の非効率さを生んでいる。
- 実行に時間がかかり、特にE2E-uiテストは製品に問題がなくてもテストが失敗する**不安定さ**がある.。
- 不具合を捕捉しても実行範囲が広いため、どこで問題が発生したか分かりづらい。
- 例えば、E2E-uiテストがボタンクリック失敗で落ちた場合、不安定さ、CSSミス、サーバーサイドの問題など原因特定が困難。
- 現実的な時間でテストを終えるには並列化が必要で、コンポーネントフルセットの並列デプロイや大量リソース消費が必要となり、何度も実行しにくく、**リリース頻度増加の妨げ**となる。

### 今までの取り組み

- E2Eテスト改善のためいくつかの取り組みが行われてきた。
- QAとPGが協力し、E2E-uiテスト追加を抑制し、**E2E-apiテストとフロントエンドのテストでカバー**するようにした。
- kintoneはReact化が進み、モダンなツールでフロントエンドに閉じたテストでも品質担保が可能になった。
- 2024年ではE2E-uiテストはほとんど追加されていない。
- これによりSelenium起因の不安定テスト増加は抑えられている。
- また、**テストの分割**によってE2Eテストの部分実行が可能になり、実行時間はある程度短縮可能。
- しかし、既存の数千ケースあるE2E-uiテストは手つかずのまま。
- さらに、E2E-apiテストは増えてしまっている。

### 内部設計の見直しとE2Eテストの削減

- E2Eテスト削減のため、kintoneの内部設計を見直す取り組みを開始した。
- 具体的には、サーバーサイドAPI実装において、kintoneの機能に関する部分とwebに関する部分を分離し、**機能部分に絞って結合テストを行う**というもの。
- kintoneのサーバーサイドAPI実装はコントローラー層、サービスクラス、リポジトリ、データクラスなどで構成される。
- サービスクラス、リポジトリ、データクラスは**機能に関する部分**。
- コントローラー層はJSONのエンコード/デコード、バリデーション、型変換、リクエストURIからの情報取得など**webに関する処理**が必要。
- コントローラー層では機能に関する処理とwebに関する処理が混然一体となっている。
- 主要な処理はサービスクラスに切り出されているが、コントローラー層とサービスクラスの境界は曖昧で実装者に依存していた。
- 分離前のコード例（ブックマーク追加APIコントローラー）では、JSONデコードやURL解析はweb処理だが、DTO生成やアプリIDセットは更新処理の一部であり、コントローラーに書かれていた。このようなコードはAPIテストでテストする必要があった。

#### 機能に関する部分をwebと分離

- 機能に関する処理の一切を**コントローラー層から無くす**。
- コントローラーにある処理のうち、機能に関する部分を**API専用のインターフェイス**に切り出し、コントローラーはこのインターフェイスを呼び出す構造にする。
- コントローラー層で可能な処理は、webに関する処理とAPI用インターフェイスの呼び出しのみ。
- 機能実装に関する知識が入り込まないよう、API用インターフェイスのメソッド呼び出しは**一回だけ**。
- 分離を完全にするため、クラスの依存も制限し、機能実装を**内部実装としてコントローラー層から隔離**する。
- コントローラー層が依存する機能関連クラスはAPI用インターフェイスを含め `.exporttoweb` パッケージ下にあるクラスのみにする。
- API用インターフェイスのメソッドのリクエストモデルやレスポンスモデルも必要に応じて専用に切り出す。
- SpringのDIにより実装は自動注入され、コントローラー層に公開されるのはAPI用インターフェイスとそれに付随するリクエスト/レスポンスモデルのみ。
- 結果としてコントローラー層は `.exporttoweb` にあること以外、機能実装について何も知らなくなる。
- ブックマーク機能への適用例では、コントローラーはJSONエンコード/デコード、型変換、**BookmarkApiService**（API用インターフェイス）の呼び出しのみを行う。
- BookmarkApiService には機能として提供される振る舞い（`add`, `list`など）が定義され、リクエスト/レスポンスモデル（`AddRequest`, `AddResponse`など）も含まれる。
- 機能から分離されたコントローラー層は、ユーザーと機能（BookmarkApiService）を繋ぐ役割になる。
- API用インターフェイスは、kintoneの機能を定義する**抽象データ型**のようなもの。

### 結合テストの導入

- 機能とwebを分離したことにより、E2E-apiテストを減らして代わりに**API用インターフェイスの結合テスト**が利用できるようになる。
- 機能として提供すべき振る舞いはAPI用インターフェイスに定義され、それ以外の機能に関する実装は内部実装として隠ぺいされているため。
- API用インターフェイスのテストでカバーできないのはwebに関する処理（JSONエンコード/デコードなど）。
- webの部分には機能に関する処理が全く含まれていないため、機能をテストする場合にweb部分まで含むE2E-apiテストを書く必要がなくなる。
- ブックマーク機能では、BookmarkApiService以外にブックマーク追加処理が存在しないため、BookmarkApiService以外をテストする必要はない。
- テスト例（`BookmarkWebApiServiceTest`）では、`add`や`AddRequest`、`list`といったBookmarkApiServiceの公開インターフェイスにのみ依存しており、内部実装（BookmarkDto、リポジトリなど）の知識が出てこない。
- これにより、**内部の変更に対して頑健**であり、kintoneの機能部分のみをテスト対象に絞れている。
- API用インターフェイスのテストはクラスのテストとなる。
- DBは本物のDBと結合し、ユーザー管理などのその他のコンポーネントはフェイクを利用している。これは以前紹介されたテスト記事と同じ構成である.。

### 導入と効果、課題

- 導入箇所はまだ少ないが、効果は実感している。
- アプリ設定のカテゴリ機能では、数十あったE2E-uiテストが、ハッピーパス検証の一件のみになった。
- 残った一件以外のE2EテストはAPI用インターフェイスのテストとフロントエンドテストに置き換わった。
- この仕組みは既存コードを大きく変えないため、特別な準備なく分離作業に取り掛かれ、大きな支障なく終えることができた。
- 分離作業はコントローラー層の機能関連処理をAPI用インターフェイスの実装に移動するというもの。リクエスト/レスポンスモデル生成といった新規クラスはあるが、処理自体はそのまま。他の機能でも作業内容は大きく変わらない印象。
- コードの分割が完了していると、このような内部設計変更はやりやすいと感じた。カテゴリ機能もコード分割済み。
- 機能の範囲やコード規模、影響範囲が小さく閉じることで、素早く完了できた。
- 問題をできるだけ起こさず、段階的に学習しながら進められるようにも思う。
- 今後も同様の改善を続けることで、E2Eテストを大きく削減できると考えている。
- kintoneチームは以前からE2E-uiテスト削減を目指し、E2E-apiテストやフロントエンドテストでスコープを絞る努力をしてきた。今回のAPI用インターフェイステストもその流れにある改善。
- 新しい書き方や考え方の学習は必要かもしれないが、今までの考え方の延長で進められるのではないか。
- 機能実装とwebが分離されたことで、E2Eテスト削減以外に**コードの可読性も改善**された。
- 分離前は、更新系API実装で更新処理の一部がコントローラー層の様々な場所に散らばる可能性があった（例: `setAppId`）。
- 分離後は、処理が散らばる範囲が内部実装に抑えられたため、読みやすくなった。
- 更新用DTO作成処理も機能関連処理となりwebと分離されたため、コードを読む際にweb処理が混じらずDTO組み立てに集中できるようになった。
- **課題**としては、**似たようなクラスが増えてしまう**ことが挙げられる。AddRequest/AddResponseとAddInput/AddOutputのように、ほぼ同じ構成のクラスが二つできることがある。
- クラスを省略するのは難しく、Springの仕組み上、クラスが増えてしまうのは避けられない印象。

### 今後の展望

- webと分離した機能部分の**内部実装を整理**したいと考えている.。
- 更新用処理が内部実装内で散らばってしまう問題や、正しいDTOの状態が分かりづらい問題は残っている.。これはDTO中心で処理を組んできたkintoneの長年の問題.。
- この問題を解決するには、機能の正しい状態のみをクラスで表現するなど、状態や情報の整合性/一貫性を担保しやすくする仕組みが必要.。
- 内部実装の整理には**クリーンアーキテクチャ**がヒントになると考えている。
- 今回E2Eテスト削減のために作った構造はクリーンアーキテクチャの構造と似ている。
- クリーンアーキテクチャのControllerとUse caseの間にあるBoundaryに、今回作成したAPI用インターフェイスが似た役割を果たす。
- 今回の内部設計変更は、クリーンアーキテクチャにおけるControllerとBoundaryの導入に相当する。
- クリーンアーキテクチャを参考に内部実装の中にUse caseやEntityを見出していけば、自然と整理されるのではないか。
- このような内部実装を大きく変える変更をしても、API用インターフェイスは変わらない。
- したがって、E2Eテストから置き換えて作った**結合テストが活躍してくれる**と期待している。

### まとめ

- 機能に関わる処理をwebと分離することによるE2Eテストの削減について紹介された.。
- 膨大なE2Eテストが開発を非効率にしていた.。
- 内部設計を見直し、機能実装をwebから分離し、機能実装だけを対象にテストを行う仕組みを導入した.。
- アプリ設定のカテゴリ設定機能などで試した結果、実際にE2Eテストが削減された.。
- この仕組みによりコードの可読性も改善できそうなことが分かった.。
- この仕組みはまだ導入が始まったばかり.。今後もE2Eテストの大幅な削減を目指して続ける予定.。

## Geminiによる補足
### 感想
E2Eテストで実施していたテストを関心の分離をすることで結合試験にしてテストを軽くできたよという事例紹介記事。
すごいスタンダードな書き方というわけではないけど、面白いなぁと思った。
