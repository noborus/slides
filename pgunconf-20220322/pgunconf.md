---
marp: true
page_number: true
paginate: true
class: lead
---
# <!-- fit -->作ったツールの紹介

[https://github.com/noborus](https://github.com/noborus)

斉藤　登

---


## PostgreSQLに関連した以下のツールを紹介します

* trdsql
* ov
* pgsp
* jpug-doc-tool

全部Goで書かれたCLIツールです。

---

## trdsql

<style scoped>
    section { 
        font-size: 300%;
    }
</style>

PostgreSQLユーザーにこそ使ってほしいtrdsql

[https://github.com/noborus/trdsql](https://github.com/noborus/trdsql)

---

### trdsqlとは？

CSV,LTSV,JSON等にSQLを実行できるツール

```console
trdsql -ih "SELECT id, name, price FROM fruits.csv" 
```

```csv
1,apple,100
2,orange,50
3,melon,500
```

同様のツールはいくつか存在する `q`, `textql`...

trdsqlはDBエンジンを変更できる！

---

### trdsqlはPostgreSQLに接続可能

類似ツールの多くは内部でSQLite3を使用している。

trdsqlもSQLite3を（内蔵して）使用しているが、PostgreSQL,MySQLに変更可能。

`-driver`オプションと接続先を表す`-dsn`オプションで変更可能。

```console
trdsql -driver postgres -dsn "host='/var/run/postgresql/'" "SELECT * FROM fruits.csv"
```

（設定ファイルにも書けます）。

PostgreSQLのSQL構文、SQL関数が使えるのでストレスフリー

---

### trdsqlの速度

処理時間の多くはDBへのINSERT時間。

sqlite3, MySQLはバルクインサートを使用してINSERTしている。

```log
2022/03/10 18:04:36 CREATE TEMPORARY TABLE `testdata/test.csv` ( `c1` text, `c2` text );
2022/03/10 18:04:36 INSERT INTO `testdata/test.csv` (`c1`, `c2`) VALUES (?,?);
2022/03/10 18:04:36 INSERT INTO `testdata/test.csv` (`c1`, `c2`) VALUES (?,?),(?,?);
2022/03/10 18:04:36 SELECT * FROM `testdata/test.csv`
```

PostgreSQLはCOPYを使用している。

```log
2022/03/10 18:04:22 CREATE TEMPORARY TABLE "testdata/test.csv" ( "c1" text, "c2" text );
2022/03/10 18:04:22 COPY "testdata/test.csv" ("c1", "c2") FROM STDIN;
2022/03/10 18:04:22 SELECT * FROM "testdata/test.csv"
```

（`--debug`オプションをつけると実行しているSQLを出力します。）

実は**PostgreSQLを使用するのが一番速い**。
（ネットワークによる遅延が無ければ、倍近く速い）

---

### PostgreSQLのインポート、エクスポート

trdsqlはファイルにSQLを実行するだけでなく、もっといろいろ使える。

それは以下の仕様になっているため。

* 接続したデータベースで渡されたSQLをそのまま実行
（渡すSQLはエスケープ処理の書き換えのみ行う）。
* 指定したテーブルがファイルでなくてもエラーにしない。

ファイルのJOINだけでなく、ファイルとテーブルのJOINも可能。

---

#### エクスポート

そのため既存のテーブルにSQLを実行しても出力可能。

```console
trdsql -driver postgres -dsn "host='/var/run/postgresql/'" -omd "SELECT * FROM actor"
```

| actor_id | first_name |  last_name   |     last_update      |
|----------|------------|--------------|----------------------|
|        1 | Penelope   | Guiness      | 2013-05-26T14:47:57Z |
|        2 | Nick       | Wahlberg     | 2013-05-26T14:47:57Z |
|        3 | Ed         | Chase        | 2013-05-26T14:47:57Z |
|        4 | Jennifer   | Davis        | 2013-05-26T14:47:57Z |
|        5 | Johnny     | Lollobrigida | 2013-05-26T14:47:57Z |

---

#### Tips

trdsqlはいろんな圧縮、伸長をサポートしている。

`-out`でファイル名を指定するときに`.zst`を付けると`zstd`で圧縮される。

```console
trdsql -oh -out w.csv "SELECT * FROM worldcitiespop"
=> 145MB w.csv
```

```console
trdsql -oh -out w.csv.zst "SELECT * FROM worldcitiespop"
=>  44MB w.csv.zst
```

trdsqlでは、gzip, bz2, zstd, lz4, xzで圧縮されたファイルは直接指定可能。

```console
trdsql -ih "SELECT count(*) FROM w.csv.zst"
=>  3173958
```

---

### インポート

trdsqlの動作として、元々ファイルを指定した場合にテンポラリテーブルを作成して、インポートしている。

そのため、テンポラリテーブルではなく実テーブルを指定する機能を開発しようとするが、考慮することが爆発的に増える。

* テーブルがすでに存在している場合…
* 列名を変えたい…
* 型を定義したい…

機能開発は中止。

---

SQLで実行すればよいことに気づいた。

```console
trdsql "CREATE TABLE fruits AS SELECT id::int,name,price::int FROM fruits.csv"
```

`CREATEE TABLE テーブル AS`の代わりに`INSERT INTO テーブル SELECT`を使用すれば既存のテーブルにもインポート可能。

`ON CONFLICT DO NOTHING`や`ON CONFLICT DO UPDATE`を使用すれば、専用のインポートツールよりも柔軟なインポートが可能。

---

#### インポート速度は？

`ファイル ⇒ テンポラリテーブル ⇒ 実テーブル`

という経緯を辿るため、psqlのCSV COPYよりは遅い。

`ファイル ⇒ テンポラリテーブル`はtrdsqlの内部的にCOPYを使用しているため、そこそこ。

`テンポラリテーブル ⇒ 実テーブル`はPostgreSQL内のサーバー側の処理なので、かなり速い。

トータルでもpsqlのCOPY（ノーマルでは最速）の倍ぐらい。
INSERTを組み立てているようなインポートツールよりも数倍高速。

---

#### JSON対応

trdsqlはJSONL(NDJSON）やトップが配列になっているJSONが対象だった。
ただ、そこから外れるけど、リスト形式なJSONはよくある。


```json
{
    "userList": [
        {
            "userID": 1,
            "nickname": "taro",
        },
        {
            "userID": 2,
            "nickname": "hanoko",
        },
        {
            "userID": 3,
            "nickname": "momoko",
        }
    ]
}
```

---

そのままだとusrListという1列1行のテーブルになる。

```console
trdsql -omd "SELECT * FROM sample.json"
```

```at
|                                              userList                                              |
|----------------------------------------------------------------------------------------------------|
| [{"nickname":"taro","userID":1},{"nickname":"hanoko","userID":2},{"nickname":"momoko","userID":3}] |
```

前まではSQLのJSON関数でなんとかするか、jqで前処理をしてパイプで渡せば良いと考えていた。

```console
jq ".userList" sample.json | trdsql -omd -ijson "SELECT * FROM -"
```

---

#### jq構文

`gojq`というGo製のjqクローンが開発されていた。
packageとして使えるので、取り込んでしまった方が便利。

ファイル名`::`の後にjq構文を書くと解釈してからSQLを実行できるようにした。

```console
trdsql -omd "SELECT * FROM sample.json::.userList"
```

| userID | nickname |
|--------|----------|
|      1 | taro     |
|      2 | hanoko   |
|      3 | momoko   |

---
<style scoped>
    section { 
        font-size: 300%; 
    }
</style>

⇒ 次のツール

### PostgreSQLの開発当初からあった問題が2021年に進展したのをご存知でしょうか？

それは…

---

<style scoped>
    section { 
        font-size: 300%; 
    }
</style>

### lessにヘッダーオプションが入った！

（まだベータリリース版）

```console
less --header 2
```

---

lessの前に前提となるPagerの話

### Pager
<style scoped>
    section { 
        font-size: 140%; 
    }
</style>

出力をターミナル（エミュレーター）の表示域に合わせて表示するプログラム。

```console
ls | less
```

のように使用できる。

psqlやmysqlのREPLは自分でパイプに渡せないので内部で使用する。

CLIコマンドでもPagerを呼ぶコマンドが多い。
（むしろ最近のほうが自動で呼ぶ傾向が強い。bat, procs, git, gh...)

多くは環境変数PAGERで設定されたコマンドを使用する。

Pagerに渡った後はPagerの操作になる
⇒psqlを使っていると思っている半分はPagerを操作している。
**psqlにってPagerは大事**

---

### less

http://greenwoodsoftware.com/less/

less(1983年〜)はPostgres(1986年〜)よりも前から存在。

psqlのデータ表示に使用されるPAGERはpsql側では規定していないけど、
lessはPAGERのデファクトスタンダードといえる存在。

いつの間にかGitHubにも置かれていてissue対応してます！
https://github.com/gwsw/less

1983年からの開発で2021年にヘッダーオプションが入った。

それまでは最初に列名を出力してもスクロールすると消えていた。
psqlの出力が画面に収まらないと '\x'で縦に表示しろというのが定番でした。

---

ヘッダーオプションを使用することで、常に列名が表示できるようになった。

![width:800px](img/less.png)

（ヘッダーオプションを使用すると折り返さず横スクロール表示になる）。

---

### pspg

[pspg](https://github.com/okbob/pspg) はテーブル表示に特化したページャー。
pspgを使用することで、列名を常に表示、列の固定を利用して閲覧できる。
テーブル表示なので、画面端でも折り返さないで横スクロールして表示する。

![width:600px](img/pspg.png)

`pspg`はpsqlを想定して作られているので、psqlを使用するならpspgの方が便利。

---

## ov

[https://github.com/noborus/ov](https://github.com/noborus/ov)

私が新しく作った汎用ページャー。

1つの表示方法だけでなく、いろんな表示を動的に切り替えられる。

ヘッダー表示しても折返しができるようにした。
画面幅に収まらなくても横スクロールしないで表示できる。

そのために以下の機能を追加。

* 行背景の交互表示
* 列のハイライト

---

### ovは機能盛りだくさん

* 検索 - インクリメンタルサーチ、正規表現のインクリメンタルサーチ
* フォローモード（tail -f相当）、複数ファイルのフォローモード
* 圧縮ファイル対応（gzip,bzip2,zstd,lz4,xz）
* execモード（標準出力と標準エラー出力を分けて表示）

---

#### PSQL_WATCH_PAGER

psqlの`\watch`はこれまでPAGERに対応していなかったが、開発中のPostgreSQL 15ではPSQL_WATCH_PAGERをセットすればPAGERに出力できる。
pspgが対応（pspgの作者が入れた）。

`\watch`の出力はスクロールして流れていくので複数行出力しての変化は見づらい。
先頭からの位置が変わらないと変化に気づきやすい。
Unixコマンドで言えばpsコマンドを定期的に実行するか、topコマンドを使用するかの違い。

---

実際に`\watch`を実行した結果がPAGERに渡されると以下のような形式でくる。

```
Thu Mar 17 15:53:27 2022 (every 1s)   <------ タイトル
                                      <!----- 空行
 a | b 
---+---
 1 | 2
(1 row)
                                       <!----- 空行（次の行は1秒後に出力）
Thu Mar 17 15:53:28 2022 (every 1s)    <------ タイトル
                                       <!----- 空行
 a | b 
---+---
 1 | 2
(1 row)
                                       <!----- 空行
```

一応空行がきたら終わり。しかし、タイトルの後にも空行…

---

* 1回目の空行でタイトルが終わって結果のはじまり、
* 2回目の空行で結果の終わり

この結果に依存して表示モードを作るのは辛い…空行が途中で入ったり、空行を取りこぼしたりするとズレて表示が崩れやすい。

結果の区切り文字を^L(PageFeed)追加することを提案して、CommitFestにも登録した。

pspg作者のPavel Stehuleさんだけ賛成してくれたけど、他の反応はイマイチ…

ほとんどの開発者は必要としないけど、Pager作る側は必要性を痛感している問題。

---
<style scoped>
    section { 
        font-size: 300%;
    }
</style>

⇒ 次のツール

## pgsp

https://github.com/noborus/pgsp

![width:400px](img/pgsp.png)

---

pg_stat_progess_* という、処理中状況を表すViewを表示するだけの
シンプルなツール

### 対象View

バージョンによって増えていきますが。以下のViewを対象にしています。

* pg_stat_progress_analyze
* pg_stat_progress_basebackup
* pg_stat_progress_cluster
* pg_stat_progress_copy
* pg_stat_progress_create_index
* pg_stat_progress_vacuum

---

```SQL
SELECT * FROM pg_stat_progress_analyze;
```

実行すれば、その時点での状況を教えてくれる。

```
  pid  | datid | datname | relid |         phase         | sample_blks_total |  
   sample_blks_scanned | ext_stats_total | ext_stats_computed | child_tables_total |
    child_tables_done | current_child_table_relid 
-------+-------+---------+-------+-----------------------+------------------
-+---------------------+-----------------+--------------------+-------------------
-+-------------------+---------------------------
 30481 | 16386 | noborus | 25855 | acquiring sample rows |             30000
  |               21320 |               0 |                  0 |                  0
   |                 0 |                         0 
(1 row)
```

結果はViewによって異なります。

sample_blks_scannedがsample_blks_totalまでいけば、（たぶん）処理終了。
別のphaseに移って、違うところの数が増えていく場合もある。

---

処理中にはレコードが追加されて、処理が終わるとレコードが消える。

`psql`の`\watch`を使用すれば監視できるが、前述の通り`\watch`はスクロールして流れていくので変化は見づらい。

前述のPSQL_WATCH_PAGERによって改善するかもしれないけど人にやさしくない。

プログレスのViewなんだからプログレスバーを表示する監視ツールがあるでしょう
⇒無かった。

---

### pgspの特徴

そこで作ったのが`pgsp`

* 1つのviewだけでなく、複数のview（デフォルトは全部）に対して定期的に問い合わせる。
* 処理中はわかる範囲でプログレスバーを表示。
* レコードが消えても指定した秒数間は表示し続ける。
* ターミナルの表示域（幅、高さ）によって、表示方法を変更。
* オプションで、監視間隔、終了してから表示し続ける秒数等に対応。

---
<style scoped>
    section { 
        font-size: 300%; 
    }
</style>

⇒ 次のツール

## jpug-doc-tool

PostgreSQLマニュアル翻訳のためのツール

https://github.com/noborus/jpug-doc-tool/

---

### 自己紹介

PostgreSQLマニュアル翻訳プロジェクトに参加している斉藤です。

PostgreSQLマニュアル翻訳がGitHubに移行してからビルドツールの修正等を担当しています。

* UTF-8への変更
* ビルド環境の修正
* CIの整備
* CSS、HTMLのヘッダーフッター
* PDF作成
* https://pgsql-jp.github.io/ の管理

翻訳もやってますが、英語は得意じゃないです。

---

### jpug-doc

https://github.com/pgsql-jp/jpug-doc

PostgreSQLマニュアル翻訳プロジェクトはjpug-docという名前で管理されています。

詳しくはQiitaの[PostgreSQL日本語マニュアルについて](https://qiita.com/noborus/items/03f98e43c216d7e23767)を参照してください。

---

<style scoped>
    section { 
        font-size: 130%;
    }
</style>

### jpug-doc-tool誕生のきっかけ

PostgreSQL 13のマニュアル翻訳には終わらない危機があった。
中身は変わらないが表記法が変わって変更量が爆発した。

一番影響が大きかったfunc.sgml。

| バージョンアップ | 変更行数 |
|:-----------|----------:|
| 11.0⇒12.0    |    2,168 |
| 12.0⇒13.0    |   36,575 |

変更（+,-）の行数が実際の行数を上回った。

| バージョン | 行数 |
|:-----------|----------:|
| 12.0 | 22,673 |
| 13.0 | 21,153 |

---
<style scoped>
    section {
        font-size: 130%;
    }
</style>

### 実際の変更例

内容は変わらないが、タグが変わって、
13のマニュアルに対して12の翻訳をマージできず

```xml
<entry>round()</entry>
<entry>numeric</entry>
<!--
    <entry>Rounds to nearest integer</entry>
-->
    <entry>最も近い整数への丸め</entry>
```

を以下のようにする必要がある。

```xml
<entry>
    <para>round</para>
    <para>numeric</para>
    <para>
<!--
      Rounds to nearest integer
-->
最も近い整数への丸め
    </para>
</entry>
```

---

英語と日本語のペアのリストを抽出して、
新しいバージョンで置き換える方法にした。

```txt
en:Rounds to nearest integer
ja:最も近い整数への丸め
```

実装は正規表現バリバリ。
XML処理系で置き換えようとするとインデントが元に戻せなかった…

数パターンに対応したので、他の人に使えるように体裁を整えたのが
`jpug-doc-tool`

---

### jpug-doc-toolの変遷

#### チェック機能を追加

英語と日本語のペアのリストにより以下のチェックが可能になった。

* タグが同じか
* 日本語に含まれている英単語が英語にもあるか
* 数値は同じか

13の翻訳完了後も数十カ所発見（最近http->https等の修正が多くあった）。

---

#### 置き換え機能の強化

完全一致の置き換えだけでなく、類似した文に注意書きを入れて置き換え可能に。

`a SQL` ⇒ `an SQL` のような修正は日本語の修正は必要なかった。

さらにAPIを利用した機械翻訳も！

---

### みんなの自動翻訳＠TexTra

https://mt-auto-minhon-mlt.ucri.jgn-x.jp/

利用規約にオープンソースライセンスの翻訳に使用できることが明言されている。
いわゆるAI翻訳で精度も日々向上している。

アカウント作成する必要はあるが、APIも公開している。

GoからAPIを利用できるライブラリを作成。

https://github.com/noborus/go-textra

jpug-doc-toolにも組み込んだ。

---

アカウント情報を設定ファイルに書いたら、以下を実行すると…

```console
jpug-doc-tool replace --mt brin.sgml
```

未翻訳の箇所を翻訳して置き換える。

```
API...[Some of the built-in operator ] Done
API...[bloom operator classes accept ] Done
API...[Defines the estimated number o] Done
API...[Defines the desired false posi] Done
API...[minmax-multi operator classes ] Done
API...[Defines the maximum number of ] Done
API...[Returns whether all the ScanKe] Done
API...[To write an operator class for] Done
API...[Support procedure numbers 1-10] Done
API...[The minmax-multi operator clas] Done
replace: brin.sgml
```

---

### 「みんなの自動翻訳」を利用した置き換え

```diff
diff --git a/doc/src/sgml/brin.sgml b/doc/src/sgml/brin.sgml
index 7b90452dd8..0c60b2bc79 100644
--- a/doc/src/sgml/brin.sgml
+++ b/doc/src/sgml/brin.sgml
@@ -778,14 +778,24 @@ LOG:  request for BRIN range summarization for index "brin_wi_idx" page 128 was
    <title>Operator Class Parameters</title>
 
    <para>
+<!--
     Some of the built-in operator classes allow specifying parameters affecting
     behavior of the operator class.  Each operator class has its own set of
     allowed parameters.  Only the <literal>bloom</literal> and <literal>minmax-multi</literal>
     operator classes allow specifying parameters:
+-->
+<!-- 《機械翻訳》 -->
+一部の組み込み演算子クラスでは、演算子クラスの動作に影響するパラメータを指定できます。
+各演算子クラスには、使用可能なパラメータの独自のセットがあります。
+パラメータを指定できるのは、<literal>bloom</literal>演算子クラスと<literal>minmax-multi</literal>演算子クラスのみです。
    </para>
 ```

 最後に人がチェックして修正し、《機械翻訳》コメントを消せばOK。

---

### 「みんなの自動翻訳」をコマンドで利用

コマンドラインから翻訳ツールとしても使用できる。

```console
jpug-doc-tool mt "This form is much slower and requires an 
<literal>ACCESS EXCLUSIVE</literal> lock on each table while it is
 being processed."
```

generalNT_en_jaがデフォルトの翻訳エンジンだが、カスタマイズした翻訳エンジンc-1640_en_jaが利用している。

```console
c-1640_en_ja: この形式は非常に遅く、処理中の各テーブルに
<literal>ACCESS EXCLUSIVE</literal>ロックを要求します。
generalNT_en_ja: この形式は非常に遅く、処理中の各テーブルに
<literal>ACCESS EXCLUSIVE</literal>ロックを要求します。
```

---

<style scoped>
    section { 
        font-size: 170%; 
    }
</style>

![bg 50% right](img/end.png)

紹介したツール
（全部MITライセンスです）

**trdsql**
https://github.com/noborus/trdsql

**ov**
https://github.com/noborus/ov

**pgsp**
https://github.com/noborus/pgsp

**jpug-doc-tool**
https://github.com/noborus/jpug-doc-tool

みんなの自動翻訳APIクライアント
Goパッケージ
https://github.com/noborus/go-textra
