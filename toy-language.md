# 小さな言語処理系をつくる

ここからは手を動かします。デバッガを作るには、まずデバッグされる側、
つまり**言語処理系**が必要です。この章では、Ruby で小さなインタプリタを作ります。
名前を **Toy**（トイ）言語と呼ぶことにしましょう。
この章のコードはすべて実際に動かして確認したものです。

> [!NOTE]
> この章だけは、デバッガの話が一切出てきません。
> しかし急がば回れです。デバッガは処理系の内部に手を入れて作ります。
> 内部を自分で理解していないものに手は入れられません。
> ここで作る処理系が、第 6 章以降のすべての土台になります。

## インタプリタの 3 段階

ソースコードはただの文字の並びです。`x = 1 + 2` という文字列を見ても、
Ruby（や CPU）はそれが「足し算して代入する」という意味だとは分かりません。
そこでインタプリタは、文字列を 3 つの段階を経て実行可能な構造へと変えていきます。
この流れは、本格的なコンパイラの教科書『Compilers: Principles, Techniques, and Tools』
（通称ドラゴンブック）でも最初に説明される、言語処理系の王道の構成です[](#cite:aho2006)。

```mermaid
graph LR
    A["ソースコード<br/>(文字列)"] -->|字句解析| B["トークン列<br/>(単語の列)"]
    B -->|構文解析| C["抽象構文木<br/>(木構造)"]
    C -->|評価| D["実行結果"]
```

- [字句解析](#index:字句解析) (lexical analysis): 文字の並びを、意味のある最小単位
  ([トークン](#index:トークン)) に区切る。`x = 1 + 2` を
  `x` `=` `1` `+` `2` という 5 つのトークンに分ける作業です。
- [構文解析](#index:構文解析) (parsing): トークンの並びを、文法に従って木構造に組み立てる。
  この木を [抽象構文木](#index:抽象構文木) (Abstract Syntax Tree, [AST](#index:AST)) と呼びます。
- [評価](#index:評価) (evaluation): できあがった木をたどりながら、実際に計算を実行する。

Toy 言語は、この 3 段階を素直に実装した [ツリーウォーク型インタプリタ](#index:ツリーウォーク型インタプリタ)
(tree-walking interpreter) です。AST という木を歩き (walk) ながら実行するので、こう呼びます。
もっとも理解しやすい実行方式で、教育用の言語実装で広く採用されています[](#cite:nystrom2021)。

## Toy 言語の仕様

まず、これから実装する言語がどんなものかを決めておきます。Toy 言語は、
整数だけを扱う、ごく小さな手続き型言語です。

```text
# これは Toy 言語のプログラム例（# から行末はコメント）
def add(a, b) {
  c = a + b
  return c
}

x = 10
y = 20
z = add(x, y)
print(z)
```

仕様を箇条書きにします。

- 値は整数のみ。真偽値はなく、`0` を偽、それ以外を真とみなします。
- `名前 = 式` で変数に代入します。変数はあらかじめ宣言しません。
- 算術演算子 `+ - * / %` と比較演算子 `< > <= >= == !=` が使えます。
  比較の結果は `1`（真）または `0`（偽）です。
- `print(式)` で値を出力します。
- `if 条件 { ... } else { ... }` で分岐します（`else` は省略可）。
- `while 条件 { ... }` で繰り返します。
- `def 名前(引数, ...) { ... }` で関数を定義し、`return 式` で値を返します。
- 1 つの文は 1 行に書きます（この「1 文 = 1 行」がデバッガで重要になります）。

整数しかない、というのは思い切った単純化です。けれども、ブレークポイントやステップ実行、
変数の検査といったデバッガの本質を学ぶには、これで十分すぎるほどです。

## 第 1 段階 字句解析

最初の仕事は、ソース文字列をトークンの列に変える `tokenize` です。
1 文字ずつ前から見ていき、数字が続けば 1 つの整数トークンに、英字が続けば識別子か
キーワードに、記号なら演算子トークンに……とまとめていきます。

ここで大切なのは、**各トークンが「何行目に現れたか」を覚えておく**ことです。
デバッガが「42 行目で止まる」と言えるのは、もとをたどればこの行番号情報のおかげです。
各トークンを `[種類, 値, 行番号]` という 3 要素の配列で表すことにします。

```ruby
KEYWORDS = %w[if else while def return print]

def tokenize(src)
  tokens = []
  line = 1
  i = 0
  while i < src.length
    c = src[i]
    if c == "\n"
      tokens << [:newline, "\n", line]
      line += 1; i += 1
    elsif c == " " || c == "\t" || c == "\r"
      i += 1                                   # 空白は読み飛ばす
    elsif c == "#"
      i += 1 while i < src.length && src[i] != "\n"   # コメントは行末まで無視
    elsif c =~ /[0-9]/
      j = i
      j += 1 while j < src.length && src[j] =~ /[0-9]/
      tokens << [:int, src[i...j].to_i, line]  # 数字の並び → 整数トークン
      i = j
    elsif c =~ /[a-zA-Z_]/
      j = i
      j += 1 while j < src.length && src[j] =~ /[a-zA-Z0-9_]/
      word = src[i...j]
      type = KEYWORDS.include?(word) ? :keyword : :ident
      tokens << [type, word, line]             # 英数字の並び → キーワードか識別子
      i = j
    else
      two = src[i, 2]
      if %w[== != <= >=].include?(two)
        tokens << [:op, two, line]; i += 2     # 2 文字の演算子を先に判定
      elsif "+-*/%<>=".include?(c)
        tokens << [:op, c, line]; i += 1
      elsif "(){},".include?(c)
        tokens << [:punct, c, line]; i += 1    # 括弧やカンマなどの区切り記号
      else
        raise "unexpected character: #{c.inspect} at line #{line}"
      end
    end
  end
  tokens << [:eof, nil, line]                  # 終端を表す番兵トークン
  tokens
end
```

ポイントを 3 つ補足します。第一に、改行 `\n` を読むたびに `line` を 1 増やし、
それ以降のトークンには新しい行番号がつきます。第二に、`==` のような 2 文字の演算子を
`=` より先に調べないと、`==` が `=` 2 つに割れてしまいます。長い記号を先に試すのは
字句解析の定石です。第三に、最後に `:eof`（end of file）という終端トークンを足しています。
これは構文解析で「もう入力が終わった」と判定するための目印（番兵）です。

`%w[...]` は文字列の配列を作る Ruby の記法で、`%w[a b c]` は `["a", "b", "c"]` と同じです。
`src[i...j]` は文字列の `i` 文字目から `j` の手前までを取り出します。

## 第 2 段階 構文解析

次は、トークンの列を AST に組み立てる構文解析です。
Toy 言語では [再帰下降構文解析](#index:再帰下降構文解析) (recursive descent parsing) という、
もっとも素直な手法を使います。文法の規則ひとつひとつに対応するメソッドを用意し、
それらが互いを呼び合うことで木を作ります。

### AST をどう表すか

AST のノード（節）は、Ruby の配列で表します。配列の先頭にノードの種類を表す
シンボル（`:int` など）を置き、残りに中身を入れます。たとえば `1 + 2` は次のようになります。

```ruby
[:binop, "+", [:int, 1], [:int, 2]]
#  ↑種類   ↑演算子  ↑左の子      ↑右の子
```

「足し算」のノードの下に「左の子」と「右の子」がぶら下がる。これが木構造です。
式のノードは次の 4 種類です。

- `[:int, 数値]` … 整数リテラル
- `[:var, 名前]` … 変数の参照
- `[:binop, 演算子, 左, 右]` … 二項演算
- `[:call, 関数名, [引数, ...]]` … 関数呼び出し

文（statement）のノードは、`[:assign, ...]`, `[:print, ...]`, `[:if, ...]`,
`[:while, ...]`, `[:def, ...]`, `[:return, ...]`, `[:exprstmt, ...]` の 7 種類です。
そして**すべての文を `[:stmt, 行番号, 中身のノード]` で包みます**。

この `:stmt` という包みが、本書のいちばん大事な仕掛けです。
文と文の切れ目に行番号の札を貼っておくのです。
デバッガはこの「文の切れ目」でだけ実行を止めることにします。
つまり `:stmt` のひとつひとつが、デバッガにとっての「止まれる場所」になります。
この設計の意味は、第 6 章で実装するときにはっきりします。

### パーサ本体

構文解析器を `Parser` クラスとして書きます。`@pos` が「いま何番目のトークンを見ているか」、
`peek` が現在のトークンを覗く、`advance` が 1 つ進む、`expect` が「このトークンのはず」と
確認しながら進む、という補助メソッドを用意します。

```ruby
class Parser
  def initialize(tokens)
    @tokens = tokens
    @pos = 0
  end

  def peek; @tokens[@pos]; end
  def advance; t = @tokens[@pos]; @pos += 1; t; end

  def at?(type, value = nil)                 # 現在のトークンが期待どおりか
    t = peek
    t[0] == type && (value.nil? || t[1] == value)
  end

  def expect(type, value = nil)              # 期待どおりなら進む、違えばエラー
    raise "expected #{value || type}, got #{peek.inspect}" unless at?(type, value)
    advance
  end

  def skip_newlines
    advance while at?(:newline)
  end
```

`expect` は、文法に合わないソースを早い段階で見つけるための番人です。
たとえば `if` のあとに条件式が来るはずの場所に別のものが来たら、
そこで「expected ...」とエラーになります。

プログラム全体は「文の並び」です。文の並びは、ある終端（プログラム末尾の `:eof`、
あるいはブロックを閉じる `}`）に出会うまで、文を読み続けることで解析できます。

```ruby
  def parse_program
    body = parse_statements_until(:eof)
    expect(:eof)
    body
  end

  def parse_statements_until(end_type, end_value = nil)
    stmts = []
    skip_newlines
    until at?(end_type, end_value)
      stmts << parse_statement
      skip_newlines
    end
    stmts
  end
```

1 つの文を読む `parse_statement` が、構文解析の中心です。
先頭のトークンを見て、どの種類の文かを判断します。
そして最後に、読み取った中身を `[:stmt, line, node]` で包みます。
この `line`（文の先頭トークンの行番号）こそ、デバッガが使う行番号です。

```ruby
  def parse_statement
    line = peek[2]                           # この文が始まる行番号を覚える
    node =
      if at?(:keyword, "print")
        advance
        [:print, parse_expr]
      elsif at?(:keyword, "if")
        parse_if
      elsif at?(:keyword, "while")
        parse_while
      elsif at?(:keyword, "def")
        parse_def
      elsif at?(:keyword, "return")
        advance
        [:return, parse_expr]
      elsif at?(:ident) && @tokens[@pos + 1][0] == :op && @tokens[@pos + 1][1] == "="
        name = advance[1]
        expect(:op, "=")
        [:assign, name, parse_expr]          # 「識別子 =」で始まれば代入文
      else
        [:exprstmt, parse_expr]              # それ以外は式文（関数呼び出しなど）
      end
    [:stmt, line, node]                      # ★ 行番号の札を貼って包む
  end
```

代入文かどうかは「識別子のあとに `=` が続くか」を 1 つ先のトークンまで覗いて判断します
（これを「先読み」と呼びます）。`if` / `while` / `def` の中身は、それぞれ専用のメソッドに任せます。
これらは「キーワード → 条件式 → `{` → 文の並び → `}`」という似た形をしています。

```ruby
  def parse_if
    expect(:keyword, "if")
    cond = parse_expr
    expect(:punct, "{")
    then_body = parse_statements_until(:punct, "}")
    expect(:punct, "}")
    else_body = []
    if at?(:keyword, "else")
      advance
      expect(:punct, "{")
      else_body = parse_statements_until(:punct, "}")
      expect(:punct, "}")
    end
    [:if, cond, then_body, else_body]
  end

  def parse_while
    expect(:keyword, "while")
    cond = parse_expr
    expect(:punct, "{")
    body = parse_statements_until(:punct, "}")
    expect(:punct, "}")
    [:while, cond, body]
  end

  def parse_def
    expect(:keyword, "def")
    name = expect(:ident)[1]
    expect(:punct, "(")
    params = []
    unless at?(:punct, ")")
      params << expect(:ident)[1]
      while at?(:punct, ",")
        advance
        params << expect(:ident)[1]
      end
    end
    expect(:punct, ")")
    expect(:punct, "{")
    body = parse_statements_until(:punct, "}")
    expect(:punct, "}")
    [:def, name, params, body]
  end
```

`then_body` や `body` が「文の並び」、つまり `:stmt` で包まれたノードの配列であることに
注目してください。`if` の中の文も `while` の中の文も、ちゃんと行番号の札を持っています。
だからデバッガは、ブロックの内側の行でも止まれます。

### 式の解析と演算子の優先順位

式の解析では、演算子の [優先順位](#index:優先順位) (precedence) を正しく扱う必要があります。
`1 + 2 * 3` は `1 + (2 * 3)` であって `(1 + 2) * 3` ではありません。
掛け算は足し算より「強く結びつく」からです。

これを実現する定番のテクニックが、**優先順位ごとにメソッドを分け、
弱い演算子のメソッドから強い演算子のメソッドを呼ぶ**やり方です。
比較（いちばん弱い）→ 加減 → 乗除（いちばん強い）→ 基本要素、の順に呼び出します。
強い演算子ほど木の「深い＝先に計算される」位置に来るので、優先順位が自然に表現されます。

```ruby
  def parse_expr; parse_comparison; end

  def parse_comparison
    left = parse_add
    while at?(:op) && %w[< > <= >= == !=].include?(peek[1])
      op = advance[1]
      left = [:binop, op, left, parse_add]
    end
    left
  end

  def parse_add
    left = parse_mul
    while at?(:op) && %w[+ -].include?(peek[1])
      op = advance[1]
      left = [:binop, op, left, parse_mul]
    end
    left
  end

  def parse_mul
    left = parse_primary
    while at?(:op) && %w[* / %].include?(peek[1])
      op = advance[1]
      left = [:binop, op, left, parse_primary]
    end
    left
  end
```

いちばん下の `parse_primary` が、これ以上分解できない基本要素（整数、括弧でくくった式、
変数、関数呼び出し）を扱います。識別子のあとに `(` が続けば関数呼び出し、
そうでなければ変数の参照です。

```ruby
  def parse_primary
    t = peek
    if at?(:int)
      advance; [:int, t[1]]
    elsif at?(:punct, "(")
      advance
      e = parse_expr
      expect(:punct, ")")
      e
    elsif at?(:ident)
      name = advance[1]
      if at?(:punct, "(")                    # 識別子の次が ( なら関数呼び出し
        advance
        args = []
        unless at?(:punct, ")")
          args << parse_expr
          while at?(:punct, ",")
            advance
            args << parse_expr
          end
        end
        expect(:punct, ")")
        [:call, name, args]
      else
        [:var, name]                         # ただの変数参照
      end
    else
      raise "unexpected token: #{t.inspect}"
    end
  end
end

def parse(tokens)
  Parser.new(tokens).parse_program
end
```

これで `tokenize` と `parse` がそろい、ソース文字列から AST が得られるようになりました。

## 第 3 段階 評価

最後は、AST を実際に実行する評価器です。ここを `Interpreter` クラスとして書きます。
このクラスの形が、第 6 章でデバッガを生やす土台になるので、設計を少し丁寧に見ます。

### 環境とコールスタック

評価には 2 つの状態が必要です。ひとつは [環境](#index:環境) (environment)、
すなわち「変数名 → 値」の対応表です。Ruby のハッシュ（連想配列）で表します。
もうひとつは [コールスタック](#index:コールスタック) です。
関数を呼ぶたびに新しい環境が必要になり、呼び出しが入れ子になるので、
環境を積み上げるスタック（積み重ね）が要ります。

スタックに積む 1 段分を [スタックフレーム](#index:スタックフレーム) (stack frame)、
略してフレームと呼びます。1 つのフレームは「どの関数のものか (`name`)」
「その関数のローカル変数 (`env`)」「いま何行目を実行中か (`line`)」を持ちます。
この `line` を覚えておくことが、第 7 章のバックトレースで効いてきます。

```ruby
Frame = Struct.new(:name, :env, :line)
```

`Struct.new` は、指定した名前の属性を持つ簡単なクラスを作る Ruby の機能です。

### Interpreter クラス

評価器の本体です。初期化のとき、ソースを解析して AST を作り、
さらに `collect_defs` でプログラム中の関数定義をすべて集めて表 `@funcs` に登録します
（関数は、定義より前の行から呼ばれても動くように、先にまとめて拾っておきます）。

```ruby
class Interpreter
  def initialize(source)
    @program = parse(tokenize(source))
    @funcs = {}
    @frames = []
    collect_defs(@program)
  end

  def collect_defs(body)
    body.each do |stmt|
      _tag, _line, node = stmt
      if node[0] == :def
        _, name, params, fbody = node
        @funcs[name] = [params, fbody]
      end
    end
  end

  def run
    @frames.push(Frame.new("main", {}, 0))   # 最初のフレーム（main）を積む
    exec_body(@program)
    @frames.pop
  end
```

`exec_body` は文の並びを順に実行します。そして `exec_stmt` が、文を 1 つ実行する関数です。
ここに、本書でいちばん重要な 3 行が含まれます。

```ruby
  def exec_body(body)
    result = nil
    body.each { |stmt| result = exec_stmt(stmt) }
    result
  end

  def exec_stmt(stmt)
    _tag, line, node = stmt
    @frames.last.line = line                 # 現在行を更新
    on_line(line, stmt)                       # ★ デバッガ用のフック（今は何もしない）
    eval_node(node)
  end

  def on_line(line, stmt); end                # 既定では空っぽ
```

`exec_stmt` は、文を実行する直前に必ず 2 つのことをします。
ひとつは現在のフレームの `line` を更新すること。
もうひとつが `on_line` という**フック（差し込み口）の呼び出し**です。
いまの `on_line` は中身が空で、何もしません。
しかし第 6 章で、ここを上書きするだけでデバッガが完成します。
「文を実行する直前に、必ず一度ここを通る」。この一点が、デバッガのすべての入り口になります。

> [!IMPORTANT]
> なぜ `on_line` をいま、空のまま用意するのでしょうか。
> それは、インタプリタ統合型デバッガの心臓部が「実行ループに差し込んだ 1 つのフック」だと
> 体で理解してほしいからです。デバッガは外付けの魔法ではなく、
> 処理系の実行ループに開けた小さな穴から生えてくるのです。

### ノードを評価する

`eval_node` が、ノードの種類ごとに実際の計算をおこないます。
木をたどりながら、子ノードを再帰的に評価していくところが「ツリーウォーク」です。
たとえば `:binop` は、左の子と右の子をそれぞれ評価してから、演算子を適用します。
この再帰的な評価のスタイルは『計算機プログラムの構造と解釈』(SICP) でも
言語処理系の核として説明されている、由緒ある考え方です[](#cite:abelson1996)。

```ruby
  def eval_node(node)
    case node[0]
    when :int      then node[1]
    when :var      then lookup(node[1])
    when :assign   then @frames.last.env[node[1]] = eval_node(node[2])
    when :binop    then eval_binop(node[1], eval_node(node[2]), eval_node(node[3]))
    when :print    then v = eval_node(node[1]); puts v; v
    when :if
      if truthy?(eval_node(node[1])) then exec_body(node[2]) else exec_body(node[3]) end
    when :while
      exec_body(node[2]) while truthy?(eval_node(node[1]))
      nil
    when :def      then nil                  # 定義は collect_defs で処理済み
    when :return   then throw(:return, eval_node(node[1]))
    when :exprstmt then eval_node(node[1])
    when :call     then call_func(node[1], node[2].map { |a| eval_node(a) })
    else raise "unknown node: #{node.inspect}"
    end
  end
```

変数の参照 `lookup`、真偽判定 `truthy?`、二項演算 `eval_binop` は次のとおりです。
`0` を偽、それ以外を真とみなすので、比較演算子は結果を `1` か `0` で返します。

```ruby
  def lookup(name)
    env = @frames.last.env
    env.fetch(name) { raise "undefined variable: #{name}" }
  end

  def truthy?(v); v != 0; end

  def eval_binop(op, a, b)
    case op
    when "+" then a + b
    when "-" then a - b
    when "*" then a * b
    when "/" then a / b
    when "%" then a % b
    when "<" then a < b ? 1 : 0
    when ">" then a > b ? 1 : 0
    when "<=" then a <= b ? 1 : 0
    when ">=" then a >= b ? 1 : 0
    when "==" then a == b ? 1 : 0
    when "!=" then a != b ? 1 : 0
    end
  end
```

### 関数呼び出しとコールスタックの伸び縮み

関数呼び出し `call_func` は、コールスタックが伸び縮みする様子がそのまま現れる、
見どころの部分です。手順はこうです。引数の値を仮引数名に結びつけた新しい環境を作り、
新しいフレームをスタックに**積み (push)**、関数本体を実行し、終わったらフレームを
**降ろします (pop)**。`return` は Ruby の `throw`/`catch` を使って、
関数本体のどこからでも一気に脱出できるようにしています。

```ruby
  def call_func(name, args)
    params, body = @funcs.fetch(name) { raise "undefined function: #{name}" }
    env = {}
    params.each_with_index { |p, i| env[p] = args[i] }   # 引数を仮引数に束縛
    @frames.push(Frame.new(name, env, 0))                # フレームを積む
    result = catch(:return) { exec_body(body); nil }     # return で脱出できるように
    @frames.pop                                          # フレームを降ろす
    result
  end
end
```

`catch(:return) { ... }` は、ブロックの中で `throw(:return, 値)` が呼ばれると、
その値を結果として即座に脱出する仕組みです。`return` 文が `eval_node` の中で
`throw(:return, ...)` を実行すると、`while` や `if` の何重もの内側にいても、
一気にこの `catch` まで飛んで関数を抜けられます。

`@frames` の push と pop に注目してください。これがコールスタックの実体です。
関数を呼べば 1 段積まれ、関数から戻れば 1 段降りる。
第 7 章のバックトレースは、この `@frames` を上から下へ並べて表示するだけで実現できます。
**デバッガの機能の多くは、処理系がもともと持っているこうした内部状態を、
人間に見せてあげるだけ**なのです。

## 動かしてみる

これで Toy 言語のインタプリタが完成しました。実際に走らせてみましょう。
フィボナッチ数列を計算するプログラムです。

```ruby
src = <<~TOY
  def fib(n) {
    if n < 2 {
      return n
    }
    return fib(n - 1) + fib(n - 2)
  }

  i = 0
  while i < 8 {
    print(fib(i))
    i = i + 1
  }
TOY

Interpreter.new(src).run
```

実行すると、次のように出力されます。

```text
0
1
1
2
3
5
8
13
```

フィボナッチ数列が正しく表示されました。再帰呼び出し、ループ、条件分岐、
すべてがきちんと動いています。

> [!TIP]
> `<<~TOY ... TOY` は Ruby の「ヒアドキュメント」という記法で、複数行の文字列を
> 書くためのものです。`<<~` を使うと、各行の先頭の余分なインデントが自動的に取り除かれます。

土台が完成しました。次章では、いよいよこの `Interpreter` クラスに
デバッガを生やします。鍵になるのは、さきほど空のまま用意しておいた
`on_line` フックです。ここに数十行のコードを足すだけで、ブレークポイントと
ステップ実行が動き出します。
