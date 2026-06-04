# ブレークポイントとステップ実行

ついにデバッガを作ります。前章で作った `Interpreter` クラスに、ブレークポイントと
ステップ実行を持つデバッガを生やしていきましょう。
鍵になるのは、`exec_stmt` の中に仕込んでおいた `on_line` フックです。
このフックを上書きするだけで、デバッガの中核ができあがります。

## 設計方針：Interpreter を継承する

`Interpreter` を直接書き換えるのではなく、それを**継承**した `Debugger` クラスを作ります。
こうすると、もとのインタプリタは何も変えずに（デバッグ機能なしでも動くまま）、
デバッグ版だけを別に用意できます。継承とは、あるクラスの性質をすべて引き継いだうえで、
一部だけを差し替えたり付け足したりする仕組みです。
ここでは `on_line` メソッドだけを差し替えます。

```ruby
require_relative "toy"     # 前章の Interpreter, Parser, tokenize を読み込む

class Debugger < Interpreter
  def initialize(source)
    super(source)              # Interpreter の初期化（AST の構築など）
    @lines = source.split("\n")  # ソースを行ごとに分けて保持（list 表示用）
    @breakpoints = []          # ブレークポイントを置いた行番号の一覧
    @mode = :step              # 実行モード（後述）
    @step_depth = 0            # next（ステップオーバー）のための基準の深さ
  end
```

`super(source)` は親クラス `Interpreter` の `initialize` を呼び出します。
これで AST の構築や関数の収集はそのまま再利用できます。
あとはデバッガ固有の状態——ソース行、ブレークポイント、実行モードなど——を足すだけです。

## 心臓部：on_line フックで止まるか決める

デバッガの心臓は、`on_line` フックの上書きです。
思い出してください。`exec_stmt` は、文を 1 つ実行する**直前に必ず** `on_line(line, stmt)` を
呼びます。つまり `on_line` は「文の切れ目」を通るたびに呼ばれます。
ここで「いま止まるべきか？」を判断し、止まるべきなら対話プロンプト（REPL）を開けばよいのです。

```ruby
  def on_line(line, stmt)
    stop = @breakpoints.include?(line)              # ブレークポイントに当たった？
    stop ||= true if @mode == :step                 # ステップ実行中なら毎回止まる
    stop ||= (@frames.size <= @step_depth) if @mode == :next  # ステップオーバー
    repl(line) if stop
  end
```

止まる条件は 3 つの「または」で決まります。

1. いまの行にブレークポイントが置かれている。
2. モードが `:step`（ステップ実行）である。このときは毎回止まります。
3. モードが `:next`（ステップオーバー）で、かつコールスタックの深さ `@frames.size` が
   基準の深さ `@step_depth` 以下である。これがステップオーバーの肝で、あとで詳しく説明します。

このどれかが成り立てば、`repl` を呼んで対話を始めます。たったこれだけで、
「ブレークポイントで止まる」「1 行ずつ進む」というデバッガの基本動作が表現できてしまうのです。

> [!NOTE]
> なぜ「文の切れ目」でしか止まらないのか、ここで腑に落ちると思います。
> `on_line` は `exec_stmt` から、つまり `:stmt` で包まれたノードを実行するたびに
> 呼ばれます。第 5 章で「すべての文を `[:stmt, 行番号, 中身]` で包む」と決めたのは、
> まさにこの「止まれる場所」を作るためでした。式の途中（`1 + 2` の `+` のところ）では
> 止まりません。それが、行単位デバッガの自然な粒度なのです。

## 実行モードという考え方

デバッガの「いまどう動くか」を、3 つのモードで表しました。
ステップ実行の 3 種類（[](debugger-features.md) で紹介したステップイン／ステップオーバー）も、
このモードの組み合わせで表現できます。

| モード | 意味 | どこで止まるか |
|--------|------|----------------|
| `:run` | 実行（continue） | ブレークポイントに当たった行だけ |
| `:step` | ステップイン | 次の文（関数の中に入ってでも） |
| `:next` | ステップオーバー | 同じ深さに戻ってきた次の文 |

