---
title: Reactコンポーネントが「純粋である」とはどういうことか？　丁寧な解説
url: https://zenn.dev/uhyo/articles/react-pure-components
author: uhyo
published_date: 2025-05-08
tags:
  - ブログ
  - 概念
  - テクノロジーマネージメント
  - React
  - 副作用
---
# 要約
関数が純粋であるというのは次の性質を指す。
- **副作用を含まない**。つまり、関数を実行した結果として、関数の外部に対して影響を与えないこと。
- **参照透過性を持つ。** つまり、同じ入力を与えたときに、常に**同じ結果**を返すこと。

このブログでは「同じ結果」を掘り下げて「指示書」という考え方を提案している。
つまりJSとして `===` ではないオブジェクトでもReactに同じDOMを組み立てる事を指示していれば同じであるという考え方だ。

Reactで純粋な関数を扱うにあたって、勘違いされそうなポイントとして下記２点がある。
- イベントハンドラはDOMを組み立てる時点で評価されないので、この中身まで純粋である必要はない
- hookは入力の一種なので外部への参照ではない

## 関連ノート (Related Notes)
- [[Reactの文脈における2種類の副作用]]

## 未解決の疑問/考察 (Open Questions/Thoughts)
(さらに知りたいこと、疑問点など)

## Geminiによる補足
(Geminiに質問した結果や要約などを記載)
