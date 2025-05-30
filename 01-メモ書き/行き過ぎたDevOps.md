# プロダクト開発の不確実性
ソフトウェア開発など新しいものを作る際には、主に二つの不確かさがあります。それは「何を作るべきか（目的不確実性）」と「どうやって作るか（方法不確実性）」です。これらをうまく管理することが成功の鍵となります。
## 目的不確実性（何を創るべきか）の管理
この不確かさは、「本当にユーザーが求めるものか？」「この機能は価値があるのか？」といった点に関するものです。これに対応するには、仮説を立てて検証するアプローチが有効です。
 * まず小さく試す (仮説検証型)：
   例えば、「MVP（実用最小限の製品）」を作り、本当に必要な機能だけを素早く市場に出します。これにより、実際のユーザーの反応を見て、製品の方向性が正しいか、何が求められているかを早期に学びます。A/Bテストでどちらの案が良いか試すのもこの一種です。
 * ユーザーを深く知る (顧客中心型)：
   「ユーザーリサーチ」を通じて、顧客インタビューやアンケートを行い、彼らが抱える課題や真のニーズを掘り下げます。これにより、「きっとこれが欲しかったはず」という思い込みを防ぎます。
 * 少しずつ確認しながら進める (反復型)：
   「アジャイル開発」のように、短い期間で開発とフィードバックを繰り返します。これにより、状況の変化に対応しつつ、目的からズレていないかを定期的に確認しながら、価値あるものを段階的に作り上げていきます。
## 方法不確実性（どうやって創るか）の管理
この不確かさは、「技術的に実現できるのか？」「どうすれば効率よく安全に作れるか？」といった、手段やプロセスに関するものです。これには、計画性や技術的な検証、標準化が役立ちます。
 * 計画で備える (計画・リスク管理型)：
   プロジェクトの初期に「リスクマネジメント」を行い、技術的な課題や潜在的な障害を洗い出し、対策を準備しておきます。これにより、予期せぬ問題による手戻りや遅延を防ぎます。
 * 技術を試して確かめる (技術検証型)：
   新しい技術や複雑な仕組みを導入する際は、「PoC（概念実証）」で小規模に試作し、技術的な実現可能性や課題を事前に把握します。これにより、「作ってみたけど動かなかった」という事態を避けられます。
 * 仕組みで安定させる (標準化・自動化型)：
   「CI/CD（継続的インテグレーション/継続的デリバリー）」のような仕組みを導入し、テストやリリース作業を自動化・標準化します。これにより、人為的ミスを減らし、品質を保ちながら迅速かつ確実に開発を進めることができます。
これらのアプローチは、どちらか一方だけではなく、プロジェクトの特性や状況に応じて組み合わせて活用することが、不確実性を効果的にコントロールし、成功に導くために重要です。

# 行き過ぎたDevOps
Devは主に新規機能の開発を指します。
これは主に「新規機能が市場に受け入れられるか」という目的不確実性が支配的です。

Opsは主に機能の維持のための活動を指します。
これは主に「どのように行うか」の方法不確実性が支配的です。

DevとOpsのタスクは明確に区別ができないことも多いですが、両者の抱えてる不確実性で支配的であるものを意識しながらタスクを進める必要があります。
目的不確実性がないタスクにまでA/Bテストを行う必要はないですし、目的が不確実なタスクの計画を立てても無駄になります。