---
emoji: "✈"
title: "Juliaの型推論アルゴリズムを実装する"
topics:
  - "Julia"
  - "抽象解釈"
  - "コンパイラ最適化"
  - "型推論"
type: "tech"
published: true
---


[先日書いた記事](https://zenn.dev/aviatesk/articles/juliatypeinference20200927aviatesk)ではJuliaの型推論について、特にその機能の1つである[`isa`](https://docs.julialang.org/en/v1/base/base/#Core.isa)を使って条件分岐した先のブロックにおけるflow-sensitivityについて調べてみました。

この記事ではより基礎的な部分に注目して、Juliaの型推論のアルゴリズム(の一部)を理解し、実装してみようと思います。今回紹介する技術はより一般にプログラムの「抽象解釈」("abstract interpretation", "data-flow analysis")と呼ばれているもので、実際に実装するのも["constant folding"(「定数畳み込み」)](https://www.google.com/search?q=constant+propagation&oq=constant+propagation&aqs=chrome..69i57j0l7.4612j1j4&sourceid=chrome&ie=UTF-8)と呼ばれる一般的なコンパイラ最適化問題です。なのでJuliaに興味がない人にも読んでいただけると嬉しいです。


## アルゴリズム

### data flow problem

["a data-flow problem"(抽象解釈問題)](https://dl.acm.org/doi/10.1145/512950.512973)は以下の4つの要素を用いて定義します:
1. **$P = I_0 ... I_n \in \text{Instr}$**: プログラム
2. **$L = <A, \sqcup, \sqcap>$**: $P$が取りうる抽象値のlattice[^0]
3. **$![.!] : \text{Instr} \rightarrow (A \rightarrow A)$**: instruction $\text{Instr}$がプログラムの抽象状態$A$に対してどのように作用するのかを与えるプログラムの抽象的な意味論
4. **$a_0 \in A$**: プログラムの初期状態

ここで:
- $\text{Instr}$: プログラムを構成する基本的な"instructions"(命令たち)。条件分岐を起こすinstructionと、そうでないものが区別されていればなんでも良い
- $A$: プログラムの抽象状態を表現する集合[^1]
- $\sqcup, \sqcap$: それぞれjoinとmeet[^0]に対応する演算で、$A$に対して作用する

[^0]: lattice: https://en.wikipedia.org/wiki/Lattice_(order) / join and meet: https://en.wikipedia.org/wiki/Join_and_meet

[^1]: 論文では記号として[Unicode Character “𝕮” (U+1D56E)](https://www.compart.com/en/unicode/U+1D56E)が用いられていますが、このフォントがKaTeXでサポートされていないのでこの記事では$A$を用います。

$P, L, ![.!]$の具体的な中身はそれぞれの抽象解釈問題ごとに決まります。

### "A Graph-Free Approach to Data-Flow Analysis"

data-flow problemを解くアルゴリズムは、[Muchnick, Steven S., and Neil D. Jones. "Program flow analysis: Theory and applications." 1981.](https://www.researchgate.net/publication/220689271_Program_Flow_Analysis_Theory_and_Applications)で提案された、[basic block](https://en.wikipedia.org/wiki/Basic_block)をノードとするをグラフ("BB graph")をコアなデータ構造として用いるアルゴリズムが、_de facto_ standardとなっていたようです。

[Mohnen, Markus. “A Graph-Free Approach to Data-Flow Analysis.” _CC_ (2002).](https://www.semanticscholar.org/paper/A-Graph-Free-Approach-to-Data-Flow-Analysis-Mohnen/5ad8cb6b477793ffb5ec29dde89df6b82dbb6dba?p2df)はそのアルゴリズムをグラフを明示的なデータ構造として用いないように拡張したアルゴリズムを提案しました。

データ構造としてのグラフを用いないことで以下のような恩恵が得られることを実験的に示しています:
- メモリ効率の改善: 多くの実験ケースでメモリ使用率を1/3程度に減らしつつ、実行時間のトレードオフは無視できるレベルであった
- 実装が容易: BB graphを構築する必要がない

論文で提案されているアルゴリズムを理解しやすくするために、まずは具体的な問題を考えてみましょう。

### 問題設定: constant folding propagation problem

この記事ではdata-flow problemの一例として、["constant folding"(「定数畳み込み」)](https://www.google.com/search?q=constant+propagation&oq=constant+propagation&aqs=chrome..69i57j0l7.4612j1j4&sourceid=chrome&ie=UTF-8)を考えます。

constant folding propagationは、コンパイル時に分かる定数をできるだけ見つけて実行時の計算と置き換える、というコンパイラ最適化技術です。従ってそれを行うdata-flow problemとしては、プログラム中のそれぞれの時点における各変数が定数であるかどうか決めていく、という設定になります。

それでは早速 1.) プログラム $P$, 2.) lattice $L$, 3.) abstract semantics $![.!]$, 4.) プログラムの初期状態 $a_0$ のそれぞれを確認しましょう。

#### 問題設定 1. プログラム $P$

以下では簡単のため、プログラム$P$は整数に対する演算のみを行うものとし、$\text{Instr}$としては以下の3つのみを考えることにします:
- conditional branching: `condition && goto instruction`
- unconditional branching: `goto instruction`
- assignment: `lhs := rhs`

#### 問題設定 2. lattice $L$

プログラム$P$に含まれる各変数が取りうる抽象値$C$としては以下のものだけを考えます:
- $\top$: latticeの頂に位置する
- $c \in \mathbb{N}$: 定数
- $\bot$: latticeの底に位置する

$\top$と$\bot$の直観的な意味として、論文では以下のように説明されています:
- $\top$: "non constant due to missing information"
- $\bot$: "non constant due to conflict"

$C$の順序関係を次のように定義します: $c_1 \le c_2$ iff a) $c_1 = c_2$, b) $c_1 = \bot$, c) $c_2 = \top$

