# 一歩進んだ機能

基本のデバッガができたので、ここでは「使えると一気に世界が変わる」3 つの発展機能を作ります。
[条件付きブレークポイント](#index:条件付きブレークポイント)、
[ウォッチポイント](#index:ウォッチポイント)、そして実行を巻き戻せる
[全知デバッガ](#index:全知デバッガ) です。
どれも、インタプリタ統合型ならではの手軽さで実装できることを体験しましょう。

この章では、対話よりも仕組みに集中するため、コマンド入力ではなく
プログラムから設定する形（非対話）でデバッガを動かします。考え方は対話版とまったく同じです。

## 条件付きブレークポイント：特定の 1 回だけ捕まえる

「1 万回まわるループの、`i` がちょうど 100 のときだけ止めたい」。
これが [条件付きブレークポイント](#index:条件付きブレークポイント) (conditional breakpoint) です。
普通のブレークポイントに「条件式」を添え、その条件が真のときだけ止めます。

実装はおどろくほど簡単です。行番号ごとに条件式の AST を覚えておき、
`on_line` でその行に来たら条件を**その場で評価**して、真なら止めるだけ。
条件式の評価には、第 5 章で作った `tokenize`・`parse_expr`・`eval_node` を再利用します。
止まった場所の文脈（最上段フレーム）で評価されるので、ループ変数 `i` などをそのまま条件に書けます。

```ruby
require_relative "toy"

class CondDebugger < Interpreter
  def initialize(source)
    super(source)
    @cond_breaks = {}   # 行番号 => 条件式の AST
  end

  def break_if(line, expr)
    @cond_breaks[line] = Parser.new(tokenize(expr)).parse_expr
  end

  def on_line(line, _stmt)
    if (cond = @cond_breaks[line])
      if truthy?(eval_node(cond))            # 条件をその場で評価
        puts "[break] line #{line}: 条件成立 (#{describe})"
      end
    end
  end

  def describe
    @frames.last.env.map { |k, v| "#{k}=#{v}" }.join(", ")
  end
end
```

`break_if(行, "条件")` で設定します。たとえば、1 から 5 までの合計を求めるプログラムで
「`i` が `3` のときだけ知らせて」と頼んでみましょう。

```text
1  i = 1
2  sum = 0
3  while i <= 5 {
4    sum = sum + i
5    i = i + 1
6  }
7  print(sum)
```

```ruby
d = CondDebugger.new(src)
d.break_if(4, "i == 3")     # 4 行目に来て、かつ i == 3 のときだけ
d.run
```

実行すると、こう表示されます。

```text
[break] line 4: 条件成立 (i=3, sum=3)
15
```

4 行目はループのたびに通りますが、知らせてきたのは `i == 3` の 1 回だけです。
そのとき `sum` がすでに `3`（= 1 + 2）になっていることも確認できます。
**条件の評価に処理系の評価器をそのまま使える**——これがインタプリタ統合型の強みです。

> [!TIP]
> 条件付きブレークポイントは、ループや再帰の「ある特定の状況」だけを狙い撃ちにできます。
> Zeller が説く「仮説を立てて実験する」デバッグ[](#cite:zeller2009) において、
> 「`i` が 100 を超えたあたりで壊れるはず」という仮説を、最小の手間で検証する道具になります。

## ウォッチポイント：値が変わった瞬間をとらえる

次は [ウォッチポイント](#index:ウォッチポイント) (watchpoint) です。
これは「場所」ではなく「値」に着目します。
「この変数の値が変わったら、どこで変わったのか教えて」という機能です。
「いつの間にか変な値になっている」タイプのバグに絶大な効果を発揮します。

実装の考え方は「前回見た値を覚えておき、文の切れ目ごとに今の値と比べる」です。
`on_line` は文の切れ目ごとに呼ばれるので、ここで監視対象の変数を調べ、
前回と違っていれば変化を報告し、新しい値を覚え直します。

```ruby
class WatchDebugger < Interpreter
  def initialize(source)
    super(source)
    @watches = []       # 監視する変数名
    @last = {}          # 前回観測した値
  end

  def watch(name)
    @watches << name
  end

  def on_line(line, _stmt)
    @watches.each do |name|
      env = @frames.last.env
      next unless env.key?(name)
      if @last[name] != env[name]
        puts "[watch] #{name}: #{@last[name].inspect} -> #{env[name]} (line #{line})"
        @last[name] = env[name]
      end
    end
  end
end
```

先ほどの合計プログラムで `sum` を監視してみます。

```ruby
d = WatchDebugger.new(src)
d.watch("sum")
d.run
```

```text
[watch] sum: nil -> 0 (line 3)
[watch] sum: 0 -> 1 (line 5)
[watch] sum: 1 -> 3 (line 5)
[watch] sum: 3 -> 6 (line 5)
[watch] sum: 6 -> 10 (line 5)
[watch] sum: 10 -> 15 (line 5)
15
```

`sum` が `0 → 1 → 3 → 6 → 10 → 15` と変化していく様子が、すべて捕捉できました。
最初の `nil -> 0` は、`sum` がまだ存在しない状態から `0` が代入されたことを表します
（`nil` は「値がない」ことを示す Ruby の特別な値です）。

> [!NOTE]
> ここでは文の切れ目ごとに値を比べる、もっとも素朴な方式をとりました。
> このやり方は分かりやすい反面、文を 1 つ実行するたびに監視変数を毎回チェックするため、
> 監視が多いと実行が遅くなります。`gdb` などの本物のデバッガが「ハードウェア
> ウォッチポイント」（CPU の機能で、特定のメモリ番地への書き込みを検出する仕組み）を
> 使うのは、まさにこの遅さを避けるためです[](#cite:stallman2023)。
> 第 9 章で、この「仕組みの違い」をもう少し掘り下げます。

## 全知デバッガ：実行を巻き戻す

最後は、本書でいちばんわくわくする機能です。普通のデバッガは前にしか進めません。
「行きすぎた、さっきの状態に戻りたい」と思っても、最初からやり直すしかありませんでした。

Lewis は、実行中の状態変化を**すべて記録**しておけば、あとから好きな時点の状態を
何度でも見られる、というアイデアを [全知デバッガ](#index:全知デバッガ) (omniscient debugger) として
提案しました[](#cite:lewis2003)。「全知」とは、実行の全歴史を知っている、という意味です。
インタプリタの中にいる私たちにとって、状態の記録は造作もありません。
`on_line` で、各ステップの「行番号・関数名・変数の値」を履歴に書き留めるだけです。

```ruby
class RecordingDebugger < Interpreter
  def initialize(source)
    super(source)
    @history = []
  end

  def on_line(line, _stmt)
    snapshot = @frames.last.env.dup     # その瞬間の変数の値を複製して保存
    @history << [line, @frames.last.name, snapshot]
  end

  def dump_history
    @history.each_with_index do |(line, name, env), i|
      puts "##{i} line #{line} [#{name}] #{env.inspect}"
    end
  end
end
```

`env.dup` がポイントです。`env`（変数表）は実行とともに書き換えられていくので、
そのまま記録するとすべて同じ最終状態を指してしまいます。`dup`（複製）で、
**その瞬間のスナップショット（その時点の値のコピー）**を残すのが肝心です。

合計プログラムを記録して、履歴を一覧してみましょう。

```ruby
d = RecordingDebugger.new(src)
d.run
d.dump_history
```

```text
15
#0 line 1 [main] {}
#1 line 2 [main] {"i" => 1}
#2 line 3 [main] {"i" => 1, "sum" => 0}
#3 line 4 [main] {"i" => 1, "sum" => 0}
#4 line 5 [main] {"i" => 1, "sum" => 1}
#5 line 4 [main] {"i" => 2, "sum" => 1}
#6 line 5 [main] {"i" => 2, "sum" => 3}
#7 line 4 [main] {"i" => 3, "sum" => 3}
 ...（中略）...
#13 line 7 [main] {"i" => 6, "sum" => 15}
```

実行の全行程が、ステップ番号つきで残りました。`#5` を見れば「2 周目に入った瞬間、
`i` は `2`、`sum` はまだ `1`」と、過去の任意の時点の状態を、いつでも振り返って観察できます。
ステップ番号を前後にたどれば、それはもう「時間をさかのぼるデバッグ」です。
この履歴の上に「`step` / `back`（1 つ戻る）」コマンドをかぶせれば、
リバースデバッガのできあがりです。

> [!IMPORTANT]
> このやり方は、すべての状態を記録するためメモリを大量に消費します。
> 短いプログラムなら問題ありませんが、長時間動くプログラムには向きません。
> そこで実用的な巻き戻しデバッグでは、「全状態を記録する」のではなく、
> 「最小限の情報だけ記録して、必要なときに同じ実行を再現する」という
> [レコード&リプレイ](#index:レコードリプレイ) の発想が使われます。
> `rr` はこの方式で、現実のプログラムを低コストで記録・再生し、
> 巻き戻しデバッグを実用化しました[](#cite:ocallahan2017)。
> 素朴な全知デバッガと実用的な `rr`、両者は「過去を観察する」という目的を共有しつつ、
> コストの払い方が違うのです。

## ここまでで作ったもの

第 6 章から本章まで、私たちは小さなインタプリタの上に、次の機能を積み上げてきました。

- ブレークポイント（行・条件付き）
- ステップ実行（ステップイン・ステップオーバー）
- 変数の検査とコールスタックの探索（バックトレース・フレーム移動）
- ウォッチポイント
- 実行履歴の記録（全知デバッガ）

これらはすべて、`exec_stmt` に仕込んだ**たった 1 つのフック `on_line`** から生えています。
「実行ループに穴を 1 つ開け、そこから観察と制御をおこなう」——
インタプリタ統合型デバッガの設計思想を、手を動かして体得できたはずです。

次章では視点を変え、私たちのおもちゃのデバッガと、`gdb` や Ruby の `debug.gem` のような
**本物のデバッガ**を並べて、どこが同じでどこが違うのかを確かめます。
