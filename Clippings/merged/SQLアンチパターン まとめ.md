---
title: "SQLアンチパターン まとめ"
source: "https://zenn.dev/kody/articles/31b4cc78c8434a"
author:
  - "[[Zenn]]"
published: 2023-12-07
created: 2025-07-29
description:
tags:
  - "clippings"
---
9

2[tech](https://zenn.dev/tech-or-idea)

## はじめに

これは フラー株式会社 Advent Calendar 2023 の7日目の記事です。6日目は [@shogo82148](https://qiita.com/shogo82148) さんの [「今年もアドベントカレンダー（物理）買いました」](https://shogo82148.github.io/blog/2023/12/06/2023-12-06-advent-calendar/) 」でした。

---

[SQLアンチパターン](https://amzn.asia/d/ctR3tX6) の本を読んだので、各章のまとめです。  
少し長いですが、最後までお付き合いください。

## ジェイクウォーク（信号無視）

## 目的

以下のような `Products` テーブルがあると仮定します。

```sql
CREATE TABLE Products (
  product_id SERIAL PRIMARY KEY,
  product_name VARCHAR(1000),
  account_id BIGINT UNSIGNED,
    -- 他の列...
  FOREIN KEY (account_id) REFERENCES Accounts(account_id)
);
```

当初の `Products` テーブルは1つのアカウント (`Accounts` テーブル) を参照しています。

つまり、製品とアカウントには「多対1」の関連があります。

しかし、後にアカウントが複数の連絡先を持つ場合があることが分かり、多対1関連だけでなく、製品からアカウントに対する1対多の関連をサポートする必要があり、テーブル設計を見直す必要があります。

## アンチパターン

### カンマ区切りフォーマットのリストを格納する

以下のように `account_id` 列を `VARCHAR` 型の列として再定義し、複数アカウントのIDをカンマ区切りで登録できるようにする。

```sql
CREATE TABLE Products (
  product_id SERIAL PRIMARY KEY,
  product_name VARCHAR(1000),
  account_id VARCHAR(1000), -- カンマ区切りのリスト
    -- 他の列...
  FOREIN KEY (account_id) REFERENCES Accounts(account_id)
);
```

### デメリット

1. **特定のアカウントに関数製品の検索するためのクエリを作ることが困難になり、メンテナンス性やパフォーマンスなどが悪化する**
	1. 例えばMySQLではaccount\_idが12のアカウントが指定されたすべての製品を取得するためには以下のようなクエリを記述します。
		```sql
		SELECT * FROM Products WHERE account_id REGEXP '[[:<:]]12[[:>:]]';
		```
	2. インデックスを使うメリットが得られない。
	3. パターンマッチ構文はデータベース製品によって書き方が異なる。
2. **アカウントIDの妥当性検証ができない**
	1. ユーザーが `banana` のような無効な入力をした場合はどうやって防げるのでしょうか。
	```sql
	INSERT INTO Products (product_id, product_name, account_id) VALUES (DEFAULT, 'Visual TurboBuilder', '12,34,banana');
	```
3. **リストの長さ制限がある**
	1. 例えばデータ型が `VARCHAR` (30)の場合エントリの長さが2文字の場合、10個リストに格納でき（カンマを含めて1エントリ3文字）、しかしエントリの長さが6文字の場合、リストに格納できるのは4個のみです。
	2. また、将来必要となるかもしれないリストの長さをどうやって判断することができるのでしょうか。

## 解決策: 交差テーブルを作成する

`account_id` をProductsテーブルに格納するのではなく、新たに作成したテーブルの各行に `account_id` を1つずつ格納します。。新たに作成した `Contacts` テーブルによって、 `Products` と `Accounts` の間には「多対多」の関連が生じます。

このように設計することでパターンマッチなどを使用することなく、検索することができます。

```sql
SELECT * FROM Contacts WHERE account_id = 12;
```

また、カンマやスラッシュを使用しないため、SQLのデータ型によって入力内容を制限できます。

さらに各エントリは交差テーブルの個別の行に格納されるのでリストの長さ制限の問題も解消されれます。

## まとめ

ひとつひとつの値は個別の行と列に格納しましょう。

## ナイーブツリー（素朴な木）

## 目的

ツリー状の構造や、階層的な構造を表現したいです。

各記事にはコメントを書き込めるほか、読者同士でのディスカッションも可能で、コメントはスレッド形式で表示されます。スレッドはディスカッションのトピックに応じて枝分かれします。

Xのツイートに対する返信をイメージするとピンとくるかと思います。

## アンチパターン

### 常に親のみに依存する

上記の目的を達成するために、 `parent_id` 列を加えることが単純な方法です。

しかし、この方法は思慮が浅い、ナイーブな解決策です。

このテーブルを定義するDDLを以下に示します。

このような設計は **隣接リスト** と呼ばれます。

下記はコメントを階層構造で表すためのサンプルデータです。

| comment\_id | parent\_id | comment |
| --- | --- | --- |
| 1 | NULL | このバグの原因は何かな？ |
| 2 | 1 | ヌルポインターのせいじゃないかな？ |
| 3 | 2 | そうじゃないよ。それは確認済みだ。 |
| 4 | 1 | 無効な入力を調べてみたら？ |
| 5 | 4 | そうか、バグの原因はそれだな。 |
| 6 | 4 | よし、じゃあチェック機能を追加してもらえるかな？ |
| 7 | 6 | 了解。修正したよ。 |

### デメリット

- **全ての子孫を取得するクエリが困難**
	- コメントとその直近の子は、比較的単純なクエリで取得できる
	- しかし、このクエリが対象にできるのは2つのみである。ツリーの性質上、深さが制限がない場合が多いため、回想の深さに関わらず子孫を取得するクエリを実行できる必要がある。
- **隣接リストのツリーのメンテナンス問題**
	- 隣接リストでは、ノードの追加やサブツリーの移動は容易にできます。ただし、ノードの削除は簡単ではありません。
	- サブツリーを全体を削除したい場合は外部キー定義時にON DELETE CASCADE修飾子を付けることで自動化できますが、ノードの昇格や、移動は自動化できません。
	- 以下は非葉ノードを削除するときのクエリの例です。
	- このように、本来シンプルかつ効率的に行えるべきことでも、多くのコードが必要になる

## 解決策: 代替ツリーモデルを使用

代替ツリーモデルには3種類存在する。

- ****経路列挙（Path Enumeration）****
	- `Comments` テーブルで、 `parent_id` 列の代わりに `path` 列を大きめの `VARCHAR` として定義する
	| comment\_id | path | comment |
	| --- | --- | --- |
	| 1 | 1/ | このバグの原因は何かな？ |
	| 2 | 1/2/ | ヌルポインターのせいじゃないかな？ |
	| 3 | 1/2/3 | そうじゃないよ。それは確認済みだ。 |
	| 4 | 1/4/ | 無効な入力を調べてみたら？ |
	| 5 | 1/4/5/ | そうか、バグの原因はそれだな。 |
	| 6 | 1/4/6/ | よし、じゃあチェック機能を追加してもらえるかな？ |
	| 7 | 1/4/6/7 | 了解。修正したよ。 |
	- パスに対して比較を行うことによって先祖や子孫の特定を比較的容易に行えます。
	- 経路列挙にはジェイウォークの [デメリット](https://www.notion.so/390d807e6e5d4ee3b2573fe46e7e77f4?pvs=21) と同様な弱点がある。つまり、パスの正確な形式や、パス値の既存ノードへの対応を保証できなくなる。
- ****入れ子集合（Nested Set）****
	- 入れ子は直近の親ではなく、子孫に関数集合に関する情報を各ノードに格納する。
	- 以下の表と図を見ればイメージできるかと思います。
	| comment\_id | nsleft | nsright | comment |
	| --- | --- | --- | --- |
	| 1 | 1 | 14 | このバグの原因は何かな？ |
	| 2 | 2 | 5 | ヌルポインターのせいじゃないかな？ |
	| 3 | 3 | 4 | そうじゃないよ。それは確認済みだ。 |
	| 4 | 6 | 13 | 無効な入力を調べてみたら？ |
	| 5 | 7 | 8 | そうか、バグの原因はそれだな。 |
	| 6 | 9 | 12 | よし、じゃあチェック機能を追加してもらえるかな？ |
	| 7 | 10 | 11 | 了解。修正したよ。 |
	- 入れ子構造の大きな長所は、非葉ノードを削除すると、削除されたノードの子孫は、削除されたノードの親の直接の子であると自動的に見なされることである。
		- 例えば、 `comment_id` が6のデータを削除してもツリー構造には影響がありません。
	- ただし、入れ子集合では、直近の親の取得などの、隣接リストでは簡単に実行できるクエルの一部が複雑になってしまうデメリットが存在する。また、ノードの挿入や、移動などのツリーの操作が他のモデルよりも複雑になる。
	- **入れ子集合が適しているケース**
		- 個々のノードの操作ではなく、サブツリーに対する迅速かつ容易なクエリ実行が重要なケース。
		- ノード挿入や移動は、関連するノードの左右値の再計算が必要になるため、複雑になり、ノードの挿入が頻繁に求められるケースでは、最適とは言えません。
- **閉包テーブル（Closure Table）**
	- Commentsテーブルに加えて、TreePathsテーブルを可惜に定義します。TreePathsテーブルは、それぞれがCommentsテーブルの外部キーである2つの列を持ちます。
	- このテーブルの各行には先祖 / 子孫関係を共有するノードの組み合わせを格納し、ツリー上の離れた位置にあるノードも含めた、全てのノードが対象になる。以下に表と図を示します。
	| 先祖 | 子孫 |
	| --- | --- |
	| 1 | 1 |
	| 1 | 2 |
	| 1 | 3 |
	| 1 | 4 |
	| 1 | 5 |
	| 1 | 6 |
	| 先祖 | 子孫 |
	| --- | --- |
	| 1 | 7 |
	| 2 | 2 |
	| 2 | 3 |
	| 3 | 3 |
	| 4 | 4 |
	| 4 | 5 |
	| 先祖 | 子孫 |
	| --- | --- |
	| 4 | 6 |
	| 4 | 7 |
	| 5 | 5 |
	| 6 | 6 |
	| 6 | 7 |
	| 7 | 7 |
	- **先祖や子孫を取得するクエリが容易**
		- コメントIDが4の子孫を取得するには、TreePathsで先祖が4の行を探す。
		- コメントIDが6の先祖を取得するには、子孫が6の行を探す
	- 新たに葉ノードを挿入する場合
		- 例えばコメントID5の子を挿入するするケース
	- サブツリーを移動する場合
		- 例えば、コメントID6を、コメントID4の子からコメントID3の子の位置に移動したいケース
		- コメントID6の全ての先祖とそれらの子孫を削除する
			- (1, 6)、(1, 7)、(4, 6)、(4, 7)を削除し、(6, 6)、(6, 7)は削除しない。
			```sql
			DELETE FROM TreePaths
			WHERE ancestor IN (
			  SELECT ancestor
			  FROM TreePaths
			  WHERE descendant = 6
			  AND ancestor != descendant
			) -- 1,4
			AND descendant IN (
			  SELECT descendant
			  FROM TreePaths
			  WHERE ancestor = 6
			); -- 6,7
			```
		- 移動先の先祖とサブツリーの子孫の組み合わせを挿入する
			- (1, 6)、(2, 6)、(3, 6)、(1, 7)、(2, 7)、(3, 7)が追加される。
			```sql
			INSERT INTO TreePaths (ancestor, descendant)
			SELECT supertree.ancestor, subtree.descendant
			FROM TreePaths AS supertree
			CROSS JOIN TreePaths AS subtree
			WHERE supertree.descendant = 3
			AND subtree.ancestor = 6;
			```
	- 閉包テーブルの問題点は階層が深くなると、多くの行数が必要になり、スペースが消費されるというトレードオフが生じる。

## まとめ

本書では、閉包テーブルについて下記のように記述されてました。

```sql
「最も用途が幅広く、また唯一、ノードが複数のツリーへ所属することができます。」
```

私自身も閉包テーブルが1番しっくりきました。

## IDリクワイアド（とりあえずID）

## 目的

主キーの規律を確立する

テーブル＝ID列を持たないといけないという勘違いをしてしまうこともある。

## アンチパターン

書籍やフレームワークの影響で、全てのテーブルには以下の特徴を持つ主キー列が存在しなくてはならないという考えが普及している。

- 列名はid
- テータ型は32ビットまたが64ビットの整数
- 一意のあたが自動的に生成される

全てのテーブルにid列を加えると、意図に反した影響が生じることがある。

### 重複行を許可してしまう

複合キーは、複数の列で構成されたキーです。複合キーは、交差テーブルで良く用いられます。

例えば下記の交差テーブルの場合、 `bug_id` と `product_id` の値の組み合わせが、テーブル上で一意であることを保証しなければなりません。

```sql
CREATE TABLE BugsProducts (
  id SERIAL PRIMARY KEY,
  bug_id BIGINT UNSIGNED NOT NULL,
  product_id BIGINT UNSIGNED NOT NULL,
  FOREIN KEY (bug_id) REFERENCES Bugs(bug_id),
  FOREIN KEY (product_id) REFERENCES Products(product_id)
);
```

上記のテーブルの場合下記のように重複を許可してしまいます。

```sql
INSERT INTO BugProducts(bug_id, product_id) VALUES (1234, 1), (1234, 1);
```

重複を防ぐには `id` 以外の2列にUNIQUE制約を宣言することで解決できるかもしれません。

しかし、 `id` 以外の2列にUNIQUE制約が必要ならば、 `id` 列はただの無駄です。

### キーの意味が分かりにくくなる

`code` という単語は、暗号化、すなわち情報を簡潔かつ秘密裏に伝達する方法、という意味を持っています。プログラムを書くときは、意味をより明確にしていこうと「コーディング」しているはずです。

`id` という列名は極めて一般的であるため、明確な意味を持たずクエリの明確化には役立ちません。列名を `bug_id` や `product_id` などにすれば、クエリの結果はずっと分かりやすくなります。

**主キーは「あるテーブル上の1つの行」を識別するためのものなので、主キー列名でもそのテーブルの種類が分かるようにしておくべきです。**

### USINを使用する

予約後ONに続けて、2つのテーブルを結合条件を表す式を書く形式のJOIN構文に、馴染みのある人が多いのではないのでしょうか。

```sql
SELECT * FROM Bugs AS b
INNER JOIN BugsProducts AS bp
ON b.bug_id = bp.bug_id;
```

SQLでは2つのテーブルを結合するための、さらに簡潔な構文をサポートしています。両方のテーブルに同じ名前の列名がある場合、上記のクエリをUSINGを使用して以下のように書き直せます。

```sql
SELECT * FROM Bugs INNER JOIN BugsProducts USING (bug_id);
```

しかし、全てのテーブルで主キーを `id` という名前にしなければならないとすると、従属テーブル側の外部キー列には、参照する主キーと同じ名前は使えません。代わりに、冗長なON構文を常に使用する必要があります。

## アンチパターンを用いても良い場合

ただし、以下のような場合には自然キーを無視して疑似キーを振ってもよい場合があります。

- ORMフレームワークなどでID要求されている場合
- 主キーが物理的に長く、インデックスの作成が効率的でない場合

## 解決策：状況に応じて適切に調整する

- 分かりやすい列名にしよう
	- 主キーが識別する対象のエンティティを表すものにすべき
		- 例えば、 `Bugs` テーブルの主キーの名前は `bug_id` が相応しい
	- 可能ならば、外部キーの列名にも同じような命名規則を用いる
- 規則に縛られない
	- ORMフレームワークの多くは `id` という名前の擬似キーが使われることを規約としている
- 自然キーと複合キーの活用
	- 下記のような交差テーブルのように、行を識別するための最適な方法が複数の属性列である場合、複合主キーを使うと良い
		```sql
		CREATE TABLE BugsProducts (
		  bug_id BIGINT UNSIGNED NOT NULL,
		  product_id BIGINT UNSIGNED NOT NULL,
		  PRIMARY KEY (bug_id, product_id),
		  FOREIN KEY (bug_id) REFERENCES Bugs(bug_id),
		  FOREIN KEY (product_id) REFERENCES Products(product_id)
		);
		```
	- 1つ注意するべき点は複合主キーを参照する外部キーもまた、列の組み合わせでなくてはならない。

## まとめ

規約は、役立つと思える場合のみ従いましょう。

## キーレスエントリ（外部キー嫌い）

こちらは外部キー制約を省略するアンチパターンです。

## アンチパターン：外部キー制約を使用しない

外部キー制約を省略すれば、データベースの設計はシンプルになり、柔軟性が高まり、実行速度が速くなると思っている人もいるかもしれません。しかし、そこには代償があります。つまり、開発者が参照整合性を保証するためのコードを書く必要があります。

### 完璧なコードを前提にしている

参照整合性を保証するためには、データの関連付けを常に維持するためのコードを書くことです。

外部キー制約を設定しなかった場合、参照整合性が損なわれないことを確認する必要があります。

例えば、行を挿入する前に、親の行の存在を確認する必要があります。

`Accounts` テーブルが親、 `Bugs` テーブルが子

```sql
SELECT account_id FROM Accounts WHERE account_id = 1;
```

アカウントが存在する事を確認したのちに、アカウントを参照するレコードを追加できます。

```sql
INSERT INTO Bugs (account_id) VALUES (1);
```

しかし、もし `account_id` が1のアカウントのユーザーが確認クエリを実行した直後に、アカウントの削除をしていたらどうなるのでしょうか。存在しないしないアカウントによって報告されたバグ、という不正なレコードが `Bugs` テーブルに存在することになります。

対処策は、 `Accounts` テーブルを明示的にろっくしながらチェックを行い、バグの登録後にロックを解除することです。しかし、ロックを必要とするアーキテクチャでは、同時接続ユーザーが増え、スケーラビリティが求められるようになるにつれ、様々な問題に直面することもあります。

## 解決策：外部キー制約を宣言する

外部キー制約による参照整合性の強制によって、データ不整合を検出してから修正するのではなく、データベースの登録時点でこれらのミスを阻止できます。

```sql
CREATE TABLE Bugs (
    -- 他の列...
  account_id BIGINT UNSIGNED NOT NULL,
  FOREIN KEY (account_id) REFERENCES Accounts(account_id)
);
```

### 複数テーブルの変更をサポートする

外部キー制約には、アプリケーションコードには真似できない機能もあります。

それは **カスケード更新** です。

カスケード更新とは、外部キー制約にON UPDATE句やON DELETE句を宣言することで、親の行の更新や、削除が可能になり、さらにその行を参照しているあらゆる子の行もデータベースが適切に処理してくれるようになります。

```sql
CREATE TABLE Bugs (
    -- 他の列...
  account_id BIGINT UNSIGNED NOT NULL,
  FOREIN KEY (account_id) REFERENCES Accounts(account_id)
  ON UPDATE CASCADE
  ON DELETE RESTRICT
);
```

### オーバーヘッド、•••••にはなりません

外部キー制約によって、多少のオーバーヘッドが生じるのは事実です。しかし、以下に挙げるように、他の選択肢と比べると、外部キー制約の方がより効率的であることがわかります。

- 挿入、更新、削除の前に、チェックのために `SELECT` クエリを実行する必要がない
- 複数テーブルの変更を防ぐために、テーブルをロックする必要がない
	- もちろんロックが必要になるケースも存在する
- 他の方法のように孤児が生じてしまうことがないので、テータ品質管理用スクリプトを定期的に実行する必要がない

テータベースでもミスの発生を未然に防ぐために、外部キー制約を用いましょう！

## まとめ

データベースのミスの発生を未然に防ぐために、外部キー制約を用いましょう。

## EAV（エンティティ・アトリビュート・バリュー）

## 目的

可変属性をサポートするための設計がしたい。

例えば、 `Bug` テーブルと `FeatureRequest` （機能要望）テーブルが基底型である `Issue` テーブルでの共通の属性を共有しています。各 `Issue` は報告者と関連付けられます。また、 `Issue` は製品とも関連も持っており、作業優先度の属性もあります。しかし、 `Bug` には独自の属性もあります。同じく、 `FaretureRequest` にも独自の属性を持つ可能性があります。

## アンチパターン：汎用的な属性テーブルを使用する

可変属性をサポートする必要がある時、魅力的な解決策だと思えるのは、もう1つの別のテーブルを作成して、属性を「行」に格納することです。

下記のER図では、2つのテーブルが示されています。属性テーブル（今回の例では `IssueAttributes` ）の各行には3つの例があります。

- エンティティ
	- 通常の場合、親テーブルに対応する外部キーです。親テーブルの方では、エンティティ毎に1行が割り当てられています。
- 属性
	- 従来型のテーブルでは、属性は列の名前に相当しますが、この新しい設計では属性名が各行に入って、行ごとに識別する必要あります。
- 値
	- エンティティの属性値です。

```sql
INSERT INTO Issues (issue_id) VALUES (1234);

INSERT INTO IsseuAttributes (issue_id, attr_name, attr_value)
VALUES
  (1234, 'product', '1'),
  (1234, 'date_reported', '2009-06-01'),
  (1234, 'status', 'NEW'),
  (1234, 'description', '保存処理に失敗する'),
  (1234, 'reported_by', 'Bill'),
  (1234, 'version_affected', '1.0'),
  (1234, 'severity', '機能の損失'),
  (1234, 'priority', 'HIGH');
```

例えば、あるバグは、その主キー値 `1234` によって識別されるエンティティです。これは `status` と呼ばれる属性を持ち、バグ `1234` の `status` 属性の値は「NEW」です。

テーブルを新たに作成することで、以下のメリットがあると期待されます。

- 両方のテーブルの列数をは減らせる
- 新たな属性をサポートするために、列数を増やす必要がない
- 属性が存在しないエンティティの該当列にNULLが入っている、NULLだらけのテーブルになることを防げる

しかし、このようにデータベース構造を単純化しても処理の複雑さが解消できません。

### 属性を取得するにはどうするか

例えば、報告さればバグを日別にまとめるレポートを作成する必要があるとします。

従来型のテーブル設計では、 `Issues` テーブルには `date_reported` のような単純な属性列を持つため、下記のような単純なクエリを使えます。

```sql
SELECT issue_id, date_reported FROM Issues;
```

EAV設計では、同じ情報を取得するには、 `IssueAttributes` テーブルから文字列 `date_reported` が格納された行をフェッチする必要があり、クエリが先ほどより冗長になり、明確さも低下します。

```sql
SELECT issue_id, attr_value AS date_reported
FROM IssueAttributes
WHERE attr_name = 'date_reported';
```

### データ整合性をどう保つか

EAVを使用すると、従来型のデータベース設計で得られるいくつもの利点を失いします。

- 必須属性を設定できない
	- 従来型では、NOT NULL制約を宣言するだけで、その列を必須にできます。
- SQLのデータ型を使えない
	- 従来型では、日付のデータにはDATE型で列を定義することができます。
	- 一方でEAV設計では、 `attr_value` 列のデータ型を文字列型になります。
		- あらゆる種類の属性を格納できるようにするためです。
- 参照整合性を強制できない
	- 従来型では、参照テーブルに対する外部キーを定義することによって一部の属性の値を制限できる。
- 属性名を補わなければならない
	- 属性名に一貫性がないケースが存在する
	- 例えば、あるバグには `date_reported` と名付けられた文字列、別のバグでは、 `report_date` という文字列をされる可能性がある。

### 行を再構築しなければならない

従来型の設計であれば、 `Issues` テーブルから1行取得すると、その行に全ての属性が列として格納されます。EAV設計でも、1つのIssueを、あたかも従来型のテーブルに格納されているものであるかのように、1つの行として取得したいところです。

全ての属性を行の1部分として取得するには、各属性の行のJOINが必要になります。クエリの作成時には、属性名の名前を全て指定しなければなりません。以下が例です。

```sql
SELECT i.issue_id,
       i1.attr_value AS date_reported,
       i2.attr_value AS status
FROM Issues AS i
LEFT OUTER JOIN IssueAttributes AS i1
ON i.issue_id = i1.issue_id AND i1.attr_name = 'data_reported'
LEFT OUTER JOIN IssueAttributes AS i2
ON i.issue_id = i2.issue_id AND i2.attr_name = 'status'
WHERE i.issue_id = '1234';
```

属性の1つがテーブルに行として存在していない場合、内部結合（INNER JOIN）を行うと結果0行になってしまうため、外部結合（OUTER JOIN）を使用する必要があります。属性の数が増加すると、結合の数も増加し、このクエリの実行コストも指数関数的に増加します。

## 解決策：サブタイプのモデリングを行う

### シングルテーブル継承

最もシンプルな設計は全てのサブタイプを1つのテーブルに格納することです。加えて1つの属性列を、その行がどのサブタイプであるかを定義するために使用します。

多くの属性はサブタイプ固有のものです。対応する属性を持たないオブジェクトを格納する行にはNULLを入れなくてはなりません。このため、テーブルには非NULLの値を持つ列がバラバラに点在することになります。

```sql
CREATE TABLE Issues (
      issue_id SERIAL PRIMARY KEY,
      reported_by BIGINT UNSIGNED NOT NULL,
      product_id BIGINT UNSIGNED NOT NULL,
      priority VARCHAR(20),
      version_resolved VARCHAR(20),
      status VARCHAR(20),
      issue_type VARCHAR(10), -- 'BUG'または'FEATURE'が格納される
      severity VARCHAR(20), -- Bugのみが使う属性
      version_affected VARCHAR(20), -- Bugのみが使う属性
      sponsor VARCHAR(50),　 -- FeatureRequestのみが使う属性
      FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
      FOREIGN KEY (product_id) REFERENCES Proudcts(product_id)
);
```

- **デメリット**
	- テーブルによっては列数の実際的な上限に達するかもしれない。
	- どの属性がどのサブタイプに所属するかを定義するメタデータが存在しない。
- **採用が適切なケース**
	- サブタイプの数とサブタイプ固有の属性の数が少なく、\*\*アクティブレコード（Active Record）\*\*のような単一のテーブルに対するデータベースアクセスパターンを使う必要がある場合。

### 具象テーブル継承

サブタイプ毎にテーブルを作成する方法です。基底型に共通する属性と、それぞれのサブタイプに固有の属性を含んでいます。

```sql
CREATE TABLE Bugs (
    issue_id SERIAL PRIMARY KEY,
    reported_by BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    priority VARCHAR(20),
    version_resolved VARCHAR(20),
    status VARCHAR(20),
    severity VARCHAR(20), -- Bugのみが使う属性
    version_affected VARCHAR(20), -- Bugのみが使う属性
    FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
    FOREIGN KEY (product_id) REFERENCES Proudcts(product_id)
);

CREATE TABLE FeatureRequests (
    issue_id SERIAL PRIMARY KEY,
    reported_by BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    priority VARCHAR(20),
    version_resolved VARCHAR(20),
    status VARCHAR(20),
    sponsor VARCHAR(50),　 -- FeatureRequestのみが使う属性
    FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
    FOREIGN KEY (product_id) REFERENCES Proudcts(product_id)
);
```

シングルテーブル継承と比較した場合のメリットとして、サブタイプに存在しない属性列を格納する必要がないという点です。

しかし、共通属性に新しい属性を加える場合、全てのサブタイプのテーブルを変更しなければなりません。また、サブタイプテーブルに格納されたデータが、基底型とサブタイプのどちらに属しているかを示すメタデータはありません。

### クラステーブル継承

テーブルをオブジェクト指向のクラスであるかのように見なして、継承を模倣するという方法です。まず、全てのサブタイタイプに共通する属性を含む基底型のテーブルを1つ作ります。次に、サブタイプ毎に1つずつ追加のテーブルを作成し、基底型テーブルに対する外部キーの役割を持つ主キーを設定します。

```sql
CREATE TABLE Issues (
    issue_id SERIAL PRIMARY KEY,
    reported_by BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    priority VARCHAR(20),
    version_resolved VARCHAR(20),
    status VARCHAR(20),
    FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
    FOREIGN KEY (product_id) REFERENCES Proudcts(product_id)
);

CREATE TABLE Bugs (
    issue_id BIGINT UNSIGNED PRIMARY KEY,
    severity VARCHAR(20),
    version_affected VARCHAR(20),
    FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
);

CREATE TABLE FeatureRequests (
    issue_id BIGINT UNSIGNED PRIMARY KEY,
    sponsor VARCHAR(50),
    FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
);
```

メタデータによって、1体1の関連が強制されます。全てのサブタイプにまたある検索を効率良く行うことができます。

```sql
SELECT i.*, b.*, f.*
FROM Issues AS i
LEFT OUTER JOIN Bugs AS b USING (issue_id)
LEFT OUTER JOIN FeatureRequests AS f USING (issue_id);
```

クラステーブル継承は、全てのサブタイプに共通する列を参照するクエリが頻繁に実行されるときに適しています。

私自身はこの方法が一番汎用性もあり、良いかと思っております。

### 半構造化データ

サブタイプの数が多い場合や、頻繁に新しい属性を追加しなければならない場合は、LOB列を追加し、XMLやJSONなどの形式で属性名と値を共に格納することもできます。

```sql
CREATE TABLE Issues (
    issue_id SERIAL PRIMARY KEY,
    reported_by BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    priority VARCHAR(20),
    version_resolved VARCHAR(20),
    status VARCHAR(20),
    issue_type VARCHAR(10), -- 'BUG'または'FEATURE'が格納される
    attributes TEXT NOT NULL, -- その他の動的属性が格納される
    FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
    FOREIGN KEY (product_id) REFERENCES Proudcts(product_id)
);
```

- **長所**
	- 拡張性が極めて高いこと
	- 全ての行ごとに異なるサブタイプを作ることも可能
- **短所**
	- SQLが特定の属性にアクセスする手段をほとんど持っていない
	- 行の絞り込み、集約計算、ソートなどの処理をアプリケーションコードを書く必要がある

この設計は、サブタイプの数を制限できない場合や、新しい属性を随時定義するための高い柔軟性が必要な場合に適しています。

## まとめ

メタデータは、メタデータのために用いましょう。

## ポリモーフィック関連

## 目的

複数の親テーブルを参照したい。以下のようなイメージです。

## アンチパターン：二重目的の外部キーを使用する

### ポリモーフィック関連を定義する

ポリもーフィック関連を機能させるには、 `issue_id` のような外部キー列に加えて、文字列型の列 `issue_type` を追加します。以下が例です。

上記のテーブルでは、 `issue_id` のための外部キー宣言がありません。外部キーでは、テーブルを1つのみ指定しなければならない（複数のテーブルを指定できない）ため、ポリもーフィック関連を使用しているときは、メタデータで関連付けを宣言できないため、参照整合性制約を定義できません。

### 非オブジェクト指向の例

上記の例では、2つの親テーブル（ `Bugs` と `FeatureRequests` ）は同じモデルから継承したサブタイプを表していました。ポリモーフィック関連は、親テーブル同時に全く関係がない場合にも使用されます。例えば、顧客（ `Users` ）と注文（ `Orders` ）の2つのテーブルには、住所（ `Addresses` ）と関連付けられます。

`Addresses` テーブルは `Users` と `Orders` のどちらか1つを選択しなければならないので、注文商品の出荷先が顧客自身の住所であっても、両方に同じ住所の行を関連付けることはできません。

また、顧客が出荷先住所だけでなく、請求先住所を持つ場合、 `Addresses` テーブルでそれを区別する方法が必要です。

## 解決策：関連（リレーションシップ）を単純化する

### 参照を逆にする

問題の本質が何かを考えるとすぐに分かります。すなわち、ポリモーフィック関連では「 **本来あるべき関連が、逆さまになっている** 」のです。

### 交差テーブルを作成

子テーブル側である `Comments` の外部キーでは、複数の親テーブルを参照できません。代わりに、複数の外部キーを、 `Comments` テーブルを参照するために使用しましょう。複数の親テーブルそれぞれに対応した交差テーブルを作成し、各テーブルでは、 `Comments` への外部キーに加えて、各親テーブルへも同じく外部キーを定義します。以下は例です。

### 交差点に交差信号を設置する

この解決策の潜在的な弱点は、許可したくない関連付けが許可されてします可能性がある点です。交差テーブルでは、多対多の関連付けを作成するので、コメントを複数のバグまたは機能要求と関連付けることも可能です。各交差テーブルの `comment_id` 列にUNIQUE制約を宣言することで、少なくともこのルールの一部を強制することはできます。

### 共通の親テーブルを作成

下記のように基底テーブルへの `Comments` の関連付けを行います。

テーブル間の関連（リレーションシップ）には、参照元テーブルと参照先テーブルが常にそれぞれ1つしかないことを忘れないようにしましょう。

- **クエリの実行例**
	- 特定のコメントが参照しているバグまたは機能要求を取得
		```sql
		SELECT * 
		FROM Comment AS c 
		LEFT OUTER JOIN Bugs USING (issue_id)
		LEFT OUTER JOIN FeatureRequests USING (issue_id)
		WHERE c.comment_id = 2;
		```
	- 特定のバグのコメントを取得

## まとめ

テーブル間のリレーションシップには、参照もとテーブルと参照先テーブルが常にそれぞれ1つしかないことを忘れないようにしましょう。

## マルチカラムアトリビュート（複数列属性）

## 目的

1つのテーブルに属するべきだと思える属性に複数の値がある場合、それをどのように格納するかという問題です。

## アンチパターン：複数の列を定義する

ジェイウォークアンチパターンで見たように、複数値をカンマ区切りで1列に格納するべきではあります。 [カンマ区切りフォーマットのリストを格納する](https://www.notion.so/0a3213b2e647444abc958869bc47369e?pvs=21)

各列には、値を1つのみ格納すべきなので、下記のようにそれぞれ1つのタグを格納する列を複数作成することが、自然な選択のように思えます。

```sql
CREATE TABLE Bugs (
    bug_id SERIAL PRIMARY KEY,
    description VARCHAR(1000),
    tag1 VARCHAR(20),
    tag2 VARCHAR(20),
    tag3 VARCHAR(20)
);
```

表: 未使用の列はNULLのままにする

| bug\_id | description | tag1 | tag2 | tag3 |
| --- | --- | --- | --- | --- |
| 1234 | 保存処理でクラッシュする | crash | NULL | NULL |
| 3456 | パフォーマンスの向上 | printing | performance | NULL |
| 5678 | XMLのサポート | NULL | NULL | NULL |

しかし、、この設計には問題があります。

### 値の検索

例えば、特定のタグが付けられたバグを検索しようとすると、3列全てを取得しなければなりません。タグ文字列は3つの列のどれにでも格納される可能性があるためです。

```sql
SELECT * FROM Bugs
WHERE tag1 = 'performance'
OR tag2 = 'performance'
OR tag3 = 'performance';
```

`performance` と `printing` の両方のタグを付いたバグを検索するとします。検索には以下のようなクエリを用います。ORはANDよりも優先度が低いため、括弧の使い方には注意が必要です。

```sql
SELECT * FROM Bugs
WHERE (tag1 = 'performance' OR tag2 = 'performance' OR tag3 = 'performance')
AND (tag1 = 'printing' OR tag2 = 'printing' OR tag3 = 'printing');
```

複数の列から1つの値を検索するという単純なものが手間暇かかるものになってしました。（IN述語を使用すればコンパクトにすることはできますが、どちらにせよ手間暇かかるものになります）

### 値の更新

単純にUPDATEを用いて、1列のみを対象とした変更を行おうとしても、それは安全とはいません。どの列が空いているかを確認できないからです。確認するために、対象の行を取得する必要があります。

### 一意性の保証

複数列の列に同じ値を格納したくはありません。しかし、それを防ぐことができません。

### 増加する値の処理

列数が3列では足りなくなるかもしれないという点です。列ごとに1つ値を格納するという設計を続けるにはバグに与えられるタグ数の最大値とどう数の列を定義しなければなりません。しかし、テーブルの定義時にタグ数の最大値を予測するのは困難です。

## 解決策：従属テーブルを作成する

属性を格納する列を1つ持つ従属テーブルを作成し、属性を格納することです。

```sql
CREATE TABLE Tags (
    bug_id BIGINT UNSIGNED NOT NULL,
    tag VARCHAR(20),
    PRIMARY KEY (bug_id, tag),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);

INSERT INTO Tags (bug_id, tag) VALUES (1234, 'crash'), (3456, 'printing'), (3456, 'performance');
```

個々のバグに付けられたタグは全て従属テーブルの1つの列に格納されるので、あるタグを付けられたバグの検索は簡単になります。

```sql
SELECT * FROM Bugs INNER JOIN Tags USING (bug_id) WHERE tag = 'performance';
```

特定の2つのタグが付けられたバグ検索といった少し複雑な処理も簡単に記述できます。

```sql
SELECT * FROM Bugs
INNER JOIN Tags AS t1 USING (bug_id)
INNER JOIN Tags AS t2 USING (bug_id)
WHERE t1.tag = 'printing' AND t2.tag = 'performance';
```

## まとめ

同じ意味を持つ値は、1つの列に格納するようにしましょう。

## メタデータトリブル（メタデータ大増殖）

## 目的

どのようなデータベースクエリでも、データ容量が増えるにつれてパフォーマンスは低下します。インデックを賢く使用することで状況は改善しますが、それでもデータは増え続け、いつかはクエリの実行速度に影響を与えます。

メタデータトリブルの目的は、クエリの実行速度を劣化させずに、データが増加し続けるテーブルに対応できるよう、データベースの構造を設計することです。

## アンチパターン：テーブルや列をコピーする

行数が少ないテーブルへのクエリ実行の方が、行数が多い場合よりも早く処理できるかと思います。しかしこれは、求めている処理が何かにかかわらず、「すべてのテーブルの行は少ない方がいい」という誤った考えを導く危険があります。その結果、以下の2つのアンチパターンに陥ってしまします。

- 行数の多いテーブルを、複数のテーブルに分割する（あるテーブルの属性の区別しやすいデータ値に基づいてテーブルを命名する）
- 列を複数列に分割する（別の属性の区別しやすい値に基づいて列を命名する）

```sql
CREATE TABLE Bugs_202310(...);
CREATE TABLE Bugs_202311(...);
CREATE TABLE Bugs_202312(...);
```

## 解決策：パーティショニングと正規化を行う

テーブルサイズが巨大化した場合に、手作業でテーブルを分割せず、 **水平パーティショニング** 、 **垂直パーティショニング** 、 **従属テーブルの導入** などです。

### 水平パーティショニングの使用

水平パーティショニングあるいは **シャーディング** とも呼ばれます。

行を分割するいくつかのルールを定めて論理テーブルを定義すれば、あとはデータベースが必要な作業を行なってくれます。テーブルは物理的には分割されていますが、あたかも1つのテーブルを扱うように、SQLステートメントを実行できます。

```sql
CREATE TABLE Bugs (
    bug_id SERIAL PRIMARY KEY,
    -- 他の列・・・
    date_reported DATE
) 
PARTITION BY HASH ( YEAR(date_reported) )
PARTITIONs 4;
```

## 垂直パーティショニングの使用

水平パーティショニングがテーブルを行で分割するのに対し、 **垂直パーティショニング** は列でテーブルを分割します。列でもテーブル分割は、列の1部のサイズが大きい場合や、めったに使用されない場合にメリットがあります。

## まとめ

データにメタデータを増殖させないように気をつけましょう。

## ラウンディングエラー（丸め誤差）

## 目的

整数以外の数値を格納し、数学的な計算を行うことが目的です。

## アンチパターン：FLOATデータ型を使用する

SQLのFLOATデータ型は、他のプログラミング言語のfloat型と同じように、IEEE754標準に従って実巣を2進数でエンコードします。

### 丸めが避けられない

正確な値は少数では表現できません。なぜなら桁数を無限に書く必要があるからです。つまり循環小数では表現できません。

$$
\frac{1}{3}+\frac{1}{3}+\frac{1}{3}=1.00
$$
 
$$
0.333+0.333+0.333=0.999
$$

IEEE754では浮動小数点を2進数で表現されます。

### SQLでのFLOATの使用

```sql
SELECT hourly_rate FROM Accounts WHERE account_id = 123;

結果：59.95
```

しかしFLOAT列に格納された実際の値は、正確にはこの値とは異なるかもしれません。例えば値を10億倍すると、差異があることがわかります。

```sql
SELECT hourly_rate * 1000000000 FROM Accounts WHERE account_id = 123;

結果：59950000762.939
```

このクエリで返す値が、 `59950000000.000` であると予想したかもしれません。今見た結果は、値59.59がIEEE754の2進数形式の有限精度で表せる値に丸められたいたことを示します。

この場合の誤差は1000万分の1以下であり、多くの計算では十分な精度だといえますが、計算によっては、求めている結果を得られない場合があります。

```sql
SELECT * FROM Accounts WHERE hourly_rate = 59.95;

結果：一致する行はありません。
```

## 解決策：NUMERICデータ型を使用する

FLOATやその類似したデータ型の代わりに、SQLデータ型の `NUMERIC` または `DECIMAL` を用いて固定精度の少数点数を表すようにする。

```sql
ALTER TABLE Accounts ADD COLUMN hourly_rate NUMERIC(9,2);
```

精度はVARCHARデータ型の長さの指定で用いる構文と同様に、データ型への引数として指定します。精度（precision）とは、この列の値として使用可能な10進数の桁の総数です。精度に9を指定すると `123456789` のような値は格納できますが、 `1234567890` は格納できません。

第2引数にはスケールを指定します。スケール（scale）とは小数点以下に格納できる桁数です。スケールは精度の桁数に含まれるため、精度が9で、スケールを2を指定した場合は、 `1234568.89` は格納できるが、 `12345678.91` や `123456.789` のような値は格納できません。

- 長所
	- FLOATデータ型とは異なり、有理数を丸めることなく格闘できる
	- 格納された値とリテラル値59.59との等価性を比較すると、比較は成功する

しかし、3分の1のような、無限精度が必要な値を可能することはできません。

## まとめ

できる限り、FLOAT型は使わないようにしましょう。

## サーティワンフレーバー（31のフレーバー）

## 目的

列に登録できる値を特定の値に限定することが目的です。

## アンチパターン：限定する値を列定義で指定する

多くの人が、有効なデータ値を列の定義時に指定するという方法をとります。

```sql
CREATE TABLE Bugs (
    -- 他の列...
    status VARCHAR(20) CHECK (status IN ('NEW', 'IN PROGRESS', 'FIXED'))
);
```

MySQLでは、ENUMと呼ばれる非標準のデータ型をサポートしています。

```sql
CREATE TABLE Bugs (
    -- 他の列...
    status ENUM('NEW', 'IN PROGRESS', 'FIXED')
);
```

### 中身はなんだろう

例えば、ステータス値の選択肢をドロップダウンリストで表示したい場合、 `status` 列に現在許可されている列挙値を所得するにはどのようなクエリを実行すれば良いでしょうか。

おそらく、以下のようなシンプルなクエリを思いついたのではないでしょうか。

```sql
SELECT DISTINCT status FROM Bugs;
```

しかし、現時点の全てバグのステータスが `NEW` である場合、このクエリからはNEWしか返さないことが分かります。

### 新しいフレーバーの追加

最も一般的な変更は、有効値の追加または削除です。しかし、 `ENUM` の値や `CHECK` 制約を追加または削除するための構文は無く、新たな値セットで列を再定義するしか方法はありません。

```sql
ALTER TABLE Bugs MODIFY COLUMN status ENUM('NEW', 'IN PROGRESS', 'FIXED', 'DUPLICATE');
```

### 昔ながらの味は色褪せない

例えば、従来の `FIXED` を `CODE COMPLETE` と `VERIFIED` の2つのステージに分けるとします。

```sql
ALTER TABLE Bugs MODIFY COLUMN status ENUM('NEW', 'IN PROGRESS', 'CODE COMPLETE', 'VERIFIED');
```

列挙値から `FIXED` を削除する場合、statusが `FIXED` になっている既存のレコードはどう扱えば良いのでしょうか。

## 解決策：限定する値をデータで指定する

参照テーブル `BugStatus` を作成し、許可する値を1行1つずつstatus列に格納します。外部キー制約を宣言します。

```sql
CREATE TABLE BugStatus (
    status VARCHAR(20) PRIMARY KEY
);
INSERT INTO BugStatus (status) VALUES ('NEW'), ('IN PROGRESS'), ('FIXED');

CREATE TABLE Bugs (
    -- 他の列...
    status VARCHAR(20),
    FOREIGN KEY (status) REFERENCES BugStatus(status) ON UPDATE CASCADE
);
```

これで、Bugsデーブルに行の挿入や更新を行うときには、BugStatusテーブルに存在するstauts値を使わなくてはならないようにできました。

### 参照テーブルの値の更新

参照テーブルを使用すれば、普通のINSERT分で値を追加できます。

```sql
INSERT INTO BugStatus (status) VALUES ('DUPLICATE');
```

外部キーに `ON UPDATE CASCADE` オプションを指定して宣言すれば、値の名前の変更も簡単に行えます。

```sql
UPDATE BugStatus SET status = 'INVALID' WHERE status = 'BOGUS';
```

### 廃止された値のサポート

Bugsテーブルの行から参照されている場合、参照テーブルの行は削除できません。status列の外部キーが参照整合性を矯正するため、値は参照テーブルに存在しなくてはないからです。

廃止された値を区別するためには、参照テーブルに新たに、属性列を追加できます。これにより、履歴系データを保管できるようになります。

```sql
ALTER TABLE BugStatus ADD COLUMN active ENUM('INACTIVE', 'ACTIVE') NOT NULL DEFAULT 'ACTIVE';
```

値を排するには、DELETEではなく、UPDATEを用います。

```sql
UPDATE BugStatus SET active = 'INACTIVE' WHERE status = 'DUPLICATE';
```

ユーザーインタフェースに表示する値を取得するには、 `active` 列が `ACTIVE` のステータス値のみ取得します。

```sql
SELECT status FROM BugsStatus WHERE active = 'ACTIVE';
```

### 移植が容易

ENUMデータ型、CHECK制約と異なり、参照テーブルを用いて解決策は、外部キー制約を用いた参照整合性を宣言という標準なSQL機能を利用したものであるため、移行が容易です。

## まとめ

列に入力する値を限定するときには、値をセットが固定されている場合はメタデータを、流動的な場合はデータを用いましょう。

## ファントムファイル（幻のファイル）

## 目的

画像をはじめとする大容量のメディアファイルを格納し、ユーザーアカウントのようなデータベースエンティティと結びつけることが目的です。

## アンチパターン：物理ファイルの使用を必須を思い込む

```sql
CREATE TABLE Accounts (
    account_id SERIAL PRIMARY KEY,
    account_name VARCHAR(20),
    portraite_image BLOB
);
```

同様に同じタイプの複数の画像を従属テーブルに格納できます。

```sql
CREATE TABLE Screenshots (
    bug_id BIGINT UNSIGNED NOT NULL,
    image_id BIGINT UNSIGNED NOT NULL,
    screenshot_image BLOB,
    caption VARCHAR(100),
    PRIMARY KEY (bug_id, image_id),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);
```

画像のデータ型を選択するとなると、意見が分かれれます。画像のバイナリデータは、、BLOBデータ型に格納できます。しかし、BLOBを用いず、画像をファイルシステムにファイルとして格納し、ファイルパスを `VARCHAR` としてデータベースに格納する人もいます。

```sql
CREATE TABLE Screenshots (
    bug_id BIGINT UNSIGNED NOT NULL,
    image_id BIGINT UNSIGNED NOT NULL,
    screenshot_path VARCHAR(100),
    caption VARCHAR(100),
    PRIMARY KEY (bug_id, image_id),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);
```

どちらの解決策にも、長所があります。ファイルを常にデータベースの外部に格納すべきであるという考えで意見が一致しています。しかし、外部にファイルを格納するという設計には、いくつもの大きなリスクがあります。

### ファイルの削除における問題

画像がデータベースの外である場合、データベースで画像へのパスを含む行を削除しても、そのパスの指定先のファイルは自動的に削除されません。

## トランザクションの分離問題

通常、データの更新や削除においては、トランザクションをCOMMITするまで変更は他のクライアントには見えません。しかし、ファイルがデータベースの外部にある場合、ファイルを削除すると、トランザクションがコミットされる前に、他のクライアントはその変更されたファイルを目にすることになります。

### データベースのバックアップツール使用時における問題

バックアップツールには、VARCHAR列に格納されたパス名が参照する先のファイルをバックアップ対象にする方法がわかりません。このため、データベースをバックアップする際、2段階のプロセスを行う必要が出てきます。

### SQLアクセス権限使用時における問題

外部ファイルには、 `GRANT` や `REVOKE` などのsqlステートメントで割り当てるアクセス権限が適用されません。SQLにおけるアクセス権限は、テーブルや列へのアクセスを管理しますが、データベース内で文字列で指定された外部ファイルを対象にすることはできません。

### ファイルはSQLデータ型ではない

`screenshot_path` に格納されたパスは単なる文字列です。データベースは、その文字列が正当なパス名であることを検証しません。同じく、データベースは指定したパスにファイルが存在することを検証できません。ファイル名が変更されたり、ファイルの緯度移動や削除が行われても、データベース内の文字列は自動的には更新されてません。

データベースの長所の1つは、デーl他の整合性を維持してくれることです。データを外部ファイルに格納すると、この利点が失われてしまうだけではなく、データベースで処理すべきチェックを行うためにアプリケーションコードを書かなければなりません。

## アンチパターンを用いても良い場合

画像のようなデータサイズの大きいオブジェクトをデータベース外部のファイルに格納することには、いくつもの正当な理由があります。

- データベースの容量を減らせる。
- データベースのバックアップが短時間で終了し、バックアップファイルの容量も抑えれる。
- プレビューや編集が容易になる。
	- 全ての画像を一括で修正する場合、画像がデータベースの外部に格納されていれば、処理が極めてやりやすくなります。

これらの利点が特に重要であり、問題点も深刻なものにならないと判断できる場合は、画像をデータベースの外部に格納する方法を採用しても良いでしょう。

また、データベース製品によっては、外部ファイルをある程度透過的に参照できる特殊なSQLデータ型をサポートしています。このデータ型は、Oracleでは `BFILE` 、SQL Server 2008では、 `FILESTREAM` と呼ばれます。

## 解決策：必要に応じてBLOB型を採用する

上記で述べた問題のいずれかが該当する場合は、画像を外部ファイルではなく、データベースの内部に格納することを考えるべきです。

```sql
CREATE TABLE Screenshots (
    bug_id BIGINT UNSIGNED NOT NULL,
    image_id BIGINT UNSIGNED NOT NULL,
    screenshot_image BLOB,
    caption VARCHAR(100),
    PRIMARY KEY (bug_id, image_id),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);
```

画像を `BLOB` 列に格納する場合、全ての問題を解決できます。

`BLOB` の最大サイズはデータベース製品によって異なりますが、いずれの場合も、ほとんどの画像を格納するための十分な容量があります。

多くの場合、データベースに格納する前の元画像はファイルとして存在していると思いますが、データベースには `BLOB` 列に読み込むための方法が必要です。いくつかのデータベースには、外部ファイルをロードする関数を提供しているものがあります。例えば、MySQLの `LOAD_FILE` 関数は、ファイルを読み込むと、内容の `BLOB` 列への可能に使用できます。

```sql
UPDATE Screenshots
SET screenshot_image = LOAD_FILE('image/screenshot1234-1.jpg')
WHERE bug_id = 1234 AND image_id = 1;
```

`BLOB` 列の内容をファイルに保存することも可能です。例えば、MySQLには `SELECT` ステートメントのオプション区があり、列名や行の終了などの装飾を付けずに、クエリ結果をそのままファイルに格納できます。

```sql
SELECT screenshot_image
INSERT DUMPFILE 'image/screenshot1234-1.jpg'
FROM Screenshots
WHERE bug_id = 1234 AND image_id = 1;
```

## まとめ

データベース外部のリソースは、データベースでは管理できないことに注意しましょう。

## インデックスショットガン（闇雲インデックス）

## 目的

パフォーマンスの問題は、データベース開発者にとって最大のテーマだと言っていいでしょう。パフォーマンスを改善する最善の方法は、インデックスを効果的に使用することです。効果的なインデックの使用方法をマスターすることが目的です。

## アンチパターン：闇雲にインデックスを使用する

インデックスを使用するか否かの判断をインデックスを理解しないまま行うと、以下の3つのミスのどれかが起きます。

- インデックスを全く定義しないか、少ししかインデックするを定義しない
- インデックスを多く定義し過ぎるか、役立たないインデックスを定義する
- インデックスを活用しないクエリを実行する

### インデックスを全く定義しない

インデックスの更新によってデータベースにオーバーヘッドが生じるため、一部の開発者は、そのオーバーヘッドを排除しようとします。つまり、インデックスそのものを使用しなければ良いと考えてます。しかしこれでは、インデックスにはオーバーヘッドを正当化するだけのメリットがあるという事実を見逃してしますことになります。

通常のアプリケーションでは、テーブルに対するクエリ発行回数の方が、テーブルの更新回数よりも多いものです。インデックス用いてクエリを実行するたびに、インデックスの維持のために生じたオーバーヘッドを取り戻します。

またインデックスは、目的の行を素早く見つけられるという点で、UPDATEやDELETEステートメントにも役立ちします。

インデックスを定義しない列で値を検索するステートメントでは、テーブル全体を検索しなければなりません。

### インデックスを多く定義し過ぎる

インデックスのメリットを得られるのは、インデックスを使うクエリを実行するときです。使用されないインデックスを作るメリットはありません。

```sql
CREATE TABLE Bugs (
    bug_id SERIAL PRIMARY KEY,
    date_reported DATE NOT NULL,
    summary VARCHAR(80) NOT NULL,
    status VARCHAR(10) NOT NULL,
    hours NUMERIC(9,2),
    INDEX (bug_id) --- ①
    INDEX (summary) --- ②
    INDEX (hours) --- ③
    INDEX (bug_id, date_reported, status) --- ④
);
```

1. 主キーのインデックスを自動的に作成されるので、明示的に定義するのは冗長です。
2. `VARCHAR(80)` などの長い文字列を格納するデータ型へのインデックスは、コンパクトなデータ型のインデックスと比べてサイズが大きくなります。
3. `hours` 列に対して検索や、ソートを実行することはあまり考えられません。
4. 複合インデックスにはいくつものメリットがあります。しかし、多くの場合は冗長であったり、使用頻度が極めて低くなってしまいがちです。また、複合インデックスでは列の順序がとても重要です。検索、条件、結合条件、ソート順において、列を定義した順に使用する必要があります。

テーブル全てにインデックスを作成してもメリットが得られる保証はなく、オーバーヘッドを増やしてしますことになります。

## 解決策：「MENTOR」の原則に基づいて効果的なインデックス管理をする

**MENTOR** とはMeasure、Explain、Nominate、Test、Optimize、Rebuildの頭文字をとったものです。

### Measure（測定）

情報がなけらば、情報に基づく判断はできません。ほとんどのデータベースには、SQLのクエリ実行時間を記録する方法があります。

- Microsoft SQL ServerとOracleには、SQLのトレース機能とトレース結果のレポートと分析のためのツールがあります。SQL Server Profiler、TKPROFなどです。
- MySQLとPostgreSQLは、指定された閾値より実行時間が長くかかったクエリを記録できます。MySQLでは、 **スロークエリログ** と呼ばれます。設定パラメータ `long_query_time` のデフォルト値は10秒です。PostgreSQLにも、類似した設定変数 `log_min_duration_statement` があります。またクエリログの文節を支援するツールpgFouineもあります。

最も多くの時間を消費するクエリを特定していれば、最適化で最大のメリットを得るために、どこに注目すべきかがわかります。ボトルネックになっているのは1つのクエリのみで、他のクエリは効率的に実行されているかもしれません。その場合には、最も遅いクエリからまず最適化に着手します。

### Explain（解析）

最もコストがかかるクエリを特定した後は、クエリの処理が遅くなっている原因を解析します。

ベースはクエリ実行計画（Query Execution Plan: QEP）と呼ばれるクエリ最適化機能によって、クエリ実行にどのインデックスを使うを判断しています。このQEPの分析結果のレポートを取得します。

| データベース | QEP レポート機能 |
| --- | --- |
| IBM DB2 | EXPLAIN、db2explnコマンドまたはVisual Explain |
| Microsoft SQL Server | SET SHOWPLAN\_XMLまたはDisplay Execution Plan |
| MySQL | EXPLAIN |
| Oracle | EXPLAIN PLAN |
| PostgreSQL | EXPLAN |
| SQLite | EXPLAN |

### Nominate（指名）

クエリのQEPを確認して、クエリがインデックスを使用しないでテーブルにアクセスしている箇所を探しましょう。

一部のデータベースには、クエリ能登レース統

- IBM DB2 Design Advisor
- Microsoft SQL Server Database Engine Tuning Advisor
- MySQL Enterprise Query Analyzer
- Oracle SQL Tuning Advisor

自動的な提案機能がなくても、インデックスがどのような場合にクエリに役立つかを見分ける方法を学ぶことができます。QEPレポートを解釈するために、データベースのドキュメントをよく読みましょう。

### Test（テスト）

テストは重要なステップです。インデックスの作成後、再びクエリのプロファイリングを行うです。大事なのかは、変更が効果をもたらしことを確認してから作業を終了することです。

### Optimize（最適化）

インデックスはコンパクトで、使用頻度の高いデータ構造であるため、キャッシュメモリに格納されやすくなります。メモリ上のインデックスにアクセスすることによって、ディスクI/Oを伴う読み込みよりもはるかにパフォーマンスを改善できます。

データベースサーバーは、キャッシュに割り当てるシステムメモリの量を設定できます。キャッシュに割り当てるメモリの量は、絶対的な答えはありません。データベースのサイズと、利用できるシステムメモリの量によって異なるからです。

使用頻度の高いデータやインデックスのキャッシュへの格納をデータベースに依存することなく、あらかじめキャッシュメモリにロードしておくことでメリットが得られる場合もあります。MySQLでは、 `LOAD INDEX INTO CACHE` ステートメントを使用します。

### Rebuild（再構築）

インデックスはバランスが取れているときに、最も効果的です。長期に渡って行の更新や削除を行うことで、インデックスは次第に不均等になっていきます。できる限りインデックスの効率を高めたいのであれば、定期的にメンテナンスを実施する価値はあります。

インデックスに関数ほとんどの機能と同様に、各種データベース製品では、インデックスのメンテナンスについて、ベンダー固有の用語、構文、機能を使用しています。

| データベース | インデックスのメンテナンスコマンド |
| --- | --- |
| IBM DB2 | REBUILD INDEX |
| Microsoft SQL Server | ALTER INDEX … REORGANIZE、ALTER INDEX … REBUILD、またはDBCC DBREINDEX |
| MySQL | ANALYZE TABLEまたはOPTIMIZE TABLE |
| Oracle | ALTER INDEX … REBUILD |
| PostgreSQL | VACUUMまたはANALYZE |
| SQLite | VACUUM |

### まとめ

インデックスの効果的に高めるには必要な情報はベンダーによって異なるため、使用しているデータベース製品の詳しい知識が必要です。データベースマニュアル、書籍や雑誌、ブログやメーリングリストなどからも積極的に情報収集しましょう。最も重要なルールは、 **推測のみに基づいて、闇雲にインデックスをつけてはならない** ということです。

## フェア・オブ・ジ・アンノウン（恐怖のunknown）

## 目的

NULL列を含む列に対してクエリを書くことが目的です。

## アンチパターン：NULLを一般値として使う、または一般値をNULLとして使う

### 式でNULLを扱う

例えば、以下の場合、 `hours` 列がNULLの場合は結果が10になると予想するでしょう。

```sql
SELECT hours + 10 FROM Bugs;
```

NULLはゼロと同じではあります。

NULLは長さゼロの文字列でもありません。SQL標準では、文字列とNULLを連結するとNULLが返されます。（ただし、OracleとSybaseでは振る舞いが異なります。）

### NULLを許容する列の検索

以下のクエリは `assigned_to` 列に値123を持つ行のみを返します。

```sql
SELECT * FROM Bugs WHERE assigned_to = 123;
```

ということは以下のクエリは先程のクエリの補集合、つまり上記のクエリで返されなかった行が全て返されるのではないかと思います。

```sql
SELECT * FROM Bugs WHERE NOT (assigned_to = 123);
```

しかし、このクエリも `assigned_to` 列にNULLが割り当てられた行を返しません。NULLを用いた比較は、TRUEやFALSEではなく、全て不明（unknown）を返します。NULLの否定ですらNULLのままです。そのため以下のクエリでも `assigned_to` 列にNULLが割り当てられた行を返しません。

```sql
SELECT * FROM Bugs WHERE assigned_to = NULL;
SELECT * FROM Bugs WHERE assigned_to <> NULL;
```

### プリペアドステートメントでNULLを扱う

また、プリペアドステートメントでパラメータ化したSQLでNULLを一般値のように扱うことも困難です。

```sql
SELECT * FROM Bugs WHERE assigned_to = ?;
```

このクエリはパラメータに一般的な整数値を渡す時のみ予期する結果を返します。

### NULLの使用を避ける

NULLの扱いでクエリが複雑化することによって、NULLの代わりに不明（unknown）または適用不明（inapplicable）を意味する値を新たに定義するのです。

しかし、この対応では以下のようなデメリットが発生します。

- NOT NULL制約がうまく動作しない
- INT型などの数値の場合はどのように対応するのか
	- `-1` という負の値を選択した場合、 `SUM` や `AVG` のような計算を混乱させます。
- 外部キーの場合どのような非NULLの値を使えるのでしょうか？
	- 実際の参照がないことを表すために参照用のレコードを追加するのも、皮肉なことです。

## 解決策：NULLを一意な値として使う

NULLの問題のほとんどはSQLの3値論理の振る舞いについてのよくある誤解に基づいています。

### スカラー式や論理式でのNULL

以下によくある間違いの予想した結果と予想に反する実際の結果のいくつかを示します。

| 式 | 予想した結果 | 実際の結果 | 理由 |
| --- | --- | --- | --- |
| NULL = 0 | TRUE | NULL | NULLはゼロではない |
| NULL = 12345 | FALSE | NULL | 不明な値が、ある値と等しいかどうかはわからない |
| NULL <> 12345 | TRUE | NULL | 不明な値が、ある値と等しくないかどうかはわからない |
| NULL + 12345 | 12345 | NULL | NULLはゼロではない |
| NULL |  | ‘string’ | ‘string’ |
| NULL = NULL | TRUE | NULL | 不明な値と不明な値が等しいかどうかはわからない |
| NULL <> NULL | FALSE | NULL | 不明な値と不明な値が等しくないかどうかはわからない |
| NULL AND TRUE | FALSE | NULL | NULLはFALSEではない |
| NULL AND FALSE | FALSE | FALSE | AND FALSEの真理値は全てFALSEになる |
| NULL OR FALSE | FALSE | NULL | NULLはFALSEではない |
| NULL OR TRUE | TRUE | TRUE | OR TRUEの真理値は全てTRUEになる |
| NOT (NULL) | TRUE | NULL | NULLはFALSEではない |

これらの例は予約後 `NULL` を使用するときのみではなく、値が `NULL` であるどのような列や式にも当てはまります。

またNULLはTRUEでもFALSEでもないという概念がカギです。

### NULLの検索

SQL標準で定義されている `IS NULL` 述語は、対象のデータがNULLの場合TRUEを返し、その反対に、 `IS NOT NULL` は、対象のデータがNULLでない場合にTRUEを返します。

```sql
SELECT * FROM Bugs WHERE assigned_to IS NULL;
SELECT * FROM Bugs WHERE assigned_to IS NOT NULL;
```

またSQL-99標準では、比較述語 `IS DISTINCT FROM` が定義されています。以下の2つのクエリは同様です。

```sql
SELECT * FROM Bugs WHERE assigned_to IS NULL OR assigned_to <> 1;
SELECT * FROM Bugs WHERE assigned_to IS DISTINCT FROM 1;
```

`IS DISTINCT FROM` は `リテラル値またはNULLを渡したいプリペアドステートメントでも使用することができます。`

```sql
SELECT * FROM Bugs WHERE assigned_to IS DISTINCT FROM ?;
```

`IS DISTINCT FROM` のサポートはデータベース製品によって異なります。PostgreSQL、IBM DB2、Firebirdはサポートしていますが、OracleとMicrosoft SQL Serverはまだサポートしています。

MySQLは `IS NOT DISTINCT FROM` と同様な、独自hの演算子 `<==>` を提供しています。

### 列にNOT NULL制約を宣言する

NULLがアプリケーションのポリシーに反する場合や、その列においてNULLが意味をなさない場合には、列にNOT NULL制約を宣言するのが良いでしょう。アプリケーションコードに頼るのではなく、データベースで一貫した制約を強制する方がより良い方法だと言えます。

### 動的なデフォルト

クエリによっては、ロジックを単純化するために列や式の値がNULLであることを強制したくなるかもしれません。とはいは、あらかじめ値を格納したいわけでもありません。この場合に便利なのが `COALESCE` 関数です。

例えば以下のクエリでユーザーのフルネームの連結の例では、ミドルネームのイニシャルがNULLの場合でも式全体の結果はNULLにはなりません。

```sql
SELECT first_name || COALESCE('' || middle_initial || '', '') || last_name AS full_name
FROM Accounts;
```

`COALESCE` はSQL標準で定義されている関数です。データベースの製品によっては、同様の関数を、 `NVL` や `ISNULL` のような名前で提供しています。

## まとめ

データ型を問わず、欠けている値にはNULLを用いるようにしましょう。

## アンビギュアスグループ（曖昧なグループ）

## 目的

クエリで `GROUP BY` を用いて最大値や平均値だけではなく、その最大値が見つかった行の他の属性も取得するクエリを実行することです。

## アンチパターン：非グループ化列を参照する

### 単一値の原則（Single-Value Rule）

以下のクエリでは、 `product_id` の個別の値それぞれに1つの行グループが存在します。

```sql
SELECT product_id, MAX(date_reported) AS latest
FROM Bugs INNER JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

このクエリのSELECT句のリストに列挙される全ての列は、行グループごとに単一の値の行でなければなりません。これは「 **単一値の原則（Single-Value Rule）** 」と呼ばれます。

しかし、 `GROUP BY` 句には列挙されていない列では事情が異なります。データベースは、これらの列でグループ内の全ての行に同じ値があることを、常には保証できないのです。

```sql
SELECT product_id, MAX(date_reported) AS latest, bug_id
FROM Bugs INNER JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

この例では、 `product_id` に対応した `bug_id` は複数存在します。 `GROUP BY` 句には列挙されていない列では、グループ毎の単一値の保証がないため、データベースはそれらの列を単一値の原則に反するものと見なします。

## 解決策：曖昧でない列を使用する

### 関係従属性のある列のみにクエリを絞る

最もシンプルな方法は、クエリから曖昧な列を排除することです。

```sql
SELECT product_id, MAX(date_reported) AS latest
FROM Bugs INNER JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

### 相関サブクエリを使用する

相関サブクエリ（correlated subquery）には外部クエリへの参照が含まれるため、外部クエリの各行に対応する結果がことります。

```sql
SELECT bp1.product_id, b1.date.reported AS latest, b1.bug_id
FROM Bugs AS b1 INNER JOIN BugsProducts AS bp1 USING (bug_id)
WHERE NOT EXISTS
(
    SELECT * FROM Bugs AS b2 INNER JOIN BugsProducts AS bp2 USING (bug_id)
    WHERE bp1.product_id = bp2.product_id 
    --date_reportedが最新でないデータを抽出し、NOT EXISTで除外
    AND b1.date_reported < b2.date_reported
);
```

相関サブクエリを利用することで、最新の日付を持つバグの取得を行うことができます。ただし、この方法は、相関サブクエリが外部クエリの各行に対してそれぞれ実行されるため、最善のパフォーマンスは得られないことにも注意が必要です。

### 導入テーブルを使用する

サブクエリを導出テーブル（derived table）として使用すると `product_id` と各製品に対応する最新の日付のみを含む中間結果を取得できます。

```sql
SELECT m.product_id, m.latest, b1.bug_id
FROM Bugs AS b1
INNER JOIN BugsProducts AS bp1 USING (bug_id)
INNER JOIN (
    SELECT bp2.product_id, MAX(b2.date_reported) AS latest
    FROM Bugs AS b2
    INNER JOIN BugsProducts bp2 USING (bug_id)
    GROUP BY bp2.product_id
) AS m
ON bp1.product_id = m.product_id AND b1.date_reported = m.latest;
```

| product\_id | latest | bug\_id |
| --- | --- | --- |
| 1 | 2010-06-01 | 2248 |
| 2 | 2010-02-01 | 3456 |
| 2 | 2010-02-01 | 5150 |
| 3 | 2010-01-01 | 5678 |

サブクエリで返された `latest` の日付が複数行と一致する場合、1つの製品に複数行が取得される可能性があることに注意しましょう。

### OUTER JOINを使用する

外部結合では、一致する行が存在しない場合、存在しない行の列にはNULLが入ります。クエリでNULLを検出した場合、対象の行が存在しなかったものだとわかります。

```sql
SELECT bp1.product_id, b1.date_reported AS latest, b1.bug_id
FROM Bugs AS b1
INNER JOIN BugsProducts AS bp1
ON b1.bug_id = bp1.bug_id
LEFT OUTER JOIN (
    Bug AS b2 INNER JOIN BugsProducts AS bp2
    ON b2.bug_id = bp2.bug_id
)
ON (
    bp1.product_id = bp2.product_id
    AND (
        b1.date_reported < b2.date_reported
        OR b1.date_reported = b2.date_reported
        AND b1.bug_id < b2.bug_id
    )
)
WHERE b2.bug_id IS NULL;
```

| product\_id | latest | bug\_id |
| --- | --- | --- |
| 1 | 2010-06-01 | 2248 |
| 2 | 2010-02-01 | 5150 |
| 3 | 2010-01-01 | 5678 |

このクエリを理解するには、しばらく頭を捻ったり、する必要があるかもしれません。しかし、理解さえすれば、この技法は重要なツールになります。

結合を使用した解決策は、大量のデータに対するパフォーマンスが重要な場合に使用しましょう。この解決策はサブクエリベースの解決策よりもパフォーマンスが優れています。とはいえ、ある方法が別のものよりも優れていると仮定するのではなく、複数種類のクエリのパフォーマンスを実際に測定することが大切です。

### 他の列に対しても集約関数を使用する

他の列にも集約関数を適用することによって、単一の原則に従わせることができます。

```sql
SELECT product_id, MAX(date_reported) AS latest, MAX(bug_id) AS latest_bug_id
FROM Bugs INNER JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

この解決策は、最大の `bug_id` が最新日付を持つことを当てにできる場合のみに使用します。

### グループごとに全ての値を連結する

単一値の原則に準拠するために、 `bug_id` に特殊な集約関数を使うこともできます。MySQLとSQLiteは各グループ内のすべての値を1つに連結する関数 `GROUP_CONCAT` をサポートしています。デフォルトでは、カンマ区切りの文字列を返します。

```sql
SELECT product_id, MAX(date_reported) AS latest, GROUP_CONCAT(bug_id) AS bug_id_list
FROM Bugs INNER JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

| product\_id | latest | bug\_id\_list |
| --- | --- | --- |
| 1 | 2010-06-01 | 1234,2248 |
| 2 | 2010-02-01 | 3456,4077,5150 |
| 3 | 2010-01-01 | 5678,5678 |

但し、このクエリでは、最新の日付に対応する `bug_id` は特定できません。bug\_id\_listには、各グループの `bug_id` がすべて含まれます。

また、SQL標準に準拠していないというデメリットもあります。

## まとめ

曖昧なクエリ結果を避けるために、単一値の原則に従いましょう。

## ランダムセレクション

## 目的

目的は、データのランダムサンプルのみを返す、効率の良いSQLクエリを書くことです。

## アンチパターン：データをランダムにソートする

SQLでランダムな行を取得するための最も一般的な方法、ランダムにソートを行い、最初の行を取得することです。

```sql
SELECT * FROM Bugs ORDER BY RAND() LIMIT 1;
```

このクエリには弱点があります。弱点を考える前に、まずは従来型のソートと比較しましょう。

従来型のソートは、列の値を比較し、値の大きさによって、降順または昇順に行を並べ替えます。2回以上行っても同じ結果になるので、このソートには再現性があり、インデックスのメリットも得られます。

```sql
SELECT * FROM Bugs ORDER BY date_reported;
```

ソートの基準を行ごとにランダムな値を返す関数にすると、ある行の値が他の行の値より大きいか小さいかに関わらず、行はランダムにソートされます。このため、ソート結果の順番は各行の値と無関係になり、順番はソートするたびに変わります。また、RAND関数によってソートを行うということは、インデックスからメリットを得られないことを意味します。そのためパフォーマンスの低下にも繋がるのです。

## 解決策：特定の順番に依存しない

### 1と最大値の間のランダムなキー値を選択する

テーブル全体のソートを回避する1つの方法は1から主キーの最大値までの間の値をランダムに選択することです。

```sql
SELECT b1.*
FROM Bugs AS b1
INNER JOIN (
    SELECT CEIL(RAND() * (SELECT MAC(bug_id) FROM Bugs)) AS rand_id
) AS b2 ON b1.bug_id b2.rand_id;
```

この解決策は主キー値が1から開始され、かつ連続していることを前提としています。すなわち、1から最大値までに欠けている数値があってはいけません。

1から最大値までの間のすべての値が使用されていることが確実な場合にのみ、この解決策を使用しましよう。

### 全てのキー値のリストを受け取り、ランダムに1つを選択する

アプリケーションコードを用いて、結果セットの主キーから値を1つ選択する技法もあります。

```sql
SELECT bug_id FROM Bugs;
-- アプリケーションコードで取得した\`bug_id\`からランダムな\`bug_id\`を1つ取得する

SELECT * FROM Bugs WHERE bug_id = 乱数;
```

この技法はテーブル全体のソートも回避できますし、各キーもほぼ平等に選択されてます。但しこの方法には他の弱点があります。

- データベースから全ての `bug_id` 値を取得するとリストのサイズが非常に大きくなってしまう可能性があります。
- クエリを2回実行する必要があります。

### オフセットを用いてランダムに行を取得する

データの行数をカウントし、0と行数までの間の乱数を返す技法もあります。

```sql
SELECT FLOOR(RAND() * (SELECT COUNT(*) FROM Bugs)) AS id_ofset;

SELECT * FROM Bugs LIMIT 1 OFFSET id_ofset;
```

この解決策は、SQL標準にない、LIMIT句を使用しています。LIMIT句は、MySQL、PostgreSQL、SQLiteでサポートされています。

この解決策はキー値が連続していることを前提にできず、かつ各行が平等に選択される必要がある場合に使用しましょう。

## まとめ

クエリには、最適化できないものもあります。最適化できない場合は、別のアプローチを採用しましょう。

## プラマンズ・サーチエンジン（貧者のサーチエンジン）

## 目的

この目的は全文検索を行うことです。SQLの還俗の1つは、列の値がアトミックであること、つまり値と比較可能であるとことです。だたし、比較にの際には常に値全体が比較されます。部分文字列の比較は、SQLにおいては、非効率性や不正確さにつながるのです。

しかし部分文字列の比較は、多くのケースで必要になります。SQLを用いてその問題を切り抜ける方法を検討しましょう。

## アンチパターン：パターンマッチ述語を使用する

SQLには、文字列比較のための **パターンマッチ述語** があります。最も一般的なものは、LIKE述語です。

```sql
SELECT * FROM Bugs WHERE description LIKE '%crash%';
```

多くのデータベース製品は、正規表現も独自の方法でサポートしています。正規表現を使用すると、どのような部分文字列に対してもパターンマッチを行えるため、ワイルドカードは不要になります。MySQLの正規表現述語を使用した例です。

```sql
SELECT * FROM Bugs WHERE descritpion REGEXP 'crash';
```

パターンマッチ述語の最も大きなデメリットは、パフォーマンスの低下です。パターンマッチ述語は従来型のインデックスのメリットが得られないため、テーブルのすべての行をスキャンしなければなりません。

2番目の問題点は、LIKEや正規表現を用いた単純なパターンマッチでは、意図しないマッチが生じてしまうことです。

```sql
SELECT * FROM Bugs WHERE description LIKE '%one%';
```

このクエリは単語「one」を含むテキストとマッチしますが、「money」、「prone」、「lonely」などもマッチしてしまいます。この問題を解決するために、単語境界のための特別なパターンをサポートしているデータベースもあります。

```sql
SELECT * FROM Bugs WHERE description REGEXP '[[:<:]]one[[:>:]]';
```

このように、パフォーマンス、スケーラビリティ、処理の煩雑さなども問題を考えると、キーワード検索のために、単純なパターンマッチングを使用するのは、決して良い方法とは言えないのです。

## 解決策：適切なツールを使用する

最善の方法は、SQLの代わりに専用の全文検索エンジンを使用することです。

### ベンター拡張

主要なデータベース製品は、全文検索というありふれた要件に対する独自の解決策を用意しています。但しこれら独自機能は標準化されておらず、ベンダーかんの互換性もありません。使用するデータベース製品が1つである場合には、これらの機能は、SQLクエリと親和性が高く、最善策だと言えるでしょう。以下にMySQLとPostgresSQLのデータベース製品における全文検索機能の概要を示します。

### MySQLのフルテキストインデックス

MySQLでは、MyISAMストレージエンジンのみがサポートする、シンプルなふるテキストインデックが提供されています。フルテキストインデックスの定義できるのは、 `CHAR` 、 `VARCHAR` 、 `TEXT` 型の列です。

Bugsテーブルの `suumary` 列と `description` 列の内容を含むフルテキストインデックスの定義です。

```sql
ALTER TABLE Bugs ADD FULLTEXT INDEX bugfts (summary, description);
```

インデックスに格納されたテキストからキーワードを検索するには、 `MATCH` 関数を用います。 `MATCH` 関数はフルテキストインデックスに関連付けられた列名を指定する必要があります。列名は複数指定することができるので、同じテーブルの他の列に対して定義されたインデックも同時に使えます。

```sql
SELECT * FROM Bugs WHERE MATCH(summary, description) AGAINST ('crash');
```

### PostgreSQLでのテキスト検索

パフォーマンスを最適化するには、オリジナルのテキスト形式に加えて、テキスト検索可能な特別なデータ型 `TSVECTOR` を用いてコンテンツを格納する必要があります。

```sql
CREATE TABLE Bugs (
    bug_id SERIAL PRIMAY KEY,
    summary VARCHAR(80),
    desription TEXT,
    ts_bugtext TSVECTOR
    -- 他の列...
);
```

`TSVECTOR` 列は検索対象のテキスト列の内容と同期する必要があります。PostgreSQLでは、同期作業をより容易にするために、組み込みのトリガーが提供されています。

```sql
CREATE TRIGGER ts_bugtext BEFORE INSERT OR UPDATE ON Bugs
FOR EACH ROW EXECUTE PROCEDURE
tsvector_update_trigger(ts_bugtext, 'pg_catelog.english', summary, descritpion);
```

さらに、 `TSVECTOR` 列に対して `**GIN` （汎用転置インデックス）\*\*を作成しなければなりません。

```sql
CREATE INDEX bugs_ts ON Bugs USING GIN(ts_bugtext);
```

これで、PostgreSQLのテキスト検索演算子「@@」を用いて、フルテキストインデックスを活用した効果的な全文検索が行えるよういなります。

```sql
SELECT * FROM Bugs WHERE ts_bugtext @@ to_tsquery('crash');
```

他にも、検索可能なコンテンツ、検索クエリ、検索結果をカスタマイズする多くのオプションがあります。

## まとめ

問題を解決するために、必ずしもSQLを使う必要はありません。

## スパゲッティクエリ

## 目的

多く直面する問題の1つが、「どのようにして目の前の仕事を1つのクエリで実現するか」です。

タスクの複雑性を減らすことはできませんが、解決策はシンプルにしたいと考えます。そこでクエリを優雅に効率的に書くことを目標にします。そしてタスクを1つのクエリで解決することでこの目標を達成したとみなします。

## アンチパターン：複雑な問題をワンステップで解決しようとする

SQLは1つのクエリやステートメントで多くのことを実現できますが、1つのクエリで全てのタスクを処理することを強制するものではありませんし、時にそれが良くないアイデアである場合もあります。

### 意図に反した結果

1つのクエリで処理しようとすると、しばしば **デカルト積** （Cartesian product）が生じてしまします。クエリで指定する2つのテーブルが関連を制限する条件を持たないときに生まれます。この制限がないと、2つのテーブルを結合することによって、1つのテーブルの各行が、もう1つのテーブルの **すべて** の行とペアになってしまいます。各行のすべての組み合わせによって、結果は予期しているよりもはるかに多くの行数になってしますのです。

以下は例です。バグデータベースに、製品別の修正済みバグ数と未修正バグ数を数えるとします。

```sql
SELECT p.product_id, COUNT(f.bug_id) AS count_fixed, COUNT(o.bug_id) AS count_open
FROM BugsProdcuts AS p
INNER JOIN Bugs AS f ON p.bug_id = f.bug_id AND f.status = 'FIXED'
INNER JOIN BugsProducts AS p2 USING (product_id)
INNER JOIN Bugs AS o ON p2.bug_id = o.bug_id AND o.status = 'OPEN'
WHERE p.product_id = 1
GROUP BY p.product_id;
```

その製品の修正済みバグ数が11件、未修正のバグが7件であると知っていました。しかし、このクエリの結果は、頭を悩ませるものになります。

| product\_id | count\_fixed | count\_open |
| --- | --- | --- |
| 1 | 77 | 77 |

結果がこれほどまでに異なる原因はなんでしょうか。 `BugProducts` テーブルが `Bugs` テーブルの2つの部分集合と結合され、結果としてこれらの2つの部分集合のデカルト積が生じました。11行の修正済みバグが7行の未修正バグと全てペアになっていたのです。

このように1つのクエリで複数のタスクを実現しようとした結果、意図しないデカルト積が頻繁に生まれます。

### さらなる弊害

1つのクエリで複数のタスクを行おうとすると、意図しない結果が導かれるだけでなく、クエリの記述や修正、デバッグが難しくなるとうい点も肝に銘じておくできます。さらに実行時のコストもあります。結合や相関サブクエリなどの多い手の込んだクエリは、シンプルなクエリに比べて最適化処理や、高速な実行が難しくなります。SQLクエリの数を減らせば、パフォーマンスがあがるという考えが間違っていませんが、お化けのような複雑な1つのクエリは、パフォーマンスを低下されることがあります。それよりもシンプルなクエリを複数実行する方が良い場合があるのです。

## 解決策：分割統治を行う

全く同じ結果セットを生む2つのクエリをん選択できる場合は、単純なクエリを選ぶべきであると言えます。スパゲッティクエリの修正においても、この原則を念頭におくべきです。

### ワンステップずつ

デカルト積を避けるには、スパゲッティクエリを幾つかのクエリにに分割する必要があります。

```sql
SELECT p.product_id, COUNT(f.bug_id) AS count_fixed
FROM BugsProdcuts AS p
LEFT OUTER JOIN Bugs AS f ON p.bug_id = f.bug_id AND f.status = 'FIXED'
WHERE p.product_id = 1;
GROUP BY p.product_id;

SELECT p.product_id, COUNT(o.bug_id) AS count_open
FROM BugsProdcuts AS p
LEFT OUTER JOIN Bugs AS o ON p2.bug_id = o.bug_id AND o.status = 'OPEN'
WHERE p.product_id = 1;
GROUP BY p.product_id;
```

これらの2つのクエリの結果は、期待通り11と7です。

クエリを複数に分割するのは「野暮な」解決策と思えるかもしれません。しかしクエリ分割は、開発、メンテナンス、パフォーマンスなどの面でも様々なメリットをもたらします。

- 分割したクエリからはデカルト積は生じません。このため、クエリが正確な結果をもたらしていることを簡単に確認できます。
- 新たな要件が追加されたとき、複雑なクエリをさらに複雑にするよりも、単純なクエリを新たに書く方がはるかに簡単です。
- SQLエンジンは複雑なクエリよりも単純なクエリの方がよりスムーズかつ確実に実行できます。クエリ分割によって処理が重複していると思えたとしても、それを補って余りあるメリットがあります。
- コードレビューやチームメンバーのトレーニングセッションでは、シンプルな複数のクエリを説明する方が、1つの複雑なクエリを説明するよも簡単です。

### UNIONを用いる

複数のクエリの結果は、UNIONによって1つの結果セットにまとめることができます。これは、どうしても1つのクエリを実行して1つの結果を得たい場合（結果をソートしなければならない場合）に効果的です。

```sql
(
    SELECT p.product_id, COUNT(f.bug_id) AS bug_count
    FROM BugsProdcuts AS p
    LEFT OUTER JOIN Bugs AS f ON p.bug_id = f.bug_id AND f.status = 'FIXED'
    WHERE p.product_id = 1;
    GROUP BY p.product_id;
)
UNION ALL
(
    SELECT p.product_id, COUNT(o.bug_id) AS bug_count
    FROM BugsProdcuts AS p
    LEFT OUTER JOIN Bugs AS o ON p2.bug_id = o.bug_id AND o.status = 'OPEN'
    WHERE p.product_id = 1;
    GROUP BY p.product_id;
)
ORDER BY bug_count DESC;
```

### CASE式とSUM関数を組み合わせる

やや本文の趣旨と異なりますが、条件ごとの集約を1つのクエリでシンプルに行うために、CASE式とSUM関数を組み合わせる方法がよく使われます。

```sql
SELECT p.product_id,
       SUM(CASE b.status WHEN 'FIXED' THE 1 ELSE 0 END) AS count_fixed,
       SUM(CASE b.status WHEN 'OPEN' THE 1 ELSE 0 END) AS count_open
FROM BugsProducts AS P
INNER JOIN Bugs AS b USING (bug_id)
WHERE p.product_id = 1
GROUP BY p.product_id;
```

## まとめ

SQLでは、1行のコードで複雑な問題を解決できると思える場合があります。しかし、状況に応じてクエリを分割することも検討するようにしましょう。

## インプリシットカラム（暗黙の列）

## 目的

ソフトウェア開発者は概して、キーをたくさん打つことを好みません。SQLで入力が手間だと感じる例はの1つが。全ての列名を指定することです。

```sql
SELECT bug_id, date_reported, summary, description, resolution
       reported_by, assigned_to, verified_by, status, priority, hours
FROM Bugs;
```

このサンプルを見れば、SQLのワイルドカード機能を頻繁に使うのも驚くに値しません。ワイルドカードを使用するとクエリを簡潔に書くことができます。

```sql
SELECT * FROM Bugs;
```

## アンチパターン：ショートカットの罠に陥る

ワイルドカードや暗黙的な列の指定によって、タイプ数は減らせます。しかし、この方法には様々な弊害があります。

### リファクタリングにおける問題

例えば、Bugsテーブルに `date_due` 列を加える必要があるとします。

```sql
ALTER TABLE Bugs ADD COLUMN date_due DATE;
```

すると、暗黙的な列の指定でINSERTステートメントがエラーを返すようになります。

```sql
INSERT INTO Bugs VALUES (DEFAULT, CURRENT_DATE, '新規バグ', 'テストが失敗します', NULL, 123, NUUl, NULL, DEFAULT, 'MEDIUM', NULL);

-- SQLSTEATE 21S01; Column count doesn't match value count at row 1
```

また、列名を知らずに `SELECT *` クエリを実行する場合、列は定義順に基づいて参照されるため、想定と違う結果が返ってきます。

列の追加、削除、名前変更などを行うと、クエリ結果に生じた変化をコードがうまく扱えなくなる場合があります。

### 隠れた代償

クエリでワイルドカードを使うのは便利ですが、パフォーマンスとスケーラビリティに悪い影響を及ばす場合があります。クエリが多くの列をフェッチするようになるため、多くのデータがアプリケーションとデータベースサーバーの間を行き来しなくてはなりません。

## アンチパターンを用いても良い場合

ワイルドカードの使用は、SQLを素早く書きたい場合には妥当であると言えます。例えば、ある解決法を試してみたいときや、現行システムのデータを診断したい時です。1回しか使用しないクエリでは、保守性の低さはそれほど問題になりません。

## 解決策：列名を明示的に指定する

ワイルドカードや暗黙的な列指定を使用せず、 **必要な列名は明示的に指定する** ようにしましょう。

```sql
SELECT bug_id, date_reported, summary, description, resolution
       reported_by, assigned_to, verified_by, status, priority, hours
FROM Bugs;
```

```sql
INSERT INTO Accounts (account_name, first_name, last_name, email, passowrd_hash, portrait_image, hourly_rate) VALUES ('bkarwin', 'Bill', 'Karwin', 'bill@example.com', SHA2('xyzzy', 256), NULL, 49.95);
```

列名を全て入力するのは手間がかかると思うからもしれませんが、それだけの価値があるのです。

### 誤りの防止

クエリの選択リストで列を明示的に指定することで、本章でみてきたエラーや混乱が発生しにくくなります。

- テーブル定義の列の順番が変更された場合でも、クエリ結果の列の位置は変わりません。
- テーブルに列が加えられた場合でも、クエリ結果に影響はありません。
- テーブルから列が削除された場合、クエリはエラーを返します。ただし、これは良いエラーです。修正すべきコードをすぐに特定できるからです。

INSERTステートメントでも列を明示的に指定すると、同じようなメリットを得られます。

### それは多分、必要ない（YAGNI: You Ain’t Gonna Need It）

スケーラビリティとスループットを考慮すると、ネットワーク帯域幅はなるべく減らしたいと考えるのも当然です。SQLでワイルドカードを使用しないようにすると、使うかどうか分からない列ををなるべく作らないよいうにしようという意識が高まります。列が必要最小限であることは、対包量が減ることを意味するからです。結果として、帯域幅も効率的に使用できるというわけです。

## まとめ

必要な列だけ指定するようにしましょう。

## リーダブルパスワード（読み取り可能パスワード）

## 目的

パスワードを使用するアプリケーションには、ユーザーがパスワードを忘れることがつきものです。こうしたユーザーのために、最近のアプリケーションの多くは、電子メールを通じてパスワードのさいつうやリセットを行うことができます。この章の目的はパスワードのリカバリーとリセットを行うことが目的です。

## アンチパターン：パスワードを平文で格納する

平文のパスワードが含まれた電子めーるをユーザーがリスエストできるようにしてしまうことです。これはデータベース設計における重大なセキュリティ欠陥です。

### パスワードの認証

ユーザーがログインするとき、アプリケーションはユーザーの入力内容を、データベースに格納したパスワード文字列と比較します。パスワードが平文として格納されている設計では、この比較も平文として行われます。

```sql
SELECT CASE WHEN passowrd = 'opensesame' THE 1 ELSE 0 END AS passowrd_matches
FROM Accounts
WHERE account_id = 123;
```

ユーザーの入力した文字列を平文としてSQLクエリに挿入するのは、攻撃者にパスワードを晒してしまうことを意味します。

## 解決策：ソルトを付けてパスワードハッシュを格納する

このアンチパターンの大きな問題点は、パスワードのオリジナル形式が解読可能な形で格納されていることです。しかし、実は解読可能でなくとも、ユーザーの入力内容を使用したパスワード認証は行なえるのです。

### ハッシュ関数を理解する

パスワードを、一方向性の暗号学的ハッシュ関数（cryptographic hash function）を用いて暗号化します。ハッシュ関数で返されるハッシュ値は固定長の文字列であるため、ハッシュからオリジナルの文字列の長さを特定することもできません。

ハッシュには **不可逆である** という特性もあります。ハッシュアルゴリズムは、入力情報を「失う」ように設計されているため、ハッシュ値から入力文字列を復元することはできません。

### SQLでもハッシュの使用

イアkに示す例はAccountsテーブルを再定義したものです。SHA-256を用いてパスワードをハッシュ化すると常に64文字になります。このため、列を固定長の `CHAR` 列として定義します。

```sql
CREATE TABLE Accounts (
    account_id SERIAL PRIMARY KEY,
    account_name VARCHAR(20),
    email VARCHAR(100) NOT NULL,
    password_hash CHAR(64) NOT NULL
);
```

ハッシュ関数はSQL標準では定義されていないので、データベース製品ごとに異なるハッシュ拡張を使わなければなりません。例えば、SSLサポートを有効にしたMySQLでは、SHA2関数が使えます。SHA2関数の第2引数にはビット長を渡します。

```sql
INSERT INTO Accounts (accoiunt_id, account_name, email, passoword_hash)
VALUES (123, 'billkarwin', 'bill@example.com' SHA2('xyzzy', 256));
```

ユーザー入力同じハッシュ関数を適用し、その結果をデータベースに格納されたハッシュ値と比較することで、パスワードの妥当性を確認できます。

```sql
SELECT CASE WHEN passoword_hash = SHA2('xyzzy', 256) THEN 1 ELSE 0 END AS password_matches
FROM Accounts
WHERE account_id = 123;
```

また、パスワードハッシュの値をハッシュ関数が返せない文字列に変更することで、アカウントを簡単にロックできます。

### ハッシュにソルトを加える

ハッシュを格納していても、攻撃者にデータベースへのアクセスを許してしまえば、攻撃者はとうい&エラーでパスワードを解読しようとするでしょう。この種の辞書攻撃を防ぐ方法の1つは、暗号化前のパスワードへの「 **ソルト** 」の付加です。ソルトとは、ハッシュ関数に渡す前にパスワードに連結する無意味な文字列のことです。例えば、パスワードに使う文字列が「password」である場合、そのままハッシュ化した場合とソルトを付けてハッシュ化した場合とでは、ハッシュ値が異なります。

```sql
SHA2('password', 256)
= '5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8'

SHA2('password' || 'G0y6cf3$.ydLVkx4I/50', 256)
= '9cb669bbba0bfd55189f7b58c1d85014ec4438e815e2993847a289bb41c46de8'
```

それぞれのパスワードに異なるソルト値を付加することで攻撃者はパスワードの解読にはトラアンドエラーと同じだけの時間がかかるため、攻撃者をふりだしに戻することができます。

```sql
CREATE TABLE Accounts (
    account_id SERIAL PRIMARY KEY,
    account_name VARCHAR(20),
    email VARCHAR(100) NOT NULL,
    password_hash CHAR(64) NOT NULL
    salt BINARY(20) NOT NULL
);

INSERT INTO Accounts (account_id, account_name, email, password_hash, salt)
VALUES (123, 'billkarwin', 'bill@example.com',
SHA2('xyzzy' || 'G0y6cf3$.ydLVkx4I/50', 256), 'G0y6cf3$.ydLVkx4I/50');

SELECT (password_hash = SHA2('xyzzy' || salt, 256)) AS password_matches FROM Accounts
WHERE account_id = 123;
```

レインボーテーブル対策を行うには、ソルトとパスワードを合わせた長さが最低でも20文字は必要です。また各パスワードごとに異なるソルトを付加するべきです。また、前述のサンプルではソルトの文字列に 印刷可能な値が含まれていますが、ソルトにはランダムな、印刷不可能な文字を用いることもできます。

### SQLからパスワードを隠す

ソルトをパスワードに孵化することで、セキュリティを十分に確保できるようになったと思うかもしれません。しかしパスワードはSQL文の中ではまだ平文として使用されています。つまり、攻撃者によってネットワークバケットが傍受された場合や、SQLクエリが記録されたログファイルが攻撃者の手に渡ってしまった場合には、パスワードを読み取られてしますのです。

これを防ぐ方法は、SQLクエリではパスワードを平文として使わないことです。アプリケーションコードでハッシュを計算し、SQLクエリでは、ハッシュのみを用いるようにします。

```sql
-- ソルトを取得する
SELECT salt FROM Accounts WHERE account_name = 'bill';

-- アプリケーションコードでパスワードをハッシュ化する

-- アプリケーションコードでハッシュ化した値で比較する
SELECT (password_hash = '@hash') AS passowrd_matches
FROM Accounts
WHERE account_name = 'bill'
```

ウェブアプリケーションには、まだ攻撃者によってネットワーク上のデータが傍受される可能性がある場所があります。それは、ユーザーのブラウザとウェブアプリケーションサーバーの間のネットワークです。このため、ブラウザからアプリケーションにパスワードを送るときには、セキュアHHTP（HHTPS）を使用する方法がよく用いられています。

### パスワードをリカバリーするのではなく、リセットする

データベースには、パスワードではなく、パスワードのハッシュを格納しているので、リカバリーをすることはできません。しかし、パスワードを忘れたユーザーにアクセスを許可する方法はあります。以下に実装例を2つ紹介します。

1番目の方法は、パスワードを電子メールで送るのではなく、アプリケーションが生成した一時パスワードを電子メールで送る方法です。セキュリティを強化するために、一時パスワードを短時間で無効にすることもできます。もちろん、ユーザーの初回ログイン時にパスワードの変更を強制するようにアプリケーションを設計すべきです。

2番目の方法は、電子メールに新しいパスワードを記載する代わりに、リクエストをデータベーステーブルに記録し、一意トークンを識別子として割り当てるというものです。

```sql
CREATE TABLE PasswordResetRequest (
  token CHAR(32) PRIMARY KEY,
  account_id BIGINT UNSIGNED NOT NULL,
  expiration TIMESTAMP NOT NULL,
  FOREIGN KEY (account_id) REFERENCES Accounts(account_id)
);

SET @token = MD5('billkarwin' || CURRENT_TIMESTAMP || RAND());

INSERT INTO PasswordResetRequest (token, account_id, expiration) VALUES (@token, 123, CURRENT_TIMESTAMP + INTERVAL 1 HOUR);
```

次に、電子メールにトークンを記載します。この方法であれば、第三者が違法にパスワードリセットをリクエストしてきた場合でも、アカウントの実際の所有者のみにトークンを記載した電子メールが送信されます。

```
From: daemon
To: bill@example.com
件名 : パスワードのリセット

アカウントのパスワードリセット依頼に回答します。
1 時間以内に以下のリンクをクリックしてパスワードを変更してください。
1 時間が経過するとリンク先のページにはアクセスできなくなり、パスワードも変更できません。
http://www.example.com/reset_password?token=f5cabff22532bd0025118905bdea50da
```

もちろん、第三者にこのページへアクセスされば場合には危険が生じます。ただし、危険な制限でこのリスクは低減できます。例えば、特殊画面の表示時間を短くしたり、パスワードの再設定対象のアカウントを画面に表示しないという方法があります。

暗号化技術は、日々進化を続けています。極めてセキュアなシステムを開発する必要がある場合は、以下のようなさらに高度な技法の採用を検討しましょう。

- PBKDF2 - 広く普及している暗号化標準で、鍵強化(keystrengthening)に使用されています
	- [http://tools.ietf.org/html/rfc2898](http://tools.ietf.org/html/rfc2898)
- Bcrypt - アダプティブハッシュ(adaptivehashing)関数の実装です
	- [http://bcrypt.sourceforge.net/](http://bcrypt.sourceforge.net/)

## まとめ

あなたが読み取れるものは、攻撃者にも読み取れます。

## SQLインジェクション

## 目的

多くのSQLはアプリケーションコードとして連携して使われます。つまりクエリの文字列内にアプリケーションの変数を挿入うする必要があります。このような動的SQLを安全に記述することが目的です。

## アンチパターン：未検証の入力をコードとして実行する

SQLインジェクションは、SQLクエリ文字列に動的に挿入された文字列が、開発者の意図していない方法でクエリの構文を改変することによって生じます。例えば、以下のようなクエリがあります。 `$bug_id` は変数を表します。

```sql
SELECT * FROM Bugs WHERE bug_id = "$bug_id"
```

例えば、 `$bug_id` 変数の値の文字列が `1234; DELETE FROM Bugs` である場合、先ほどのクエリは次のようになります。

```sql
SELECT * FROM Bugs WHERE bug_id = 1234; DELETE FROM Bugs
```

このタイプのSQLインジェクションは、甚大な被害をもたらす可能性があります。

## 解決策：誰も信用してはならない

SQLコードの安全性を保証するための唯一絶対の方法などというものは存在しません。次に紹介する技法のすべての習得し、状況に応じて適切に使い分けるようにしましょう。

### 入力のフィルタリング

ユーザーからの入力に危険な文字列が含まれていないかどうかを探るよりも、その入力にとって無効な文字を取り除くようにするべきです。

### 動的値のパラメータ化

動的な部分がシンプルな値から構成されているときには、SQLから分離させるために **プリペアドステートメント** を用いることができます。攻撃者が悪意ある値をパラメータに渡しても、RDBMSはそのパラメータを値であると解釈します。

つまり、アプリケーション変数をリテラル地としてSQL文字列と連結する必要がある場合は、プリペアドステートメントを使うべきです。

ただし、パラメータに複数の値を渡したい場合（IN句など）は処理が少し複雑になります。以下はGo言語を用いて実装例です。

```
query := "SELECT \`account_id\`, \`account_name\`, \`email\` FROM \`Accounts\` "
placeholders := strings.Repeat("?, ", len(accountIDs))
query += fmt.Sprintf("WHERE \`account_id\` IN(%s) ", placeholders[:len(placeholders)-2])
query += "ORDER BY \`account_id\`"

args := make([]any, 0, len(accountIDs))
for _, id := range accountIDs {
    args = append(args, id)
}
rows, _ := tx.QueryContext(ctx, query, args...)
for rows.Next() {}
```

### 動的値を引用符で囲む

SQLインジェクション対策にはほとんどの場合、プリペアドステートメントが最善策となります。

しかし、パラメータプレースホルダーを持つクエリのインデックスに対して、クエリオプティマイザーがおかしな判断をする場合があります。

例えば、Accountsテーブルのaccount\_status列に「ACTIVE」か「BANNED」の2つの内どちらかの値が格納されているとします。そして、99%の行で「ACTIVE」が使用されているとします。

この場合、「account\_status = 'ACTIVE'」を求めるクエリの場合、インデックスを読みに行くと非効率となってしまいます。しかし、プリペアドステートメントで `account_status = ?`という式を使う場合オプティマイザーは作成済みのクエリを実行する場合にどちらの値が来るか判断できません。このため、不適切な最適化を行ってしまう場合があります。

このような稀なケースでは、一般的に推奨されているプリペアドステートメントではなく、SQLステートメントの中に直接値を挿入する方が良い場合もあります。その際には、 ****エスケープ処理**** を行う必要があります。MySQLであれば `\` がエスケープ文字になります。

### 他の開発者にコードをレビューしてもらう

欠陥を見つけるための最善策は、他人の目でチェックしてもらうことです。SQLインジェクション対策のコードレビューをする際には、以下のガイドラインに従いましょう。

1. アプリケーション変数や文字列連結、文字列置換によって構築されているSQLステートメントを特定
2. SQLステートメントで使われている、全ての動的コンテンツの起点を辿り、外部ソースから来る全てのデータを特定（ユーザー入力、外部ファイル、環境周り、外部のウェブサービス、サードパーティのコード、データベースから取得した文字列など）
3. これらの外部コンテンツには全て潜在的なリスクがあると想定し、フィルター、バリデーター、マッピング配列などを用いて、これら信用度の低いコンテンツを変換
4. プリペアドステートメントまたはエスケープ関数を用いて、SQLステートメントと外部データを組み合わせる
5. 他にもストアドプロシージャなどの動的SQLステートメントが隠れている可能性がある場所をチェック

## まとめ

ユーザーには、値の入力は許可しても、コードの入力を許可してはいけません。

## シュードキー・ニートフリーク（疑似キー潔癖症）

## 目的

次の表のように番号が連番になっていいないことを、気にする人もいます。

| bud\_id | status | product\_name |
| --- | --- | --- |
| 1 | OPEN | Open RoundFile |
| 2 | 4 | ReConsider |
| 4 | OPEN | ReConsider |

こうした欠番を気にしてアンチパターンに陥ってしまいます。

この章では、上記の問題を解決することが目的です。

## アンチパターン：隙間を埋める

欠けている行を見つけたら、多くの人は次の2つの方法のうちのどちらを用いるでしょう。

### 欠番を割り当てる

擬似キーの自動採番メカニズムを用いて新しい主キー値を割り当てる代わりに、新しい行に、検出した最小の番号を割り当てる方法です。

| bud\_id | status | product\_name |
| --- | --- | --- |
| 1 | OPEN | Open RoundFile |
| 2 | 4 | ReConsider |
| 4 | OPEN | ReConsider |
| 3 | NEW | Visual TurboBuilder |

しかし、最も小さな欠番の値を特定するには、不要な自己結合クエリを実行しなければなりません。

```sql
SELECT b1.bug_id = 1 AS max_bug_id
FROM Bugs AS b1
LEFT OUTER JOIN Bugs AS b2 ON b1.bug_id + 1 = b2.bug_id
WHERE b2.bug_id IS NULL
ORDER BY b1.bug_id LIMIT 1;
```

### 既存行に番号を振り直す

欠番を埋めてすべての値が連続するように、既存行のキーの値を更新するというものです。

| bud\_id | status | product\_name |
| --- | --- | --- |
| 1 | OPEN | Open RoundFile |
| 2 | 4 | ReConsider |
| 3 | OPEN | ReConsider |

既存行に番号を振り直すためには、先ほどの例で新しい行を挿入するために行なったものと同様のやり方で、欠番のキー値を特定する必要があります。また主キーを更新するのに、UPDATEステートメントを実行する必要があります。これらのステップはどちらも、 **競合状態** を引き起こす可能性があります。しかも、欠番の数が多い場合は、ステップを何度も繰り返す必要があります。

振り直してキーの値を、振り直す前にその行を参照してしたすべての子レコードに反映する必要もあります。この作業は外部キー定義でON UPDATE CASCADEを宣言している場合は簡単ですが、宣言していない場合は、外部キー制約を一時的に無効にし、手作業ですべての子レコードを更新し、再び制約を有効化しなければなりません。

## 解決策：擬似キーの欠番は埋めない

主キーの値は、一意で非NULLの値でなければなりません。各行を識別できなくてはならないからです。しかし、主キーのルールはそれだけです。行の識別のために連続している必要はないのです。

### GUIDの使用

同じ数を複数回使用しないために、ランダムな擬似キー値を生成する方法もあります。一部のデータベースは、グローバル一意識別子（Globally Unique IDentifier: GUID）をサポートしています。

GUIDは128ビットの擬似乱数です。同じ識別子が生成される可能性が極めて低い為に、GUIDは事実上、一位の値であるとみなされます。

GUIDは従来型の擬似キージェネレーターと比べて、以下の利点をもたらします。

- 複数のデータベースサーバー間で、重複した値を生成することなく、変更して擬似キーを生成できる
- 欠番に関する不満を誰も口にしなくなる（ただし、主キーの値を32この16進数として入力しなければならないことをぼやく人は増えるでしょう）

上記の2番目は、デメリットにもなります。

- 値が長いため、タイプしずらくなる
- 値がランダムなので、値からパターンを推測したらい、値の大小から生成された順番を推測したりできない
- GUIDの格納には16バイトが必要です。このため、一般的な4バイト整数の擬似キーと比べて多くのスペースが必要になり、実行時間も長くなる

## まとめ

擬似キーは、行の一意に識別するためにあります。行番号と混同しないようにしましょう。

## シー・ノー・エビル（臭い物に蓋）

## 目的

開発者は少ないコードでクールに仕事をしたいと思います。簡潔なコードには、より合理的ないくつかの理由があります。

- より短い時間でアプリケーションのコーディングができる
- テスト、文章化、ピアレビューの対象となるコードの量が減る
- コードが少ないので、バグが混入する可能性も少ない

このような理由があるため、開発者はほとんど本能的に、不要なコードをできるだけ削除しようとします。

## アンチパターン：肝心な部分を見逃す

アンチパターンに陥るときは、2つのパターンがあります。1つ目は、データベースAPIの戻り値を無視すること。もう1つは、アプリケーションコード内に点在するSQLしか読まなみいことです。

## 解決策：エラーから優雅に回復する

- データベースからエラーが返ってくることを想定して例外チェックやエラーハンドリングをしましょう（コードが多少長くなったとしても）
- デバックにはSQLクエリを構築するコードではなく、実際に構築されたSQLクエリを使用することも重要です
	- 一時変数を使ってSQLクエリを構築してからデータベースAPIに渡しましょう。そうすることでSQLの入った変数を使用前に調べられるようになります。
	- アプリケーションとは別の出力先にSQLを出力するようにしましょう。例えば、ログファイル、IDEのデバックコンソール、デバック情報を表示するブラウザ拡張などです。
	- ウェブアプリケーション出力のHTMLコメント内にSQLクエリを表示しないようにしましょぅ。

ORMフレームワークを使用する場合は、生成したSQLをログに出力することで問題を解決できます。

## まとめ

コードのトラブルシューティングは、それだけで十分に大変な作業です。闇雲に進めても、作業を遅らせるだけです。

## ディプロマティック・イミュニティ（外交特権）

## 目的

開発者は、以下のようなソフトウェアエンジニアリングのベストプラクティスに努めて従おうとします。

- SubversionやGitなどのツールを用いて、ソースコードのバージョン管理を行う
- ユニットテストや機能テストを自動化し、実行する
- ドキュメント、仕様書、コードコメントを書き、アプリケーションの要件や実装戦略を記録する

経験のある開発者は、ベストプラティスを怠るとプロジェクトが失敗に向かって進み出すことを知っています。

## アンチパターン：SQLを特別扱いする

アプリケーションコードの開発ではベストプラクティスを受け入れる開発者であっても、データベースコードではこれらの慣行が免除されると考える傾向があります。「アプリケーション開発のルールはデータベース開発には当てはまらない」このような、考えをディプロマティック・イミュニティ（外交特権）と名付けました。

## 解決策：包括的に品質問題に取り組む

品質保証は3つの部分からなります。

1. プロジェクト要件の明確な定義・文章化
2. 要件に対する解決策の設計・構築
3. 解決策が要件を満たしていることの確認・テスト

データベース開発における品質保証は、文章化、バージョン管理、テスティングのベストプラクティスに従うことで達成できます。

### 文章化

データベースの要件と実装は、アプリケーションコードと同じように文章化すべきです。以下のチェックリストを使ってデータベースを文章化しましょう。

- ER図
	- テーブルとその関連を表すER図を書くようにしましょう。
	- SQLスクリプトや、稼働中のDBからER図を生成するツールもあります。
	- 非常に多くのテーブルを持つ複雑なデータベースは、複数のER図に分けましょう。
- テーブル、列、ビュー
	- テーブルが表現しているエンティティの分類についての説明が必要です。
		- 参照テーブルや、交差テーブル、従属テーブルにはテーブル名からエンティティを特定しにくいので、説明が必要です。
	- 各行で想定している行数や、テーブルに対して実行されるクエリ、テーブルに構築するインデックスについても記述します。
	- 定量的な値を格納する列で用いる単位や、NULLを許容するか、一意性制約があるかなども記述しましょう。
	- ビューの目的、使用が想定されるアプリケーションやユーザー、テーブル間の関連を要約する意図があるか、列のサブセットを参照できるようにするか、ビューは更新可能かなどを記述します。
- 関連（リレーションシップ）
	- トリガー
		- データのバリデーションの変換、データベース変更のロギングなどがあります。
		- トリガーに実装しているビジネスルールなどを記述しましよう。
	- ストアドプロシージャ
		- APIの文章化と同じように、文章化しましよう。
	- SQLセキュリティ
		- システムレベルのセキュリティ対策、不正アクセス検出・防御処理、SQLインジェクション脆弱性対策のための徹底したコードレビューを有無などを記述します。
	- データベースインフラストラクチャ
		- データベースの製品とバージョン
		- データベースサーバーのホスト名
		- データベースサーバの冗長化
		- ネットワーク構成とポート番号
		- クライアントアプリケーションが使用すべき接続オプション
		- データベースユーザーのパスワード
		- バックアップポリシー
		- ↑などを記述しましょう。
	- オブジェクトリレーショナルマッピング（ORM）
		- ORMライブラリを使ったクラス群を通して、実装するかもしれません。
		- 実装されるビジネスルールは何かを記述しましょう。
		- データの妥当性確認、データ変換、ロギング、キャッシュ扱い、プロファイルの取り方などについても記述しましょう。

文章を書くことは手間がかかり、最新状態に維持することも簡単ではありません。しかし、百戦錬磨のプログラマーでさえ、ソフトウェアの他の部分は文章しない場合でも、データベースを文章化する必要があることを知っています。

### バージョン管理

以下のようなデータベース開発関連ファイルを、バージョン管理システムの管理下に入れましょう。

- データ定義スクリプト
- トリガーとプロシージャ
- ブートストラップデータ（シードデータ）
- ER図とドキュメント
- データベース管理スクリプト

### テスティング

品質保証の最終パートは、品質管理です。すなわち、アプリケーションが設計通りに動作することの検証です。テスティングの重要な原則の1つは独立（isolation）です。

アイソレーションテストは、データベースの構造と振る舞いの妥当性確認を、アプリケーションコードとは独立させて行うことができます。

データベースの妥当性を検証するテストのためにチェックリストを以下に示します。

- テーブル、列、ビュー
- 制約
- トリガー
- ストアドプロシージャ
- ブートストラップデータ
- クエリ
- ORMを使用したクラス

## まとめ

アプリケーションと同じく、データベースに対しても、ソフトウェア開発のベストプラクティスを適用し、文章化、テスト、バージョン管理を行いましょう。

## マジックビーンズ（魔法の豆）

## 目的

MVCのM（モデル）を単純化する

モデルを過度に単純化し、モデルは単なるデータアクセスオブジェクトに過ぎないと見做してしますことがあります。

## アンチパターン：モデルがアクティブレコードそのもの

シンプルなアプリケーションでは、モデルに多くのカスタムロジックは不要です。オブジェクトに必要な操作は、行の作成、読み込み、更新、削除のCRUD操作です。

このようなマッピングをサポートするデザインパターンを「 **アクティブレコード（Active Record）** 」と名付けられています。

### アクティブレコードはモデルをデータベーススキーマに強く依存させてしまう

アクティブレコードクラスは、1つのテーブルまたはビューを表現するからです。例えば16のテーブルがある場合、16のモデルを定義することになります。

これはつまり、データベースのスキーマを変更する場合には、モデルクラスだけでなく、そのモデルクラスを使うアプリケーションコードも変更する必要があります。

### アクティブレコードはCRUD機能を公開してしまう

意図された用法を無視して、CRUD操作メソッドを通して直接データを更新してしまうことがある。

例えば、 `assigned_to` が登録されたら電子メールを送信するメソッドをモデルに追加します。しかし、このメソッドを迂回し、電子メールを送信せずに `assigned_to` を登録するコードを書いてしまいます。

### アクティブレコードはドメインモデル貧血症をもたらす

多くモデルが基本的なCRUDメソッド以外の振る舞いを持たないという問題です。モデルをシンプルなデータアクセスオブジェクトとして扱うと、モデルの外部でビジネスロジックのコードディングが必要です。結果として、モデルの振る舞いの凝集度が低下します。

### マジックビーンズのユニットテストは困難

MVCの各レイヤのユニットテストが難しくなります。

- モデルのテスト
	- モデル自身の振る舞いのテストを、データベースから分離して行うことができない
- ビューのテスト
	- 特定のHTML要素のレンダリングと解析のために、フレームワークは複雑で時間のかかるコードを実行しなければならない
- コントローラのテスト
	- 複数のコントローラにおけるコードの重複を招くため、コントローラのテストも複雑になる

## 解決策：モデルがアクティブレコードを「持つ」ようにする

### モデルを理解する

- 情報エキスパート
	- 操作の責任を持つオブジェクトは、その操作を果たすために必要なデータを持つべきです。
	- アクティブレコードのようなDAOとモデルとの間の関係は継承ではなく集約であるべきです。
- 生成者（Creator）
	- モデルがデータベース内のデータを扱う方法は、外部に公開されない、内部実装の詳細であるべき
	- DAOを集約するドメインモデルが、これらのデータオブジェクトを生成する責任を持つべき
	- コントローラとビューは、ドメインモデルのインタフェースを使用するべき
- 疎結合性
	- 論理的に独立しているコードは、分離して疎結合性を行うことが重要
	- コードの利用者に影響を与えずに、クラスの実装を柔軟に変更できるようになる
- 高凝集性
	- ドメインモデルクラスのインタフェースは、物理的なデータベース構造やCRUD操作ではなく、意図を示すべき
	- モデルが使用するDAOから分離すると、同じDAOを使う複数のモデルクラスを設計できるようになる

### ドメインモデルの使用

モデルは、オブジェクト指向によって対象ドメインをアプリケーションの中に表現することです。モデルとは、アプリケーションのビジネスロジックを実装する場所です。データベースとのやり取りは、、モデル内部的な実装の詳細なのです。

### プレーンなオブジェクトのテスト

理想的なのは、本物のデータベースに接続することなくモデルをテストできることです。モデルからDAOを分離させるとDAOのスタブやモックを作成できるようになり、モデルのユニットテストをデータベースから独立して行うことができます。

### 現実的に考える

フレームワークがマジックビーンズのアンチパターンを招きやすものであっても効果的に使用できます。ただし、スパゲッティコードを書いてしまわないように気をつけましょう。

この章で見てきたドメインモデリングの基本は、テストとコードの保守性を高め、継続的開発を行うために最適な設計に役立ち、アプリケーションの開発の生産性を、大いに高めることができる。

## まとめ

モデルはテーブルから分離させましょう。

## 砂の城

## 目的

サービスの安定稼働です。様々なサービスが24時間365日の連続稼働を前提で提供されています。

## アンチパターン：想定不足

問題は、どのようなことが起きるかという想定と、それぞれの事象への対策が十分に検討されていないことです。サービスを安定稼働させるには、トラブルは当然起きるものとして想定しておく必要があります。

## 解決策

重大なインシデントを回避し、サービスの安定稼働を目指すには、どのようなトラブルが起こりうるかということを可能な限り想定しておくことです。 **トラブルは必ず起きます。**

ここでは、サービスの運用を始めるにあたって実施・想定しておくべき代表的な対策について紹介します。

### ベンチマーク

大きなトラフィックが予想されるシステムでは、事前にどの程度まで処理が可能なのかということをベンチマークしておきましょう。

### テスト環境の構築

サービスをリリースしてからが本当のデバックの始まりです。利用しているミドルウェアやデータベース管理システムのトラブルシューティングでもテスト環境は大活躍します。

テスト環境に用いるシステムは、本番環境で利用しているものと同じものを1セット用意するのが理想です。

### 例外処理

データベース管理システムを用いたアプリケーションでは適切な例外処理を実装することが必須です。例えばトランザクションのデットロックや（ロック待ちの）タイムアウトはどの製品でも起きるエラーである、トランザクションをリトライする例外処理を仕込んでおくのが一般的です。データベースサーバーへの接続が切れた場合の対処や、データベース製品固有の一般的なエラーへの対処が必要となります。

### バックアップ

サービスの最後の生命線がバックアップです。ディスク装置の故障やオペミス、クラッカーによる攻撃などによって本番環境のデータが破壊されるというインシデントは起こりえます。

ディスクの冗長化はバックアップにはなりません。データが論理的に破壊されてしまうようなケースに対応できないからです。

### 高可用性

どれだけ高価なマシンを使おうと、マシンそのものの故障から完全に逃れることはできません。停止時間をできるかぎり短くするには、マシンを冗長化する仕組みを考えておく必要があります。

一般的に高可用性構成と言えば、クラスタリングソフトウェアを用い、マシンがクラッシュした際に、フェイルオーバーさせるものを指します。この場合、フェイルオーバー時にサービスを引き継いだ方のマシンにおいてファイルシステムやデータベースのクラッシュリカバリが行われます。

### ディザスタリカバリ

本当に重要なシステムでは、マシン単位の冗長化では十分でなく、データセンターないしはサイト全体の障害も考慮する必要があります。自然災害などにより、データセンターの一部又は、全体が使用不能になってしまった場合には、他のデータセンターで処理を引き継ぐというディザスタリカバリまで考慮しておくと良いでしょう。

### 運用ポリシーの策定

様々なテクニックを駆使して、問題が生じても自動でサービスを継続できる仕組みは素晴らしいですが、すべての事象をカバーできるわけではありません。

- 高可用性の限界を超えた障害
- 問題の調査
	- 特に再現性がない、トランザクションが失敗する程度のエラー情報の採取は困難
- 性能の劣化
	- 処理に時間がかかるようになったり、マシンのリソースが限界まで消費されたら最も高価的なのはクエリやスキーマをチューニングすること
	- 根本的にアプリケーションのアーキテクチャを見直す必要があるのか、その際RDBMS製品を入れ替えるのか言った検討を事前にしておくようにする

## まとめ

どれだけ盤石な基盤を築けるかは、あなたがどれだけインシデントを想定しているかが鍵なのです。

## おわり

もっと詳しく知りたい方は是非本買って読んでみてください。  
8日目は [@inoriko711](https://qiita.com/inoriko711) さんの「弊社エンジニア初、産休取得してみた」です。お楽しみに！

9

2