「止まったあとに次どう動くか」を、対話の中でユーザーがコマンドで選び、
それがモードの切り替えになる——これがデバッガの基本的な動き方です。
止まる → コマンドでモードを決める → 再開する → またどこかで止まる、の繰り返しです。

## ステップオーバーはなぜ「深さ」で実現できるのか

3 つのモードのうち、`:next`（ステップオーバー）だけは少し説明が要ります。
ステップオーバーとは「関数呼び出しに出会っても、その中には入らず、
呼び出しが終わって戻ってきた次の文で止まる」動きでした。

ここで第 5 章の `call_func` を思い出してください。関数を呼ぶとフレームが 1 つ**積まれ**、
関数から戻るとフレームが 1 つ**降ろされ**ました。つまりコールスタックの深さ
`@frames.size` は、関数の中に入ると増え、出ると元に戻ります。

そこでステップオーバーは、こう考えます。「`next` と言われた時点の深さを覚えておく
(`@step_depth`)。そして、いまの深さがそれ**以下**になった最初の文で止まる」。

- 関数呼び出しの中に入ると、深さは `@step_depth` より大きくなります。
  だから関数の中の文では止まりません（条件 `@frames.size <= @step_depth` が偽）。
- 関数から戻ってくると、深さは `@step_depth` に戻ります。
  だから戻ってきた次の文で止まります（条件が真）。

「同じ階か、それより浅い階に来たら止まる」と言い換えると分かりやすいでしょう。
基準の深さは、`next` コマンドを受け取った瞬間に記録します。

```ruby
when "next", "n"
  @mode = :next; @step_depth = @frames.size; return
```

> [!TIP]
> ステップアウト（いまの関数を抜けるまで一気に実行）も、同じ発想で作れます。
> 基準を「いまの深さより 1 つ浅い」に設定すればよいのです
> （`@step_depth = @frames.size - 1`）。本書のコマンドには入れていませんが、
> 練習問題として実装してみてください。深さという 1 つの数だけで、
> ステップイン・オーバー・アウトの 3 種類すべてが表現できると分かります。

## 対話プロンプト（REPL）を実装する

