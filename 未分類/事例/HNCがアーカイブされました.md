分類: #事例 #ブログ

# 関連情報
URL : https://ca-srg.dev/1d94358b43f78002917fc30c657b53bd
# コンテキスト
KubernetesのHierarchical Namespaces Controller (HNC) がアーカイブされた

# 要約

- **HNCのアーカイブ:**
    - KubernetesのNamespaceを階層的に管理するツールであるHNCが、2025年4月17日に正式にアーカイブされました。
    - アーカイブの主な理由は、メンテナー不足と採用事例の少なさです。
    - Kubernetesコアへの統合予定はありません。
- **技術的な影響:**
    - HNCの最新イメージはGoogle Container Registry (gcr.io) でホストされていますが、GCRは2025年4月22日以降イメージの読み取りが停止される予定です。
    - これにより、HNC Podの新規起動時にイメージ取得エラーが発生する可能性があります。
- **対策:**
    - **短期的:** HNCイメージを自社のコンテナレジストリ（例: AWS ECR）にコピーし、そこからイメージをPullするように設定を変更することが推奨されます。
    - **中長期的:** `Accurate`（サイボウズ製）や`Capsule`（CNCF Sandboxプロジェクト）などの代替技術への移行、または独自ツールの開発を検討することが推奨されます。