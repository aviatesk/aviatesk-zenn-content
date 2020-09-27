---
title: "Juliaの型推論について – isaで条件分岐したブロックにおける推論"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Julia", "動的型付け", "型推論", "type", "inference"]
published: true
---

こんにちは、門脇宗平(Shuhei Kadowaki)と言います。
GitHubでは[aviatesk](https://github.com/aviatesk)というハンドルで活動していて、JuliaのIDE([Juno](https://junolab.org/), [julia-vscode](https://www.julia-vscode.org/))のメンテナなどをやっています。

最近Juliaの型推論システムを借用してJuliaプログラムのソースコードに対する静的な型解析を行うツールの開発[^1]をしているのですが、その過程でJuliaの型推論について、その中でも特に[`isa`](https://docs.julialang.org/en/v1/base/base/#Core.isa)を使って条件分岐した先のブロックにおける推論ついて少し知見ができたので共有します。

[^1]: xref: https://github.com/aviatesk/TypeProfiler.jl, https://github.com/aviatesk/grad-thesis


## 1. Juliaの型推論のflow-sensitivity

まず前提としてJuliaの型推論は["flow-sensitive"](https://en.wikipedia.org/wiki/Flow-sensitive_typing)に行われます。ここで"flow-sensitive"とはどういうことかというと、Juliaはコードをコンパイルするとき、プログラム中の様々なコードに対してその実行コンテクストに応じた型を自動で推論するということです。
例えば以下のようなコードを考えてみましょう。

```julia
if isa(a, Int)
    return sin(a)
else
    return 0
end
```

この例では1行目で`isa(a, Int)`というように`a`の型によって条件分岐を行なっているので、2行目の`sin(a)`において`a`が`Int`型でしかありえないのは明らかだと思います。
Juliaの型推論はプログラムの実行を仮想的にシミュレートすることによりこのような「2行目の`a`は常に`Int`である」といった情報を(追加的なアノテーションをせずとも)勝手に推論してくれます。

以下のコードで確認してみましょう。
`foo`の引数`a`が型不安定(この場合は`Any`型)であっても`isa` block内では`a`の型が`isa(a, Int)`の情報を用いて`a::Int`と推論されているのがわかると思います。

```julia
julia> function foo(a)
           if isa(a, Int)
               return sin(a) # <= compiler infers `a::Int`
           else
               return 0
           end
       end
foo (generic function with 1 method)

julia> code_typed(foo, (Any,); optimize = false)[1]
CodeInfo(
1 ─ %1 = (a isa Main.Int)::Bool
└──      goto #3 if not %1
2 ─ %3 = Main.sin(a::Int64)::Float64 # <= compiler inferred `a::Int`
└──      return %3
3 ─      return 0
) => Union{Float64, Int64}
```

:::message info
[`code_typed`](https://docs.julialang.org/en/v1/base/base/#Base.code_typed)のキーワード引数として`optimize = false`として指定してあげると、型推論の後に行われる[inlining](https://en.wikipedia.org/wiki/Inline_expansion)や実行されない条件分岐先の削除など各種の最適化を行わないので、型推論の結果だけを見たい時などに便利です。
:::

このJuliaの型推論ルーチンの"flow-sensitivity"は、Juliaはある変数の型がプログラム中で変わったとしてもその変数のその文脈における型に応じた最適なコード生成を行ってくれたり、型不安定なコードを部分的に型安定にしてくれたりと、パフォーマンス上非常に重要です。例えば以下のようなコードを実行する場合、`foo`に渡される引数`a`は型不安定(`Union{Int,Char,String}`)ですが、それでも`foo`3行目の呼び出し`sin(a)`は`sin(a::Int)`としてコンパイル時に解決されます[^2]。
```julia
let
    s = 0.
    for i in 1:100000
        a = rand((1, '1', "one"))
        s += foo(a)
    end
    s
end
```

[^2]: もし呼び出されるmethod(この場合は[`sin(x::Real) in Base.Math at special/trig.jl:53`](https://github.com/JuliaLang/julia/blob/93bbe0833b4918c0231c217b34c717eb0b1ad8bb/base/special/trig.jl#L53))がコンパイル時に解決されない場合、実行時に呼び出すmethodのlookupが行われ(俗に"dynamic dispatch"と呼ばれます)、パフォーマンス上の問題になる場合があります。

:::message tips
実際に

> `foo`3行目の呼び出し`sin(a)`は`sin(a::Int)`としてコンパイル時に解決されます

が起きていることを確認したい人は、[Cthulth.jl](https://github.com/JuliaDebug/Cthulhu.jl)というパッケージを使ってみましょう。

optimize optionをoffにして、`%13  = invoke foo(::Union{Char, Int64, String})::Union{Float64, Int64}`に"descend"すると以下のような出力が得られ、期待通り`foo`内の`sin(a)`において`a`の型が`a::Int`と推論されているのが確認できます:
```julia
julia> using Cthulhu

julia> descend_code_typed(Tuple{}) do
           s = 0.
           for i in 1:100000
               a = rand((1, '1', "one"))
               s += foo(a)
           end
           s
       end
...
%2  = invoke Colon(::Int64,::Int64)::Core.Const(1:100000)
%3  = invoke iterate(::UnitRange{Int64})::Core.Const((1, 1))
%11  = invoke rand(::Tuple{Int64, Char, String})::Union{Char, Int64, String}
• %13  = invoke foo(::Union{Char, Int64, String})::Union{Float64, Int64}
%14  = call #+(::Float64,::Union{Float64, Int64})::Float64
%15  = invoke iterate(::UnitRange{Int64},::Int64)::Union{Nothing, Tuple{Int64, Int64}}
↩

│ ─ %-1  = invoke foo(::Union{Char, Int64, String})::Union{Float64, Int64}
CodeInfo(
 @ none:2 within 'foo'
1 ─ %1 = (a isa Main.Int)::Bool
└──      goto #3 if not %1
└
 @ none:3 within 'foo'
2 ─ %3 = Main.sin(a::Int64)::Float64
└──      return %3
└
 @ none:5 within 'foo'
3 ─      return 0
)
...
```
:::


## 2. `isa` conditionのflow-sensitivity

このようにJuliaの"flow-sensitive analysis"は「楽に書けてしかも速いコード」を実現するために必要不可欠な機能です。
特に`isa`による条件分岐はよく使われるのでこの「`isa`で条件分岐した先のブロックの型推論」は重要なのですが、この機能について参考になるかもしれない知見が2点ほどあるので、以下ではそれを紹介します(本題)。

### 2.1. chainした`isa` conditionの制約は最後のものしか伝播しない (Julia<1.6)

`isa`ブロックがchainした場合はどうなるかみてみましょう。
Julia@1.5で以下を実行してみます:
```julia
julia> code_typed((Any,Any); optimize = false) do a, b
           if isa(a, Int) && isa(b, Int)
               return sin(a) # <= compiler infers `a::Int` ?
           end
           return 0
       end |> first
 CodeInfo(
1 ─ %1 = (a isa Main.Int)::Bool
└──      goto #3 if not %1
2 ─      (@_4 = b isa Main.Int)::Bool
└──      goto #4
3 ─      (@_4 = false)::Core.Compiler.Const(false, false)
4 ┄      goto #6 if not @_4
5 ─ %7 = Main.sin(a)::Any # <= compiler could NOT infer the type of `a`
└──      return %7
6 ─      return 0
) => Any
```
なんと、3行目の`sin(a)`において今回は`a::Int`と推論されていません！😱

この場合`isa`の条件分岐がchainしていますが、Juliaのコンパイラがchainの最後の制約(この場合は`isa(b, Int)`)しか伝播できなかったからです。
そのためこれまではこのようなコードでは、`isa` conditionを分けて"diamond"の形をしたコードブロックにしてあげる必要がありました。

```julia
if isa(a, Int)
    if isa(b, Int)
        return sin(a) # <= compiler infers `a::Int`
    end
end
```

...なのですが、この制限は最新のJuliaでは改善[^3]されています。

[^3]: xref: https://github.com/JuliaLang/julia/pull/36355

同じコードを[Julia@1.6](https://julialang.org/downloads/nightlies/)で実行してみましょう:
```julia
julia> code_typed((Any,Any); optimize = false) do a, b
           if isa(a, Int) && isa(b, Int)
               return sin(a) # <= compiler infers `a::Int`
           end
           return 0
       end |> first
CodeInfo(
1 ─ %1 = (a isa Main.Int)::Bool
└──      goto #4 if not %1
2 ─ %3 = (b isa Main.Int)::Bool
└──      goto #4 if not %3
3 ─ %5 = Main.sin(a::Int64)::Float64  # <= compiler inferred `a::Int`
└──      return %5
4 ┄      return 0
) => Union{Float64, Int64}
```
コンパイラがシュッと"diamond"を形成してくれてバシッと`a`の型が推論されているのが確認できると思います。Julia 1.6が待ち遠しいですね。

### 2.2. callee呼び出し内の`isa` conditionは伝播しない

2020/09/27時点でのJuliaでは、別の関数(callee)呼び出しの結果を用いて条件分岐を行う場合、そのcallee関数内における`isa`による型制約を伝播させることはできません。
例えば以下の`bar`(caller)は意味的には最初の例で紹介した`foo`と一緒ですが、`bar`内の3行目の`sin(a)`において、コンパイラは`a`の型を`a::Int`として推論してくれません:
```julia
julia> function bar(a)
           if baz(a)
               return sin(a) # <= compiler infers `a::Int` ?
           else
               return 0
           end
       end
bar (generic function with 1 method)

julia> baz(a) = isa(a, Int)
baz (generic function with 1 method)

julia> code_typed(bar, (Any,); optimize = false)[1]
CodeInfo(
1 ─ %1 = Main.baz(a)::Bool
└──      goto #3 if not %1
2 ─ %3 = Main.sin(a)::Any # <= compiler could NOT infer the type of `a`
└──      return %3
3 ─      return 0
) => Any
```

これは`isa`による条件分岐が`baz`(callee)の中で行われるため、それによる制約が元のcaller関数`bar`における型推論にまで伝播していないからです。

`baz`をinlineさせたらどうでしょうか:
```julia
julia> @inline baz(a) = isa(a, Int)
baz (generic function with 1 method)

julia> code_typed(bar, (Any,); optimize = true)[1]
CodeInfo(
1 ─ %1 = (a isa Main.Int)::Bool
└──      goto #3 if not %1
2 ─ %3 = Main.sin(a)::Any # <= compiler could NOT infer the type of `a`
└──      return %3
3 ─      return 0
) => Any
```
生成されるコードではinliningが起きているのが確認できます(`Main.baz(a)::Bool`が消えて、`(a isa Main.Int)::Bool`に置き換わっている)が、それでも`sin(a)`における`a`の型はうまく推論されていません。
inliningなどの最適化は型推論 _後_ に型推論で得られた情報を用いて行われるためです。

そこで、以下のように僕らプログラマがやや妥協して型アノテーションをつけてあげることで、強制的に型情報を伝えることができます[^4]:
```julia
julia> function bar(a)
           if baz(a)
               return sin(a::Int) # <= ya should know this by yourself, compiler
           else
               return 0
           end
       end
bar (generic function with 1 method)

julia> code_typed(bar, (Any,); optimize = false)[1]
CodeInfo(
1 ─ %1 = Main.baz(a)::Bool
└──      goto #3 if not %1
2 ─ %3 = Core.typeassert(a, Main.Int)::Int64
│   %4 = Main.sin(%3)::Float64 # <= now compiler knows typeof of `a`
└──      return %4
3 ─      return 0
) => Union{Float64, Int64}
```

とはいえやはりこんな自明な型アノテーションは書きたくない人もいらっしゃるかもしれません。僕もそうです。すでにissueが上がっているので興味がある人はwatchしてみると良いと思います 👀
xref: https://github.com/JuliaLang/julia/issues/37342

[^4]: あるいは妥協せず、「マクロを用いてソースレベル(i.e. surface AST level)でcallee(`baz`)をinlineする」といったややハッキーな方法を取ることで、現行のJuliaでも、`baz`の呼び出しをキープしたままなおかつ型アノテーションをせず、`bar`内の`isa` conditionによる制約を伝播させることができるかもしれません。


## 3. まとめ

今回は`isa` conditionによる型制約の伝播を中心にJuliaの型推論の機能をみてみました。
Juliaのこのあたりの機能は日々改善されているので、おそらく2.2.で紹介した制限も将来的に改善されるでしょう。Juliaのコンパイラにはもっと頑張ってもらって、僕らJuliaプログラマはどんどん楽して高効率なコードを書けるようになってほしいですね。
