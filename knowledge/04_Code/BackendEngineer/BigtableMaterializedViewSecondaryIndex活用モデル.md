# Bigtable Materialized View & Secondary Index 活用モデル (Bigtable Materialized View & Secondary Index Utilization Model)

## モデル概要
Google BigtableのLSM TreeとKey Range Partitionに起因する限界点（Single-Column Sequential Scanの強制）を克服し、多様なクエリ要件に対応するために、Materialized ViewとSecondary Indexを導入・活用する技術的アプローチと、その特性、注意点を説明するモデル。

## モデル適用条件
Google Bigtableを使用しており、既存のRow Key設計では対応が困難な多様なアクセスパターン（異なるフィルター、ソート順）や、集計要件が発生した場合。特に、データ構造の変更を伴わずに新しいクエリ要件に対応したいケース。

## モデル構造
- **Bigtableの基本特性:** LSM TreeとKey Range PartitionによるSingle-Column Sequential Scanの強みと限界。
- **Materialized View (具体化ビュー):**
    - **定義:** 物理的なデータを持つビュー。SQLクエリで定義可能で、リアルタイム同期される。
    - **強み:** Managed Data Processing Pipeline、既存テーブル変更なしでの新データ構造定義、Wide-Column互換、Low-Priority Batch Job。
    - **注意点:** Preview Featureの制限（1テーブル1MV、クエリ修正不可）、Eventual Consistency。
- **Secondary Index:**
    - **定義:** Materialized Viewの一種で、`ORDER BY`句によりStructured Row Keyを生成し、異なるアクセスパターンを可能にする。
    - **注意点:** `ORDER BY`順序でStructured Row Keyが生成される、区切り文字`\x00\x01`、別のテーブルとして扱われる。
- **費用:** ストレージコストとコンピューティングリソース（既存クラスタのリソースを使用）。

## モデル法則
- NoSQLデータベースの特性（例: BigtableのLSM TreeとKey Range Partition）は、特定のアクセスパターンに最適化されているが、それ以外の多様なアクセスパターンには限界がある。
- Materialized ViewやSecondary Indexのような機能は、基盤となるデータ構造を変更することなく、異なるクエリ要件に対応するための強力な手段を提供する。
- 新機能の導入には、その特性（同期モデル、制限、コスト）を深く理解し、既存システムへの影響を考慮した上で慎重に進める必要がある。

## 実装ガイドライン
- BigtableのRow Key設計で対応できない多様なクエリ要件が発生した場合、Materialized ViewやSecondary Indexの導入を検討する。
- Materialized Viewを定義する際は、SQLクエリを慎重に設計し、将来的な変更の可能性を考慮する（Preview Featureの制限に注意）。
- Secondary Indexを利用する際は、`ORDER BY`句でStructured Row Keyの生成順序を明確に定義し、アプリケーション側でのアクセス方法を考慮する。
- コスト（ストレージ、コンピューティング）を事前に評価し、既存クラスタへの影響を考慮する。
- Preview Featureであるため、正式リリース後の仕様変更に備える。

## モデル限界と注意点
- Preview Featureであるため、将来的に仕様が変更される可能性がある。
- Materialized Viewのクエリ修正ができないため、設計段階での慎重な検討が必要。
- Eventual Consistencyを理解し、アプリケーション側でその特性を考慮した設計が必要。
- 別のテーブルとして扱われるため、データ管理や運用が複雑になる可能性がある。

## 関連事例

### 実装事例
- [Bigtable Materialized View & Secondary Index 紹介 | CyberAgent Developers Blog](https://developers.cyberagent.co.jp/blog/archives/57875/)
  - ABEMAにおけるBigtableのMaterialized ViewとSecondary Indexの導入事例。
  - Bigtableの特性と限界、新機能の具体的な実装方法、強み、注意点、費用に関する詳細な情報が提供されている。

### 関連モデル
- [新・単一責任原則モデル](/knowledge/04_Code/BackendEngineer/新・単一責任原則モデル.md) - データベースのアクセスパターンを分離し、各コンポーネントの責任を明確にする。
- [制約駆動型データ構造設計モデル](/knowledge/04_Code/BackendEngineer/制約駆動型データ構造設計モデル.md) - データベースのスキーマ設計において、アクセスパターンという制約を考慮したデータ構造の重要性を示す。