$C$は次の画像のようなフラットなlatticeとなります:
![lattice](https://raw.githubusercontent.com/aviatesk/aviatesk-zenn-content/master/articles/data-flow-problem-20201025-assets/lattice.png)

プログラム$P$の各時点における抽象状態$A$は、$X$をプログラム$P$中の変数の集合として、次のようなmapとして表現できます:

$$
    A := X \rightarrow C
$$

$A$に対する$\sqcup, \sqcap$演算は、$A$中の変数のそれぞれの抽象値$C$に対しpair-wiseに行うような演算として自然に考えることができます。

#### 問題設定 3. abstract semantics $![.!]$

プログラム$P$の抽象的な意味論$![.!]$を考えましょう:
- conditional branching (`condition && goto instruction`): 抽象状態$A$を変化させない
- unconditional branching (`goto instruction`): 抽象状態$A$を変化させない
- assignment: `lhs := rhs`: `rhs`が定数である時のみその定数値を`lhs`に代入、それ以外の時は$\bot$を代入

#### 問題設定 4. プログラムの初期状態 $a_0$

論文ではプログラムの初期状態$a_0$を$\bot$で初期化するとしていますが、おそらく間違いです。直観的な意味からも、プログラムの初期状態では各変数は"non constant due to missing information"と解釈されるべきなので、$\top$で初期化されるべきでしょう:

$$
    a_0 = X \rightarrow \top
$$

### アルゴリズム

以上1.~4.でdata-flow problemのセットアップができました。

論文ではそのdata-flow problemを解くアルゴリズムとして以下のアルゴリズムが提案されています:
![algorithm](https://raw.githubusercontent.com/aviatesk/aviatesk-zenn-content/master/articles/data-flow-problem-20201025-assets/alrogithm.png)

このアルゴリズムの直観的な理解としては以下のようになります:
1. プログラム自身がinstruction levelで(abstract semantics $![.!]$を通して)その抽象状態へ作用する
2. アルゴリズムは現在のinstruction $I$を示す現在のprogram counter $pc$と、作用が計算されるべき残りのinstruction達(に対応する$pc$)を保持するworking set $W$を更新しながら動作する
2. 現在のinstruction$I$の抽象状態の変化は、$I$が到達する可能性があるinstruction全てに伝播する
3. ただし、現在のinstruction$I$の抽象状態の変化は、**伝播する先の抽象状態を変化させる場合にのみ**伝播する

3.はアルゴリズム中の$I_{pc} = (\text{if} \psi \text{goto} l)$の箇所に相当します。これはつまり、このアルゴリズムは、実際のプログラム実行とは違い、条件分岐の分岐先について両方の分岐を勘定するということです。

4.はアルゴリズム中の$\text{if} new < s_{pc'}$と$\text{if} new < s_{l}$の箇所に相当します。ここで、$new, s_{pc'}, s_l$は抽象状態ですが、論文ではそれらの順序関係$<$は以下の条件と等しいとしています:

$$
    new < s_{pc'} \equiv (new \sqcap s_{pc'} = new) \land (new \ne s_{pc'})
$$

現在のinstruction $I$の抽象状態の変化は、この条件が成り立つときにのみ伝播するので、各instructionの抽象状態$A := X \rightarrow C$は、常に$C$がlattice$L$の下側へ遷移するように更新されます。そのため、lattice $L$の高さが有限であればこのアルゴリズムは必ず収束します。

### example program `prog0` and tracing

プログラム$P$の具体例として、以下のプログラム`prog0`があるとしましょう:

> `prog0`
```
0 ─ I₀ = x := 1
│   I₁ = y := 2
│   I₂ = z := 3
└── I₃ = goto I₈
1 ─ I₄ = r := y + z
└── I₅ = if x ≤ z goto I₇
2 ─ I₆ = r := z + y
3 ─ I₇ = x := x + 1
4 ─ I₈ = if x < 10 goto I₄
```
(最左列の番号はbasic blockに対応する番号です。この記事で紹介するアルゴリズムでは使用されません。)

論文では、`prog0`に対する上記アルゴリズムのtracingの例として以下の表を載せています:
![tracing example](https://raw.githubusercontent.com/aviatesk/aviatesk-zenn-content/master/articles/data-flow-problem-20201025-assets/tracing-example.png)

最終的な状態$s_8$において、プログラム中にない「`r`が定数`5`である」という情報が現れているのが分かると思います。

## 実装

前置きが長くなりましたが、それでは早速Juliaを用いて以上の問題設定とアルゴリズムを表現してみましょう。
前節と同じく、まず問題設定を記述し、次にアルゴリズムを実装します。

### 問題設定: constant folding propagation example

#### 問題設定 1. プログラム $P$

```julia
abstract type Exp end

struct Sym <: Exp
    name::Symbol
end

struct Num <: Exp
    val::Int
end

struct Call <: Exp
    head::Sym
    args::Vector{Exp}
end

abstract type Instr end

struct Assign <: Instr
    lhs::Sym
    rhs::Exp
end

struct Goto <: Instr
    label::Int
end

struct GotoIf <: Instr
    label::Int
    cond::Exp
end

const Program = Vector{Instr}
```

```
Vector{Instr} (alias for Array{Instr, 1})
```





#### 問題設定 2. lattice $L$

プログラムの抽象状態$A := X \rightarrow C$を表現します。

まず、プログラム中の各変数が取りうる抽象値$C$と、その順序関係を定義します:
```julia
# partial order, meet, join, top, bottom, and their identities

import Base: ≤, ==, <, show

abstract type LatticeElement end

struct Const <: LatticeElement
    val::Int
end

struct TopElement <: LatticeElement end
struct BotElement <: LatticeElement end

const ⊤ = TopElement()
const ⊥ = BotElement()

show(io::IO, ::TopElement) = print(io, '⊤')
show(io::IO, ::BotElement) = print(io, '⊥')

≤(x::LatticeElement, y::LatticeElement) = x≡y
≤(::BotElement,      ::TopElement) = true
≤(::BotElement,      ::LatticeElement) = true
≤(::LatticeElement,  ::TopElement) = true

# NOTE: == and < are defined such that future LatticeElements only need to implement ≤
==(x::LatticeElement, y::LatticeElement) = x≤y && y≤x
<(x::LatticeElement, y::LatticeElement) = x≤y && !(y≤x)

# join
⊔(x::LatticeElement, y::LatticeElement) = x≤y ? y : y≤x ? x : ⊤

# meet
⊓(x::LatticeElement, y::LatticeElement) = x≤y ? x : y≤x ? y : ⊥
```

```
⊓ (generic function with 2 methods)
```





次に、抽象状態$A$を各変数$X$から抽象値$C$へのmapとして表現し、その順序関係$<$を定義します:
```julia
# NOTE: the paper (https://api.semanticscholar.org/CorpusID:28519618) uses U+1D56E MATHEMATICAL BOLD FRAKTUR CAPITAL C for this
const AbstractState = Dict{Symbol,LatticeElement}

# extend lattices of values to lattices of mappings of variables to values;
# ⊓ and ⊔ operate pair-wise, and from there we can just rely on the Base implementation for
# dictionary equiality comparison

⊔(X::AbstractState, Y::AbstractState) = AbstractState( v => X[v] ⊔ Y[v] for v in keys(X) )
⊓(X::AbstractState, Y::AbstractState) = AbstractState( v => X[v] ⊓ Y[v] for v in keys(X) )

<(X::AbstractState, Y::AbstractState) = X⊓Y==X && X≠Y
```

```
< (generic function with 91 methods)
```





#### 問題設定 3. abstract semantics $![.!]$

abstract semantics $![.!]$は、実際のJuliaのexecutionを用いて簡単に実装できます。
`Call`の`head::Symbol` fieldから実際の演算メソッドを`getfield`を用いて復元します。
```julia
abstract_eval(x::Num, s::AbstractState) = Const(x.val)

abstract_eval(x::Sym, s::AbstractState) = get(s, x.name, ⊥)

function abstract_eval(x::Call, s::AbstractState)
    f = getfield(@__MODULE__, x.head.name)

    to_val(x::Num)   = x.val
    to_val(x::Const) = x.val
    argvals = Int[]
    for arg in x.args
        arg = abstract_eval(arg, s)
        arg === ⊥ && return ⊥
        push!(argvals, to_val(arg))
    end

    return Const(f(argvals...))
end
```

```
abstract_eval (generic function with 3 methods)
```





#### 問題設定 4. プログラムの初期状態 $a_0$

論文の例を修正し、$\top$で初期化します:
```julia
a₀ = AbstractState(:x => ⊤, :y => ⊤, :z => ⊤, :r => ⊤)
```

```
Dict{Symbol, LatticeElement} with 4 entries:
  :y => ⊤
  :z => ⊤
  :r => ⊤
  :x => ⊤
```





### example program `prog0`

アルゴリズムを実装する前に、プログラム例`prog0`を表現しましょう。

ナイーブに表現するとこのようになると思います:
```julia
prog0 = [Assign(Sym(:x), Num(1)),                              # I₀
         Assign(Sym(:y), Num(2)),                              # I₁
         Assign(Sym(:z), Num(3)),                              # I₂
         Goto(8),                                              # I₃
         Assign(Sym(:r), Call(Sym(:(+)), [Sym(:y), Sym(:z)])), # I₄
         GotoIf(7, Call(Sym(:(≤)), [Sym(:x), Sym(:z)])),       # I₅
         Assign(Sym(:r), Call(Sym(:(+)), [Sym(:z), Sym(:y)])), # I₆
         Assign(Sym(:x), Call(Sym(:(+)), [Sym(:x), Num(1)])),  # I₇
         GotoIf(4, Call(Sym(:(<)), [Sym(:x), Num(10)])),       # I₈
         ]::Program
```

```
9-element Vector{Instr}:
 Assign(Sym(:x), Num(1))
 Assign(Sym(:y), Num(2))
 Assign(Sym(:z), Num(3))
 Goto(8)
 Assign(Sym(:r), Call(Sym(:+), Exp[Sym(:y), Sym(:z)]))
 GotoIf(7, Call(Sym(:≤), Exp[Sym(:x), Sym(:z)]))
 Assign(Sym(:r), Call(Sym(:+), Exp[Sym(:z), Sym(:y)]))
 Assign(Sym(:x), Call(Sym(:+), Exp[Sym(:x), Num(1)]))
 GotoIf(4, Call(Sym(:<), Exp[Sym(:x), Num(10)]))
```





ちょっと醜いですね。
ここでは、Juliaの強力なメタプログラミング機能を用いて、Juliaのsyntaxから今回ターゲットとするinstruction levelのプログラム$P$を生成するマクロ`@prog`を記述しましょう[^2]。
[surface syntax AST](https://docs.julialang.org/en/v1/devdocs/ast/#Surface-syntax-AST)に対するpattern-matchingを行えるパッケージ[MacroTools.jl](https://github.com/FluxML/MacroTools.jl)を使います:
```julia
using MacroTools

macro prog(blk)
    Instr[Instr(x) for x in filter(!islnn, blk.args)]::Program
end

function Instr(x)
    if @capture(x, lhs_ = rhs_)               # => Assign
        Assign(Instr(lhs), Instr(rhs))
    elseif @capture(x, @goto label_)          # => Goto
        Goto(label)
    elseif @capture(x, cond_ && @goto label_) # => GotoIf
        GotoIf(label, Instr(cond))
    elseif @capture(x, f_(args__))            # => Call
        Call(Instr(f), Instr.(args))
    elseif isa(x, Symbol)                     # => Sym
        Sym(x)
    elseif isa(x, Int)                        # => Num
        Num(x)
    else
        error("invalid expression: $(x)")
    end
end

islnn(@nospecialize(_)) = false
islnn(::LineNumberNode) = true
```

```
islnn (generic function with 2 methods)
```





[^2]: 他には、[`Base.convert`](https://docs.julialang.org/en/v1/manual/conversion-and-promotion/)をoverloadして`Symbol`から`Sym`などへ自動的にpromoteされるようにすることで、`Sym`や`Num`などのconstructorの記述をせずにより簡潔に記述できるようにする、という手もあります。書くプログラムの見た目的にはナイーブな表現と近いものになります。

`prog0`を生成してみます:
```julia
prog0 = @prog begin
    x = 1             # I₀
    y = 2             # I₁
    z = 3             # I₂
    @goto 8           # I₃
    r = y + z         # I₄
    x ≤ z && @goto 7  # I₅
    r = z + y         # I₆
    x = x + 1         # I₇
    x < 10 && @goto 4 # I₀
end
```

```
9-element Vector{Instr}:
 Assign(Sym(:x), Num(1))
 Assign(Sym(:y), Num(2))
 Assign(Sym(:z), Num(3))
 Goto(8)
 Assign(Sym(:r), Call(Sym(:+), Exp[Sym(:y), Sym(:z)]))
 GotoIf(7, Call(Sym(:≤), Exp[Sym(:x), Sym(:z)]))
 Assign(Sym(:r), Call(Sym(:+), Exp[Sym(:z), Sym(:y)]))
 Assign(Sym(:x), Call(Sym(:+), Exp[Sym(:x), Num(1)]))
 GotoIf(4, Call(Sym(:<), Exp[Sym(:x), Num(10)]))
```





うまく$P$が作れてますね :-)

### アルゴリズム

それでは本題のアルゴリズムを実装してみましょう。論文で紹介されているままに実装すると次のようになります:
```julia
function max_fixed_point(prog::Program, a₀::AbstractState, eval)
    n = length(prog)
    init = AbstractState( v => ⊤ for v in keys(a₀) )
    s = [ a₀; [ init for i = 2:n ] ]
    W = BitSet(0:n-1)

    while !isempty(W)
        pc = first(W)
        while pc ≠ n
            delete!(W, pc)
            I = prog[pc+1]
            new = s[pc+1]
            if isa(I, Assign)
                # for an assignment, outgoing value is different from incoming
                new = copy(new)
                new[I.lhs.name] = eval(I.rhs, new)
            end

            if isa(I, Goto)
                pc´ = I.label
            else
                pc´ = pc+1
                if isa(I, GotoIf)
                    l = I.label
                    if new < s[l+1]
                        push!(W, l)
                        s[l+1] = new
                    end
                end
            end
            if pc´≤n-1 && new < s[pc´+1]
                s[pc´+1] = new
                pc = pc´
            else
                pc = n
            end
        end
    end

    return s
end
```

```
max_fixed_point (generic function with 1 method)
```




論文では`0`-indexでプログラムなどを記述しているので、この記事でもそのまま表現していますが、Juliaの配列のindexingは`1`から始まるので、`s[pc´+1]`などとしてindexingを調整していることに注意してください。

それでは実行してみましょう！
```julia
max_fixed_point(prog0, a₀, abstract_eval)
```

```
9-element Vector{Dict{Symbol, LatticeElement}}:
 Dict(:y => ⊤, :z => ⊤, :r => ⊤, :x => ⊤)
 Dict(:y => ⊤, :z => ⊤, :r => ⊤, :x => Const(1))
 Dict(:y => Const(2), :z => ⊤, :r => ⊤, :x => Const(1))
 Dict(:y => Const(2), :z => Const(3), :r => ⊤, :x => Const(1))
 Dict(:y => Const(2), :z => Const(3), :r => ⊤, :x => Const(1))
 Dict(:y => Const(2), :z => Const(3), :r => Const(5), :x => Const(1))
 Dict(:y => Const(2), :z => Const(3), :r => Const(5), :x => Const(1))
 Dict(:y => Const(2), :z => Const(3), :r => Const(5), :x => Const(1))
 Dict(:y => Const(2), :z => Const(3), :r => ⊤, :x => Const(1))
```





... 論文に載っているtrace例と違う出力が出ています。

先に結論をいうと、実はこれは論文のアルゴリズムが一部間違っているからなのですが、詳しくみてみましょう[^3]。

[^3]: ちなみにerrataなどは出ていません。

まず、先ほど紹介したtrace例は実は不完全であり、実際にアルゴリズムを動かすと、表で空欄になっている左から11行目上から5行目で$s_3$の状態が$s_8$にも伝播しているはずで、$s_8 := \text{x}/1 \text{y}/2 \text{z}/3 \text{r}/\top$となっています。
そうすると、左から11行目上から10行目で$s_8 < s_7$が成り立たず$s_7$の状態が$s_8$へ伝播せず、そのまま$W$が空となりアルゴリズムは停止します(i.e. この記事の実装でいうと`new < s[pc´+1]`が`false`となります)。

上の実装は論文に忠実に従っているので、以上で説明した挙動の通りにアルゴリズムが停止してしまいます。さぁ困った。

### 論文のアルゴリズムをデバッグ

論文のアルゴリズムの問題点は、端的に言えば、「伝播する先の抽象状態を変化させるかどうか」という判定に抽象状態同士の順序関係を用いるとうまく状態が伝播しない、ということです。
つまりこの場合、$s_8$の前状態`s[pc´+1]`($\text{x}/1 \text{y}/2 \text{z}/3 \text{r}/\top$)と現状態`new = s[pc´]`($\text{x}/2 \text{y}/2 \text{z}/3 \text{r}/5$)の間には`new < s[pc´+1]`という順序関係は成立しませんが[^4]、`new`の状態を`s[pc´+1]`へ伝播させ、`s[pc´+1]`を新たな状態($\text{x}/\bot \text{y}/2 \text{z}/3 \text{r}/5$)へと更新したいわけです。

[^4]: ないしは`s[pc´+1] < new`という順序関係も成立しません。

従ってアルゴリズムの修正の方向性としては、以下のようになります:
1. 現状態$new$と前状態$s_{pc'}$の順序関係 $new < s_{pc'} \equiv (new \sqcap s_{pc'} = new) \land (new \ne s_{pc'})$を用いず、$new$の変化を$s_{pc'}$へ伝えたい
2. 一方で、アルゴリズムの停止性を保つために、$new$の変更は、**新しい状態が前状態よりもlattice $L$において低い位置にいくように**、伝播される必要がある

これらをコードに落とし込むと次のようになります:
1. 「伝播する先の抽象状態を変化させるかどうか」の判定に、状態同士のequivalenceをそのまま用いる
2. 状態の更新に、`⊓`(meet: 最大下界を計算)を用いて、更新後の状態が更新前の状態よりも常に$L$において低い位置にいくようにする

というわけで`max_fixed_point`に以下のdiffをあてましょう:
```diff
diff --git a/articles/dataflow.jl b/articles/dataflow.jl
index c43e086..c41ffa7 100644
--- a/articles/dataflow.jl
+++ b/articles/dataflow.jl
@@ -156,14 +156,14 @@ function max_fixed_point(prog::Program, a₀::AbstractState, eval)
                 pc´ = pc+1
                 if isa(I, GotoIf)
                     l = I.label
-                    if new < s[l+1]
+                    if new ≠ s[l+1]
                         push!(W, l)
-                        s[l+1] = new
+                        s[l+1] = new ⊓ s[l+1]
                     end
                 end
             end
-            if pc´≤n-1 && new < s[pc´+1]
-                s[pc´+1] = new
+            if pc´≤n-1 && new ≠ s[pc´+1]
+                s[pc´+1] = new ⊓ s[pc´+1]
                 pc = pc´
             else
                 pc = n
```

### 修正版アルゴリズム

それでは修正版のアルゴリズムで試してみましょう:
```julia
# NOTE: in this problem, we make sure that states will always move to _lower_ position in lattice, so
# - initialize states with `⊤`
# - we use `⊓` (meet) operator to update states,
# - and the condition we use to check whether or not the statement makes a change is `new ≠ prev`
function max_fixed_point(prog::Program, a₀::AbstractState, eval)
    n = length(prog)
    init = AbstractState( v => ⊤ for v in keys(a₀) )
    s = [ a₀; [ init for i = 2:n ] ]
    W = BitSet(0:n-1)

    while !isempty(W)
        pc = first(W)
        while pc ≠ n
            delete!(W, pc)
            I = prog[pc+1]
            new = s[pc+1]
            if isa(I, Assign)
                # for an assignment, outgoing value is different from incoming
                new = copy(new)
                new[I.lhs.name] = eval(I.rhs, new)
            end

            if isa(I, Goto)
                pc´ = I.label
            else
                pc´ = pc+1
                if isa(I, GotoIf)
                    l = I.label
                    if new ≠ s[l+1]
                        push!(W, l)
                        s[l+1] = new ⊓ s[l+1]
                    end
                end
            end
            if pc´≤n-1 && new ≠ s[pc´+1]
                s[pc´+1] = new ⊓ s[pc´+1]
                pc = pc´
            else
                pc = n
            end
        end
    end

    return s
end

max_fixed_point(prog0, a₀, abstract_eval) # The solution contains the `:r => Const(5)`, which is not found in the program
```

```
9-element Vector{Dict{Symbol, LatticeElement}}:
 Dict(:y => ⊤, :z => ⊤, :r => ⊤, :x => ⊤)
 Dict(:y => ⊤, :z => ⊤, :r => ⊤, :x => Const(1))
 Dict(:y => Const(2), :z => ⊤, :r => ⊤, :x => Const(1))
 Dict(:y => Const(2), :z => Const(3), :r => ⊤, :x => Const(1))
 Dict(:y => Const(2), :z => Const(3), :r => Const(5), :x => ⊥)
 Dict(:y => Const(2), :z => Const(3), :r => Const(5), :x => ⊥)
 Dict(:y => Const(2), :z => Const(3), :r => Const(5), :x => ⊥)
 Dict(:y => Const(2), :z => Const(3), :r => Const(5), :x => ⊥)
 Dict(:y => Const(2), :z => Const(3), :r => Const(5), :x => ⊥)
```





Hooray ! 見事、$I_8$の状態$s_8$に該当する`Dict(:y => Const(2), :z => Const(3), :r => Const(5), :x => ⊥)`で、「`r`が定数`Const(5)`である」という情報が見つけられていますね。


## Julia自身の型推論と比較してみる

ところで、Juliaの型推論は大きく以下の2つのパートから成り立っています:
- part 1. 推論フレームのスコープ内でローカルに行う推論
- part 2. 関数呼び出しを跨いで行う推論

part 1.がJuliaの型推論プロセスのコアとなるルーチンで、“A Graph-Free Approach to Data-Flow Analysis.”で提唱されたアルゴリズムをベースとしています。part 2.でそのアルゴリズムを相互再帰呼び出しも扱えるように拡張しており、そちらの詳細についてはJuliaのcreatorの1人であるJeff Bezansonさんの[Ph.D thesis](https://www.google.com/search?sxsrf=ALeKk01fOq5mwVrfsuot9PCCOyRizbIqQg%3A1602874648121&ei=GO2JX4z7BqrfmAXU35SQCA&q=jeff+bezanson+phd+thesis&oq=jeff+bezanson+phd+thesis&gs_lcp=CgZwc3ktYWIQAzoECCMQJzoFCC4QyQM6BggAEBYQHjoFCCEQoAE6BAghEBU6CAghEBYQHRAeULhEWLhqYP5raAJwAHgAgAG7AYgB-hKSAQQwLjE2mAEAoAEBqgEHZ3dzLXdpesABAQ&sclient=psy-ab&ved=0ahUKEwiM55Sw5bnsAhWqL6YKHdQvBYIQ4dUDCA0&uact=5)が詳しいですが、この記事では触れません。

となると、Juliaの型推論ルーチンがこの記事と同様に元論文のアルゴリズムを修正しているのかどうか、`prog0`に対して正しく動くのかどうか気になるところです。

### まず試してみる

まずは`prog0`に対応するJulia programを作って、型推論を走らせてみましょう。Juliaの型推論はdata-flow analysisによりJuliaプログラムの型付けを行うことを目的としていますが、推論の精度を高めるために定数伝播も行います。もしJuliaの型推論ルーチンが正しく動くならば推論の結果に「`r`が定数`5`である」という情報が現れているはずです。

`prog0`をJuliaプログラムとしてそのまま表現すると以下のようになると思います:
```julia
begin
    begin
        @label I₀
        x = 1
    end
    ...
    begin
        @label I₅
        x ≤ z && @goto I₇
    end
    ...
end
```



やはり少し醜いので、この記事の$P = I_0 ... I_n \in \text{Instr}$の記述方法から実際に実行可能なJuliaコードを生成するマクロ`@prog′`を定義しましょう:
```julia
# generate valid Julia code from the "`Instr` syntax"
macro prog′(blk)
    prog′ = Expr(:block)
    bns = [gensym(Symbol(:label, i-1)) for i in 1:length(blk.args)] # generate labels for every statement

    for (i,x) in enumerate(filter(!islnn, blk.args))
        x = MacroTools.postwalk(x) do x
            return if @capture(x, @goto label_)
                Expr(:symbolicgoto, bns[label+1]) # fix `@goto i` into valid goto syntax
            else
                x
            end
        end

        push!(prog′.args, Expr(:block, Expr(:symboliclabel, bns[i]), x)) # label this statement
    end

    return prog′
end

@macroexpand @prog′ begin
    x = 1
    y = 2
    z = 3
    @goto 8
    r = y + z
    x ≤ z && @goto 7
    r = z + y
    x = x + 1
    x < 10 && @goto 4
end
```

```
quote
    begin
        $(Expr(:symboliclabel, Symbol("#3776###label0#3208")))
        var"#3788#x" = 1
    end
    begin
        $(Expr(:symboliclabel, Symbol("#3777###label1#3209")))
        var"#3785#y" = 2
    end
    begin
        $(Expr(:symboliclabel, Symbol("#3778###label2#3210")))
        var"#3786#z" = 3
    end
    begin
        $(Expr(:symboliclabel, Symbol("#3779###label3#3211")))
        $(Expr(:symbolicgoto, Symbol("#3780###label8#3216")))
    end
    begin
        $(Expr(:symboliclabel, Symbol("#3781###label4#3212")))
        var"#3787#r" = var"#3785#y" + var"#3786#z"
    end
    begin
        $(Expr(:symboliclabel, Symbol("#3782###label5#3213")))
        var"#3788#x" ≤ var"#3786#z" && $(Expr(:symbolicgoto, Symbol("#3783#
##label7#3215")))
    end
    begin
        $(Expr(:symboliclabel, Symbol("#3784###label6#3214")))
        var"#3787#r" = var"#3786#z" + var"#3785#y"
    end
    begin
        $(Expr(:symboliclabel, Symbol("#3783###label7#3215")))
        var"#3788#x" = var"#3788#x" + 1
    end
    begin
        $(Expr(:symboliclabel, Symbol("#3780###label8#3216")))
        var"#3788#x" < 10 && $(Expr(:symbolicgoto, Symbol("#3781###label4#3
212")))
    end
end
```




うまく生成できていますね。

それでは早速[`code_typed`](https://docs.julialang.org/en/v1/base/base/#Base.code_typed)を用いて型推論の結果を確認してみましょう:
```julia
code_typed(; optimize = false) do
    @prog′ begin
        x = 1
        y = 2
        z = 3
        @goto 8
        r = y + z
        x ≤ z && @goto 7
        r = z + y
        x = x + 1
        x < 10 && @goto 4

        x, y, z, r # to check the result of abstract interpretation
    end
end |> first
```

```
CodeInfo(
1 ─       Core.NewvarNode(:(r))::Any
│         (x = 1)::Core.Const(1)
│         (y = 2)::Core.Const(2)
│         (z = 3)::Core.Const(3)
└──       goto #6
2 ─       (r = y::Core.Const(2) + z::Core.Const(3))::Core.Const(5)
│   %7  = (x ≤ z::Core.Const(3))::Bool
└──       goto #4 if not %7
3 ─       goto #5
4 ─       (r = z::Core.Const(3) + y::Core.Const(2))::Core.Const(5)
5 ┄       (x = x + 1)::Int64
6 ┄ %12 = (x < 10)::Bool
└──       goto #8 if not %12
7 ─       goto #2
8 ─ %15 = Core.tuple(x, y::Core.Const(2), z::Core.Const(3), r::Core.Const(5
))::Core.PartialStruct(NTuple{4, Int64}, Any[Int64, Core.Const(2), Core.Con
st(3), Core.Const(5)])
└──       return %15
) => NTuple{4, Int64}
```





`8 ─ %15 = Core.tuple(x, y::Core.Const(2), z::Core.Const(3), r::Core.Const(5))::Core.PartialStruct(NTuple{4, Int64}, Any[Int64, Core.Const(2), Core.Const(3), Core.Const(5)])`という出力に現れているように、Juliaの型推論ルーチンは正しく p
`r`が定数(Julia compilerの表現としては`Core.Const(5)`)である」という情報を見つけることができていますね。

:::details 実行してみる
実行してみると、やはり期待通り`r == 5`という結果が得られているのが確認できます:
```julia
@prog′ begin
    x = 1
    y = 2
    z = 3
    @goto 8
    r = y + z
    x ≤ z && @goto 7
    r = z + y
    x = x + 1
    x < 10 && @goto 4

    x, y, z, r # to check the result of actual execution
end
```

```
(10, 2, 3, 5)
```




:::

### Juliaの型推論の実装での修正を確認する

どうやらJuliaの型推論ルーチンも論文の間違いを修正しているようです。実際のコードを見て確認してみましょう。

先ほどの修正点に対応する箇所を紹介します:
> これらをコードに落とし込むと次のようになります:
> 1. 「伝播する先の抽象状態を変化させるかどうか」の判定に、状態同士のequivalenceをそのまま用いる
> 2. 状態の更新に、`⊓`(meet: 最大下界を計算)を用いて、更新後の状態が更新前の状態よりも常に$L$の低い位置にいくようにする

:::message
今回紹介するコードはこの[commit](https://github.com/JuliaLang/julia/tree/d474c98667db0bf4832e4eeb7beb0e8cfc8b7481)のJuliaLang/juliaレポジトリから引用しています。
:::

#### 修正1. 「伝播する先の抽象状態を変化させるかどうか」の判定

Juliaの型推論実装において、抽象状態の更新に該当する箇所は次のコードになります:

1. https://github.com/JuliaLang/julia/blob/d474c98667db0bf4832e4eeb7beb0e8cfc8b7481/base/compiler/abstractinterpretation.jl#L1316: `newstate_else = stupdate!(s[l], changes_else)`
2. https://github.com/JuliaLang/julia/blob/d474c98667db0bf4832e4eeb7beb0e8cfc8b7481/base/compiler/abstractinterpretation.jl#L1415: `newstate = stupdate!(s[pc´], changes)`

`supdate!`は以下のような実装になっています：

> https://github.com/JuliaLang/julia/blob/d474c98667db0bf4832e4eeb7beb0e8cfc8b7481/base/compiler/typelattice.jl#L268-L307
```julia
function stupdate!(state::VarTable, changes::StateUpdate)
    if !isa(changes.var, Slot)
        return stupdate!(state, changes.state)
    end
    newstate = false
    changeid = slot_id(changes.var::Slot)
    for i = 1:length(state)
        if i == changeid
            newtype = changes.vtype
        else
            newtype = changes.state[i]
        end
        oldtype = state[i]
        # remove any Conditional for this Slot from the vtable
        if isa(newtype, VarState)
            newtypetyp = newtype.typ
            if isa(newtypetyp, Conditional) && slot_id(newtypetyp.var) == changeid
                newtype = VarState(widenconditional(newtypetyp), newtype.undef)
            end
        end
        if schanged(newtype, oldtype)
            newstate = state
            state[i] = smerge(oldtype, newtype)
        end
    end
    return newstate
end

function stupdate!(state::VarTable, changes::VarTable)
    newstate = false
    for i = 1:length(state)
        newtype = changes[i]
        oldtype = state[i]
        if schanged(newtype, oldtype)
            newstate = state
            state[i] = smerge(oldtype, newtype)
        end
    end
    return newstate
end
```



やや複雑ですが、どうやら`schanged`が状態の更新をするかどうかの判定を行っているようです。`schanged`はこのように実装されています:

> https://github.com/JuliaLang/julia/blob/d474c98667db0bf4832e4eeb7beb0e8cfc8b7481/base/compiler/typelattice.jl#L236
```julia
@inline schanged(@nospecialize(n), @nospecialize(o)) = (n !== o) && (o === NOT_FOUND || (n !== NOT_FOUND && !issubstate(n, o)))
```



この記事と違って抽象状態$A$を比較するのではなく、各変数の抽象値$C$(この記事の`LatticeElement`に相当)を比較していますが、やはり$L$の要素の順序関係ではなくequivalence(`(n !== o)`)を用いて判定を行っているのが分かると思います。

#### 修正2. 状態の更新の修正

抽象値の更新は`smerge`が行っているようです:
> https://github.com/JuliaLang/julia/blob/d474c98667db0bf4832e4eeb7beb0e8cfc8b7481/base/compiler/typelattice.jl#L226-L233
```julia
function smerge(sa::Union{NotFound,VarState}, sb::Union{NotFound,VarState})
    sa === sb && return sa
    sa === NOT_FOUND && return sb
    sb === NOT_FOUND && return sa
    issubstate(sa, sb) && return sb
    issubstate(sb, sa) && return sa
    return VarState(tmerge(sa.typ, sb.typ), sa.undef | sb.undef)
end
```



やや複雑ですが、基本的には`tmerge`が更新を引き受けているようです。`tmerge`はより複雑なのでここで紹介はしませんが、$\sqcup$(meet)に相当する演算を行うgeneric functionです。

Juliaの型推論ルーチンは最も抽象的な抽象値である`Any`を頂上とし、下に行けば行くほど具体的な抽象値[^5] になるようなlatticeに対して動作し、状態がlatticeの底から上に登るように更新されていくので[^6] 、この記事で実装したconstant folding propagation problemが使用する$\sqcap$ではなく、$\sqcap$と対をなす演算$\sqcup$を用いているというわけです。

[^5]: つまりJuliaの型推論のlatticeの順序関係としては、例えば、`⊥(Bottom) ⊑ Const(1) ⊑ Int ⊑ Union{String,Int} ⊑ ⊤(Any)`というようになります。

[^6]: Juliaの型推論は推論がうまくいかない場合、latticeの底に位置する`Core.Bottom`ではなく、頂点に位置する`Any`と推論することからも分かると思います。

いずれにせよ、Juliaの型推論ルーチンも、論文のアルゴリズムをベースとしつつも、この記事と同様の修正をあてているのが分かると思います。


## まとめと雑感

この記事では、[Mohnen, Markus. “A Graph-Free Approach to Data-Flow Analysis.” _CC_ (2002).](https://www.semanticscholar.org/paper/A-Graph-Free-Approach-to-Data-Flow-Analysis-Mohnen/5ad8cb6b477793ffb5ec29dde89df6b82dbb6dba?p2df)で提案された、BB graphを明示的なデータ構造として用いずにプログラムの抽象解釈を行うことができるアルゴリズムを紹介し、実際に実装してみました。
実装する中で論文側の間違いを発見したわけですが、erattaも出ておらず、また論文を疑って間違いを修正するということは個人的には全く経験したことがなかったので、この記事の結論に辿り着くまでにだいぶ苦労しました。[Akira Kawata](https://akawashiro.github.io/)さんが一緒に悩んでくれたので解決しました。この場で改めて感謝したいと思います。
また思いついた修正点と同様のコードがJuliaの型推論のルーチンにもあったのは、なんだか嬉しかったです。もしかしたらこの修正点の必要性は、この記事を読んだ方と、Juliaのcompilerの開発者の方しか気がついていないのかも知れません。今度Juliaのslackで聞いてみようと思います。


## ノート

- この記事の本体及び使用したコードは[https://github.com/aviatesk/aviatesk-zenn-content](https://github.com/aviatesk/aviatesk-zenn-content)で公開しています。タイポなど見つけた際は是非この記事へコメントあるいはrepositoryの方へPRをいただけると嬉しいです。
- この記事は僕がメンテナをしているJuliaのliterate programming package [Weave.jl](http://weavejl.mpastell.com/dev/)を用いて、`.jmd`ファイルから[zenn](https://zenn.dev/)でpublishする用のmarkdownを生成しています。今後zennでJuliaに関する記事を掲載しようとしている方の参考になるかも知れません.
