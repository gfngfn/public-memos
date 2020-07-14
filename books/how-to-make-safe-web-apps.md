
## 3.1 HTTPとセッション管理

* `Referer` ヘッダ
  - リンク元のURLを示す
  - フォームの送信や `a` 要素でのリンク移動のほか `img` 要素の `src` による画像への参照リクエストでもつく
  - URLにセッションIDなどの秘匿情報を含めてしまっている場合，この `Referer` ヘッダを経由して漏洩し，なりすましに使われる．
* **hiddenパラメータ**
  - `<input type="hidden" name="〈Name〉" value="〈Value〉">` でフォーム中に追加できる，GUI上はユーザに見えないパラメータ
  - ユーザが書き換えることはできるが，漏洩や第三者の書き換えに対しては堅牢．
  - 用途上クッキーやセッション変数と比較されることがある．
    * これらはセッションIDの固定化攻撃に対して脆弱．
    * ユーザに書き換えられては困る認証情報はセッション変数に保存すべし（→詳細は5.1節，5.3節）
    * セッション変数は **クッキーモンスターバグ** による漏洩に対する効果的対策がない状況がある（→詳細は4.6.4項）
* クッキー
  - サーバ側からレスポンスの `Set-Cookie` ヘッダで値を送ることによりクライアント側に値を覚えさせる機構．
  - ブラウザが一旦ドメインごとに対応するクッキー値を覚え，リクエストを投げる際にはその送信先のドメインに対応するクッキー値を `Cookie` ヘッダを入れて送る．
  - 長さに制限があり，またユーザが書き換えることもできるので，生のアプリケーションデータそのものを入れて使うことは普通しない．普通は “整理番号” を格納する．この整理番号を **セッションID** と呼ぶ．
  - セッションIDを管理する機構は脆弱性の温床なので，なるべく自作せず実績のある既存実装に頼ることが重要．
* セッションIDに求められる要件
  - 第三者がセッションIDを推測できないこと
    * 推測できると悪人がその推測可能なセッションIDをもつユーザになりすまして認証を通ってしまうため（66頁のなりすましの比喩的事例参照）
  - 第三者からセッションIDを強制されないこと
    * 強制可能な仕組みがあると **セッションIDの固定化攻撃** という手法でなりすましができてしまう（67-68頁の比喩的事例参照）
    * 認証のたびに新しいセッションIDに変更することで防げる．
* セッションIDが漏洩する原因
  - クッキー発行の際の **属性** の指定の不備
    * （これについてはすぐ下で後述）
  - ネットワークに仕掛けられた盗聴機構から漏れる（→8.3節）
    * これはTLSで暗号化して防げるが，それでも属性の指定には注意が必要
  - クロスサイトスクリプティングなど，アプリケーションの脆弱性により漏れる（→4章）
  - ブラウザの脆弱性により漏れる
  - （上で挙げた通り）URLに含めてしまっており `Referer` ヘッダから漏洩する（→4.6.3項）
* クッキーの属性
  - 以下の属性がある
    * Domain
    * …