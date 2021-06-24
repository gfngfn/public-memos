
## まえがき

TDDのルールは以下の2つだけ：

- 自動テストが失敗したときのみ新たに実装のコードを書く
- 重複を除去する

これにより以下の効果がある：

- 動作するコードが設計にフィードバックをもたらし，設計が有機的に進展する
- 誰かが書くのを待っていられないので各自が自分でテストを書くようになる
- 小さい変更に迅速に追従するために開発環境を整えるようになる
- テストの描きやすさのために結合度の低い多くのコンポーネントに分けた設計を行なうようになる

2つのルールに基づく具体的な作業の順番：

1. red： まずテストをひとつ書く．全てのテストを走らせ，新しいテストの失敗を確認する．
2. green： そのテストが通る実装を書く．ここでは重複を生じる罪を犯してよい．そして全てのテストを走らせ，全て成功することを確認する．
3. refactoring： テストを通すために生じた重複を全て除去する

TDDは「プログラミング中の不安をコントロールするための手法」．不安はチームの各個人のためらい，コミュニケーション不全，フィードバックからの逃避，苛立ちに繋がり，これを管理可能にし減退させるのがTDDの目的．

本の構成：

1. 多国通貨： 具体的なコード例によるTDDの解説
2. xUnit： リフレクションや例外などより複雑なロジックに対するテストをxUnitを介して学ぶ
3. テスト駆動開発のパターン： どのようなテストを書くかの判断に関するパターンなどについて

全体を俯瞰したいなら第3部を読み進めながら第1部・第2部を実例として読んでもいい．


## 第1部 多国通貨

この章で以下の非自明さを実感する：

- 少しずつ増えていく機能を各テストがどうやって支えるか
- いかに小さく不恰好な変更でテストを通すか
- いかに頻繁にテストを走らせるか
- いかに小さく刻んでリファクタリングするか


### 1. 仮実装

まず大体どんなテストケースをつくるかのTODOリストを書く．

> - [ ] $5 + 10 CHF = $10（スイス・フランと米ドルの交換為替が 2:1 の場合）
> - [ ] **$5 × 2 = $10**

テストのTODOリストの記述の凡例：

- **太字**： 次にとりかかる項目
- ~~打ち消し線つき~~： 終わった項目

2番目の例の方が簡単なので，まずこれに取り掛かる．ひとまず以下のようなテストコードを書く：

```java
package money;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

public class MoneyTest {
  @Test
  public void testMultiplication() {
    Dollar five = new Dollar(5);
    five.times(2);
    assertEquals(10, five.amount);
  }
}
```

`amount` がpublicなフィールドになっていたり，変数 `five` を破壊的に変更していたり，いずれ丸めが必要になるはずなのに `int` 型で金額を扱っていたりと不備は多いが，小刻みに進めるためまずはこれをたたき台にしている．気づいたことをTODOリストに追加すると以下のようになる：

> - [ ] $5 + 10 CHF = $10（スイス・フランと米ドルの交換為替が 2:1 の場合）
> - [ ] **$5 × 2 = $10**
> - [ ] `amount` をprivateにする
> - [ ] `Dollar` の副作用をなんとかする
> - [ ] `Money` の丸め処理に対応する

とはいえその前に上記のテストはコンパイルも通らない．`Dollar` クラスとそのコンストラクタ，`times` メソッド，`amount` フィールドがないため．というわけで実装の枠組みを与える：

```java
class Dollar(int amount) {
  int amount;
  Dollar(int amount) {
  }
  void times(int multiplier) {
  }
}
```

何も実装がないが，コンパイルは通る．まずはこれでテストが落ちることを確認する．`assertEquals` で想定が `10` だったが結果が `0` である旨で落ちる．ここまでで手順のredが完了．

続いてgreenの手順に移る．テストを通す最小の変更は以下：

```java
class Dollar(int amount) {
  int amount = 10;
（略）
}
```

こんな間抜けな変更があるかと思うがこれでテストは通ってgreenは終了．

最後のrefactoringでまともなテストに書き換える．以降はわざと異様に細かいステップで改善していく．まず実装として `10` がどのように求められるべきか考えると以下になる：

```java
class Dollar(int amount) {
  int amount = 5 * 2;
（略）
}
```

結果は `times` の戻り値なのだから `amount` は `times` 内で計算すべきである：

```java
class Dollar(int amount) {
  int amount;
  Dollar(int amount) {
  }
  void times(int multiplier) {
    amount = 5 * 2;
  }
}
```

