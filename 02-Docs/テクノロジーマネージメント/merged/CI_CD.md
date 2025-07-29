---
aliases:
  - CI/CD
source: "[書籍名 Chapter X / Blog URL]"
tags:
  - 概念
  - テクノロジーマネージメント
---

# CI/CD

## 概要 (Summary)
Continuous Integration（継続的インテグレーション）とContinuous Deployment/Delivery（継続的デプロイメント・デリバリー）の総称。コード変更を頻繁に統合し、自動化されたパイプラインを通じて本番環境へ迅速かつ安全にリリースする開発プラクティス。

## 詳細 (Details)

### Continuous Integration (CI)
- コード変更を頻繁にメインブランチに統合
- 統合のたびに自動ビルド・テストを実行
- 問題の早期発見と解決

### Continuous Deployment/Delivery (CD)
- **Continuous Delivery**: リリース可能な状態を常に維持
- **Continuous Deployment**: 全自動でプロダクション環境にデプロイ

- **原則:**
  - 小さく頻繁な変更
  - 自動化されたテストとデプロイ
  - 迅速なフィードバック
  - ロールバック可能性

- **メリット:**
  - リリース頻度の向上
  - バグの早期発見
  - デプロイリスクの軽減
  - 開発効率の向上
  - チーム間の協調改善

- **デメリット:**
  - 初期投資コストが高い
  - 文化変革が必要
  - テスト品質への依存度が高い
  - モニタリング・観測可能性が必須

- **手順/使い方:**
  1. バージョン管理システムの整備
  2. 自動ビルド・テストの設定
  3. デプロイパイプラインの構築
  4. モニタリング・アラートの設定
  5. ロールバック手順の確立

## 関連ノート (Related Notes)
- [[ソフトウェアテスト]]
- [[プラットフォームマネジメント（Platform Management）]]
- [[DevOps]]

## 未解決の疑問/考察 (Open Questions/Thoughts)
(さらに知りたいこと、疑問点など)

## Geminiによる補足
(Geminiに質問した結果や要約などを記載)