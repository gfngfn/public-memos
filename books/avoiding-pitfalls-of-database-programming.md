# 『SQLアンチパターン』

## 1章

「リストを表現するのにコンマ区切りの文字列を使うな」という当たり前の話なので割愛．


## 2章 Naïve Tree

replyが木構造をなす，RedditのようなSNSをつくりたいとする．このとき，素朴に考えれば1つの投稿を以下のようなテーブルで格納したくなる：

```sql
CREATE TABLE Comments (
  comment_id SERIAL PRIMARY KEY,
  parent_id  BIGINT UNSIGNED,
  author     BIGINT UNSIGNED NOT NULL,
  comment    TEXT NOT NULL,
  ...
  FOREIGN KEY (parent_id) REFERENCES Comments(comment_id),
  FOREIGN KEY (author) REFERENCES Accounts(account_id)
);
```

しかし，この形式だと（任意の深さになりうる）スレッドを一度に取得する単一のSQLクエリを書くのが原理的に困難で，実現のためには何度もクエリを叩くような実装になる．投稿とその直近の子の組を列挙するのは特に難しくないが，子孫全部を辿るクエリはおそらく書けない：

```sql
SELECT c1.*, c2.*
FROM Comments c1 LEFT OUTER JOIN Comments c2 ON c2.parent_id = c1.comment_id;
```

また，投稿の削除もしばしば厄介．部分木全体を削除したい場合，外部キー制約を満たし続けるために，末端側から順に削除しなければならない．

全ての行を取得してアプリケーションの側で木構造を復元するという素朴なやり方も明らかに非効率なので避けたい．

**隣接リストが有効なのは，直近ないし一定の定数の深さの子孫までしか取得しない用途の場合**．

なお，一部RDBMSでは**再帰クエリ**と呼ばれる仕組みが提供されており，それを使えば任意の深さの子孫を1クエリで取り出すことができるので，将来的には避けなくてもよい設計になるかもしれない．


### 解決策1： 入れ子集合 (nested set)

各投稿が `nsleft`，`nsright` というフィールドをもつようにする：

```sql
CREATE TABLE Comments (
  comment_id SERIAL PRIMARY KEY,
  nsleft     INTEGER NOT NULL,
  nsright    INTEGER NOT NULL,
  author     BIGINT UNSIGNED NOT NULL,
  comment    TEXT NOT NULL,
  ...
  FOREIGN KEY (author) REFERENCES Accounts(account_id)
);
```

- `nsleft`： その投稿の子孫の全てがもつ `nsleft` の値よりも小さな値が格納される
- `nsright`： 自身およびその投稿の子孫の全てがもつ `nsleft` の値よりも大きな値が格納される

このような性質を満たす `nsleft` と `nsright` は，木が与えられれば深さ優先探索で各投稿に簡単に割り当てられる．`nsleft` を行きしに，`nsright` を帰りしなに与えるだけ．こうすると，**或る投稿 `c` の子孫全部は，`c.nsleft < c'.nsleft < c.nsright` なる `c'` を列挙するクエリで取得できる**．

**ただし，挿入・移動といった操作は `nsleft` と `nsright` の再計算で負担が大きいため，これらの動作が頻繁に行なわれるシステムの実装には適さない**．更新の頻度が読み取りに比べてかなり低い場合に有効．


### 解決策2： 閉包テーブル (closure table)

全ての 先祖–子孫 関係の組を `TreePaths` テーブルで保持する（自身も子孫に含む．つまり，親子関係の反射推移閉包が全部陽に保持されている）：

```sql
CREATE TABLE Comments (
  comment_id SERIAL PRIMARY KEY,
  author     BIGINT UNSIGNED NOT NULL,
  comment    TEXT NOT NULL,
  ...
  FOREIGN KEY (author) REFERENCES Accounts(account_id)
);

CREATE table TreePaths (
  ancestor   BIGINT UNSIGNED NOT NULL,
  descendant BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY (ancestor, descendant),
  FOREIGN KEY (ancestor) REFERENCES Comments(comment_id),
  FOREIGN KEY (descendant) REFERENCES Comments(comment_id)
);
```

やはり特定の投稿の子孫全体は容易に1クエリで取得できるし，同様に先祖全体も取得できる．また，オマケとして頂点を複数の木に属させることも自然な拡張で実現できる．

ただし，**木が大きくなるにつれてテーブル自体が急速に大きくなる**（最悪ケースは枝分かれのない木で，2乗のペースで増える）ので，やはりトレードオフがある．


## 3章 ID Required

単に「主キーの設定を設計意図に合わせて適切にしましょう」という話なので割愛．