`5` はどこから来るべきかというとコンストラクタに渡している値からくるので，コンストラクタ内で代入するようにしてこれに伴い `times` で `amount` を使うべし：

```java
class Dollar(int amount) {
  int amount;
  Dollar(int amount) {
    this.amount = amount;
  }
  void times(int multiplier) {
    amount = amount * 2;
  }
}
```

`2` は `times` の引数の `multiplier` からくるべきなので置き換える：

```java
class Dollar(int amount) {
  int amount;
  Dollar(int amount) {
    this.amount = amount;
  }
  void times(int multiplier) {
    amount = amount * multiplier;
  }
}
```

あとはおまけに少し書き換え：

```java
class Dollar(int amount) {
（略）
  void times(int multiplier) {
    amount *= multiplier;
  }
}
```

これでTODOの項目がひとつ終わった：

> - [ ] $5 + 10 CHF = $10（スイス・フランと米ドルの交換為替が 2:1 の場合）
> - [x] ~~$5 × 2 = $10~~
> - [ ] `amount` をprivateにする
> - [ ] `Dollar` の副作用をなんとかする
> - [ ] `Money` の丸め処理に対応する

というわけで，greenの過程でテストを通す場合の「犯す罪」は “具体的なテスト項目に依存していて抽象度がおかしい” というとんでもない「罪」を含んでいてもよく，refactoringの過程の「重複を除去する」過程には “適切な抽象度に直す” という作業も含まれていたりする．


### 2. 明白な実装

次はこう書けるようにしたい：

```java
public class MoneyTest {
  @Test
  public void testMultiplication() {
    Dollar five = new Dollar(5);
    Dollar product = five.times(2);
    assertEquals(10, product.amount);
    product = five.times(3);
    assertEquals(15, product.amount);
  }
}
```

すなわち取り組むのはこれ：

> - [ ] $5 + 10 CHF = $10（スイス・フランと米ドルの交換為替が 2:1 の場合）
> - [x] ~~$5 × 2 = $10~~
> - [ ] `amount` をprivateにする
> - [ ] **`Dollar` の副作用をなんとかする**
> - [ ] `Money` の丸め処理に対応する

まずは `times` が `Dollar` の戻り値を返すようにする：

```java
Dollar times(int multiplier) {
  return new Dollar(amount * multiplier);
}
```

これで通ってgreenを達成し，かつこれ以上抽象度を適切にする必要はないのでrefactoringも達成：

> - [ ] $5 + 10 CHF = $10（スイス・フランと米ドルの交換為替が 2:1 の場合）
> - [x] ~~$5 × 2 = $10~~
> - [ ] `amount` をprivateにする
> - [x] ~~`Dollar` の副作用をなんとかする~~
> - [ ] `Money` の丸め処理に対応する

1章と2章でそれぞれ見たことを総括すると，TDDでの実装はまず以下の2つのモードがある：

- **仮実装**： ベタ書きの値を使い，実装を進めるに従い変数で抽象化していく
- **明白な実装**： すぐに頭の中の実装をコードに落とす

前者はプログラムを書く一般的な（少なくとも自分が一般的と感じる）手順ではなかなかやらなさそうで，TDDの色濃い特徴に思える．

実はさらに3つ目もあり，それは3章でやる「**三角測量**」．


### 3. 三角測量

クラスのインスタンスを参照等価な値として使うようにすることを（わざわざ）**Value Objectパターン** と呼ぶらしい．フォーマルには，**Value Object** として扱うには `equals()` と `hashCode()` というメソッドが要請される．というわけでTODOが増える：

> - [ ] $5 + 10 CHF = $10（スイス・フランと米ドルの交換為替が 2:1 の場合）
> - [x] ~~$5 × 2 = $10~~
> - [ ] `amount` をprivateにする
> - [x] ~~`Dollar` の副作用をなんとかする~~
> - [ ] `Money` の丸め処理に対応する
> - [ ] `equals()`
> - [ ] `hashCode()`

さて，TDDで次に考えるべきは「`equals()` をどう実装しようか」ではない！ 「`equals()` をどうテストしようかな」である：

```java
public class MoneyTest {
（略）
  @Test
  public void testEquality() {
    assertTrue(new Dollar(5).equals(new Dollar(5)));
  }
}
```

で，まだ `equals()` は実装していないのでredになる．greenにする間抜けな実装は以下：

```java
public boolean equals(Object object) {
  return true;
}
```