止まったあとにユーザーと対話する部分が `repl` です。
これは [REPL](#index:REPL)（入力を読んで評価して結果を表示する繰り返し）そのものです。
プロンプトを出し、1 行コマンドを読み、その種類に応じて処理し、
実行を再開するコマンド（continue / step / next）が来たらループを抜けて
インタプリタに制御を返します。

```ruby
  def repl(line)
    puts "stop at line #{line}: #{@lines[line - 1].strip}"
    loop do
      print "(dbg) "
      input = $stdin.gets
      return if input.nil?                  # 入力が尽きたら、そのまま実行を続ける
      cmd, arg = input.strip.split(" ", 2)
      case cmd
      when "break", "b"
        @breakpoints << arg.to_i
        puts "breakpoint at line #{arg}"
      when "continue", "c"
        @mode = :run; return                # 再開系コマンドは return でループを抜ける
      when "step", "s"
        @mode = :step; return
      when "next", "n"
        @mode = :next; @step_depth = @frames.size; return
      when "print", "p"
        begin
          ast = Parser.new(tokenize(arg)).parse_expr
          puts "=> #{eval_node(ast)}"
        rescue => e
          puts "error: #{e.message}"
        end
      when "backtrace", "bt"
        @frames.reverse_each { |f| puts "  #{f.name} (line #{f.line})" }
      when "list", "l"
        lo = [line - 2, 1].max
        hi = [line + 2, @lines.size].min
        (lo..hi).each do |n|
          mark = n == line ? "=>" : "  "
          puts "#{mark} #{n}: #{@lines[n - 1]}"
        end
      when "quit", "q"
        exit
      when nil
        # 空行は何もしない
      else
        puts "unknown command: #{cmd}"
      end
    end
  end
end
```

ここで実装したコマンドを整理します。`break N` で N 行目にブレークポイントを置き、
`continue` で次のブレークポイントまで走り、`step` / `next` で 1 文ずつ進めます。
`backtrace` はコールスタックを上から表示し（第 7 章で深掘りします）、
`list` は現在行の前後 2 行を表示し、`print 式` は止まった場所で任意の式を評価します。

`print` コマンドの実装に注目してください。受け取った文字列を `tokenize` して
`Parser#parse_expr` で式の AST にし、`eval_node` で評価しています。
**第 5 章で作った字句解析・構文解析・評価を、そのまま再利用している**のです。
だからこそ `p a` のような変数だけでなく、`p a*b+1` のような複雑な式も計算できます。
さらに `rescue` で囲んであるので、`p nope`（未定義の変数）のような誤りでも、
デバッガごと落ちることなく、エラーメッセージを出して対話を続けられます。

> [!IMPORTANT]
> デバッガが「その場で任意の式を評価できる」のは、評価器を自分の中に持っているからです。
> プロセス制御型のデバッガ（gdb など）が同じことをするのは、実はもっと大変です。
> 対象プロセスのメモリを読み、式を自前で解釈し、必要なら対象に計算をさせる……。
> インタプリタ統合型のデバッガは、この点で圧倒的に有利なのです。

## 動かしてみる

完成したデバッガを動かしましょう。デバッグ対象は次の Toy プログラムです（行番号つき）。

```text
1  def add(a, b) {
2    c = a + b
3    return c
4  }
5
6  x = 10
7  y = 20
8  z = add(x, y)
9  print(z)
```

`Debugger.new(src).run` で起動します。起動するとモードが `:step` なので、
最初の文（1 行目）でいきなり止まります。ここで `add` の中を調べてみましょう。
次のように対話します（`(dbg)` がプロンプト、その右が入力です）。

```text
stop at line 1: def add(a, b) {
(dbg) break 8         ← 8 行目にブレークポイントを置く
breakpoint at line 8
(dbg) continue        ← 8 行目まで一気に進める
stop at line 8: z = add(x, y)
(dbg) step            ← add の中へ「入る」
stop at line 2: c = a + b
(dbg) backtrace       ← どうやってここに来たか
  add (line 2)
  main (line 8)
(dbg) print a         ← 引数 a の値は？
=> 10
(dbg) print a * b + 1 ← その場で式も計算できる
=> 201
(dbg) continue
30                    ← プログラムの出力（print(z) の結果）
```

`step` で関数 `add` の中に入り、`backtrace` で「`main` の 8 行目から `add` が呼ばれた」
という経路が見えています。`print a * b + 1` のように、その場で思いついた式も評価できます。

次に、同じ 8 行目で `step` の代わりに `next` を使うと、どうなるでしょうか。

```text
(dbg) break 8
(dbg) continue
stop at line 8: z = add(x, y)
(dbg) next            ← add の中には入らず、一気に飛ばす
stop at line 9: print(z)
(dbg) print z         ← add の結果はちゃんと入っている
=> 30
```

`add` の中の行（2 行目や 3 行目）では一切止まらず、関数呼び出しを丸ごと飛ばして
9 行目で止まりました。これがステップオーバーです。
深さ `@frames.size` を見るだけで、この振る舞いが実現できているわけです。

> [!NOTE]
> ここで作ったデバッガは、コマンドの種類こそ少ないものの、
> ブレークポイント・ステップイン・ステップオーバー・式評価・バックトレースという、
> 実用デバッガの中核機能をすべて備えています。Ruby の `debug.gem` のような
> 本格的なデバッガも、機能の「考え方」はこれと地続きです[](#cite:sasada2021)。
> 違いは、扱う言語の複雑さと、性能や使い勝手のための作り込みの量です。

次章では、いま `backtrace` でちらりと見せたコールスタックと変数の検査を、
もう一歩踏み込んで扱います。「いまの場所だけでなく、呼び出し元の変数まで覗きたい」
という欲求に応えていきましょう。