で，ここで実装を与えてもよいが，TDDのさらに別の手法「三角測量」を使ってみる．三角測量というのは「コードを一般化できるのは複数の実例があるときだ」という意味のアナロジー．ここでは例えば「$5 = $6　は成り立たない」というテストケースを追加すると落ちる：

```java
public class MoneyTest {
（略）
  @Test
  public void testEquality() {
    assertTrue(new Dollar(5).equals(new Dollar(5)));
    assertFalse(new Dollar(5).equals(new Dollar(6)));
  }
}
```

これで一般化する必要が生じたので実装をマトモにする：

```java
public boolean equals(Object object) {
  Dollar dollar = (Dollar) object;
  return amound == dollar.amount;
}
```

これで通る．ただしこれは `null` や他のクラスのオブジェクトとは比較できない実装なので，それをTODOに加え，次章で `amount` をprivateにする：

> - [ ] $5 + 10 CHF = $10（スイス・フランと米ドルの交換為替が 2:1 の場合）
> - [x] ~~$5 × 2 = $10~~
> - [ ] **`amount` をprivateにする**
> - [x] ~~`Dollar` の副作用をなんとかする~~
> - [ ] `Money` の丸め処理に対応する
> - [x] ~~`equals()`~~
> - [ ] `hashCode()`
> - [ ] `null` との等価性比較
> - [ ] 他のクラスのオブジェクトとの等価性比較


### 4. 意図を語るテスト

`Dollar` に対して等価性が判定できるようになったため，テストをより “意図を語る” 記述へと修正できる：

```diff
 public class MoneyTest {
   @Test
   public void testMultiplication() {
     Dollar five = new Dollar(5);
     Dollar product = five.times(2);
-    assertEquals(10, product.amount);
+    assertEquals(new Dollar(10), product);
     product = five.times(3);
-    assertEquals(15, product.amount);
+    assertEquals(new Dollar(15), product);
   }
 }
```

これにより `amount` は `Dollar` 内部でのみ使用されるフィールドとなり，privateに変更できる：

```diff
 class Dollar(int amount) {
-  int amount;
+  private int amount;
   Dollar(int amount) {
     this.amount = amount;
   }
   void times(int multiplier) {
     amount = amount * multiplier;
   }
 }
```

> - [ ] $5 + 10 CHF = $10（スイス・フランと米ドルの交換為替が 2:1 の場合）
> - [x] ~~$5 × 2 = $10~~
> - [x] ~~`amount` をprivateにする~~
> - [x] ~~`Dollar` の副作用をなんとかする~~
> - [ ] `Money` の丸め処理に対応する
> - [x] ~~`equals()`~~
> - [ ] `hashCode()`
> - [ ] `null` との等価性比較
> - [ ] 他のクラスのオブジェクトとの等価性比較

ただし，ここで掛け算のテストは等価性判定に依存しているため，掛け算のテストの信頼性は等価性判定の信頼性の下に成り立っているという “リスクを受け入れた” ことは意識されねばならない．実際しばしば欠陥を見逃してしまうことはある．


### 5. 原則をあえて破るとき

1番目のTODOにそろそろ取り掛かりたいが，いきなりやるには重い．そこで `Dollar` と同様の `Franc` が実現できるサブゴールを設ける：

> - [ ] $5 + 10 CHF = $10（スイス・フランと米ドルの交換為替が 2:1 の場合）
> - [x] ~~$5 × 2 = $10~~
> - [x] ~~`amount` をprivateにする~~
> - [x] ~~`Dollar` の副作用をなんとかする~~
> - [ ] `Money` の丸め処理に対応する
> - [x] ~~`equals()`~~
> - [ ] `hashCode()`
> - [ ] `null` との等価性比較
> - [ ] 他のクラスのオブジェクトとの等価性比較
> - [ ] **5 CHF × 2 = 10 CHF**

まずは `Franc` のテストをつくる：

```java
public class MoneyTest {
（略）
  @Test
  public void testFrancMultiplication() {
    Franc five = new Franc(5);
    assertEquals(new Franc(10), five.times(2));
    assertEquals(new Franc(15), five.times(3));
  }
}
```

さて，これでレッドになったので，グリーンにする．この段階はとにかくテストを通せばよいという段階なので，罪を犯して `Dollar` を単にほぼ複製して `Franc` をつくればよい（コード略）．重複の削減は次の章で行なう．


### 6. テスト不足に気づいたら
