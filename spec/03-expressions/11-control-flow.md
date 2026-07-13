# 制御構造

## ブロック式と give 文

ブロックを式として評価するには`give`文が必要です。

```plew
val result = {
    val temp = calculateSomething()
    give temp * 2  // ブロックの戻り値
}

// エラー: give がないブロックは式として評価不可
val invalid = { 42 }  // コンパイルエラー
```

## 条件分岐

```plew
val result = if condition {
    give "true case"
} else if anotherCondition {
    give "else if case"
} else {
    give "else case"
}
```

`else if` は専用キーワードではなく、`else` の本体が単一の `if` であるときの自然な連鎖です（`elif` のような独立キーワードは持ちません。近接言語の Swift / Rust と揃えます）。

## 条件チェーン（束縛つき条件）

`if`/`while`/`guard` の条件は**条件チェーン**で、`&&` で連結した 1 つ以上の**節**からなります（`else if` の `if` も同じ）。各節は次のいずれか：

- **Bool 式** — 真なら成立。
- **反証可能束縛** `PATTERN = expr` — `expr` がパターンに一致すれば束縛して成立、しなければ不成立。

節は左→右に短絡評価され、ある節が不成立になった時点で残りは評価されず、チェーン全体が不成立になります。束縛節が導入した変数は、**後続の節とブロック本体**で有効です。

```plew
if Optional.Some(value: val flag) = optional && flag {
    // optional が Some で、かつ束縛した flag が真のときだけ実行
    // flag はここで使える
}

while Optional.Some(value: val line) = reader.next() && !line.isEmpty() {
    process(line: line)
}
```

- 成立時に本体を実行するのが `if`/`while`。**不成立時**に本体（発散必須）を実行するのが `guard`（下記）。
- **束縛節は `&&` でのみ連結できます**（`||` 不可）。`||` だと束縛が成立しない経路ができてスコープが壊れるためです。Bool 節の**内部**では `||` は自由に使えます（`if a && (b || c) { … }`）。
- 束縛の被検査式 `= expr` のあとの**トップレベル `&&`** が次の節を区切ります（被検査式自体に `&&` を含めたいなら括弧で囲む。ただし束縛の被検査式が Bool の連言になることは通常ありません）。Plew は[構造体生成が JSX 限定](../02-type-system/05-structs-enums.md)なので、本体の `{` と式を取り違える曖昧さは生じません。
- このヘッダの `&&` は[値レベルの `&&`](12-operators.md)（Bool 専用の短絡）と**同じ「短絡する連言」概念ですが、脱糖は別物**です（ヘッダの `&&` は束縛節も連結できる条件チェーン文法）。

## パターンマッチング

```plew
val result = match expression {
    42                                      => "the answer"
    Color.Red(intensity: val intensity)  => "red with intensity {intensity}"
    (val x, val y)                          => "record x={x}, y={y}"
    _                                       => "default case"
}
```

パターンの**束縛は `val`／`mut val` で明示**します（refutable な `match`/`if`/`while`/`guard` では、bare の名前は既存の値・リテラル・バリアントとの**マッチ**）。フィールド名と束縛名が同じなら **punning** で省略でき、`Color.Red(val intensity)` ≡ `Color.Red(intensity: val intensity)`、`(val x)` ≡ `(x: val x)` です。

- **match のアクセスモード**：コピー可能な storage place は裸の `match x` と書けます。`unique` な storage place は裸の `match x` では消費せず、v1 ではコンパイルエラーです。所有権を渡して分解したい場合は `match move x` と書きます。名前の無い `unique` 一時値（`match <FileResult /> { ... }` など）は owned home を持つため裸で書け、match 文なら match 文終端、値位置 match なら match 式終端で破棄されます。
- **consuming match**：`match move x` は `x` 全体を consume します。enum の arm 判定は subject を破壊せずに行い、最終 arm が決まった後で active payload を pattern に従って束縛または破棄します。したがって refutable pattern の試行途中で subject を部分破壊して次 arm を試せなくなることはありません。match 後、元の place は moved であり使えません。
- **`match borrow` / `match inout` は v1 では持ちません**。有用にするには arm scope を持つ借用束縛・inout 束縛が必要ですが、これは借用変数とライフタイム管理の導入を伴うため将来機能とします。
- **破棄 `_`** — 値を受け取らずに捨てるワイルドカードです。束縛を作らないので `val` は付けません（`Optional.Some(value: _)`、`(val x, _)` など、パターンが書ける位置ならどこでも）。`_` は**書き込み専用のシンク**で、値式としては読めません（`val x = _` のような取り出しは不可）。
- **`unique` payload の `_`**：consuming match で active payload の `unique` component を `_` で受けた場合、その component は pattern commit 時に破棄されます。複数の component は payload 宣言順に処理し、`val` で束縛した component は通常の local として後で破棄されます。
- **全フィールド明示** — 構造体・レコード・enum バリアントを `{ … }`／`( … )` で開くパターンは、その型の**全フィールドを明示**します（`val`/bare で束縛するか `_` で破棄）。未記載フィールドの暗黙無視はせず、残り全部を捨てる `..` も持ちません（Rust の「全フィールド or `..`」から `..` の逃げ道を外した形）。**フィールドを増やすと既存パターンがコンパイルエラーになり**、新フィールドの扱いを必ず決めさせます（静かな取りこぼしを防ぐ）。値全体をそのまま受けたいだけなら、開かずに `val x`（束縛）か `_`（破棄）を書きます。
- **or パターン `Pat1 | Pat2 | …`** — `|` で並べた選択肢の**どれかに一致すれば成立**し、1 つのアーム本体を共有します（`Color.Red | Color.Green => …`）。網羅性は列挙した全バリアントを数えます。`|` はパターン位置でのみ区切りで、`=>` 以降の本体に書く `|` は通常のビット OR です。
  - **束縛は全選択肢で一致**（Rust と同じ）。各選択肢は**同じ束縛名の集合を同じ型で**導入しなければなりません（本体はどの選択肢で一致しても同じ束縛を見るため）。食い違えばコンパイルエラー。**nullary バリアントは束縛を持たないので常に並べられます**（最頻ケース）。
  - **フィールド付きバリアントも結べます**。フィールド名が変種ごとに違っても、`{ field: val name }` の rename で**共通の束縛名に揃えれば**またげます：

    ```plew
    match shape {
        Shape.Circle(radius: val size) | Shape.Square(side: val size) => size * 2  // 両者 size: I64
        Shape.Point                                                         => 0
    }
    ```

    フィールド名が同じバリアント同士なら punning でそのまま（`Node.Leaf(val value) | Node.Unary(val value)`）。`_` で捨てたフィールドは束縛を作らないので選択肢間で一致する必要はありません（`A(x: val v, y: _) | B(z: val v, w: _)` は `v` だけ揃えばよい）。**束縛名や型が食い違うのはエラー**（`Circle(val r) | Square(val s)` は `r`/`s` が全選択肢で束縛されず不可）。

アームの右辺は次の 3 形式：

- **ベア式** `=> expr` — その式の値をアームの結果にする。
- **ブロック** `=> { …; give x }` — 複数文のあと `give` で値を返す。
- **発散ブロック** `=> { … panic / return … }` — 値を生まず、任意の期待型と適合する（式全体の型は発散しないアームから決まる。後述「panic と発散」）。

### 網羅性（Rust 流）

`match` は**網羅的**でなければなりません。スクルーティニ（`match` する値）の型がとり得る**すべての値がいずれかのアームに一致する**ことをコンパイラが検査し、漏れがあれば**コンパイルエラー**にします（どのケースが未カバーかを示す）。`match` は値を生む式なので、「どのアームにも当たらず結果値が無い」状態を型システムとして許せません（Rust と同じ方針）。

- **enum**：全バリアントを列挙すれば網羅（`Optional` なら `Some`／`None` の両方）。
- **`Bool`**：`true`／`false` の両方。
- **ワイルドカード `_` と捕捉束縛**：残り全部を受けます。`_ => …`（値を捨てる）か `val x => …`（残りを `x` に束縛）。整数・`String` など**全値を列挙できない型は `_`（または捕捉束縛）が必須**です。

アームは**上から順に試し、最初に一致したアームだけ**を実行します（C のような fall-through は無く、`break` も不要）。先行アームで完全に覆われて**到達不能になったアームは警告**します（Rust 同様）。本体が発散するアーム（`_ => panic "unreachable"` など）もパターンはカバーに数えるので、「論理上起きないはず」を明示的に落とすのに使えます。

> `match` アームに**ガード（`if` 条件）は持ちません**。絞り込みは「束縛してからネストした `match`」か、`match` の前段の `if`／`guard`（[条件チェーン](#条件チェーン束縛つき条件)）で行います。これにより網羅性は**純粋に構造的**に判定でき、Rust の「ガード付きアームは網羅に数えない」という例外規則が要りません。

## ループ

```plew
// 無限ループ（式として使用可能）
val result = loop {
    val data = getData()
    if data.isValid() { 
        break data.process()  // ループの戻り値
    }
    if shouldRetry() { continue }
    break <Error.Failed />  // エラー時の戻り値
}

// While ループ
while condition {
    // 処理
}

// For ループ（レンジ・分解対応。ループ変数は val で宣言）
for val i in 0..<n {
    // 0, 1, …, n-1（半開）。0..=n なら n まで含む
}

for (val key, val value) in dictionary {
    // 処理
}

for Person { val name, val age } in people {
    // 処理
}
```

- ループ変数は `val`／`mut val` で**新規宣言**します。bare 名にすると既存の `mut` 変数へ代入し、ループ後も生存します（`val`＝新規・bare＝既存。→ [値](../01-basics/03-values.md)）。
- **値を返せるのは `loop` だけ**：`loop { … break x }` は式として `x` を返します。`while`／`for` は値を持たず、`break` は**値なしの脱出のみ**（`break x` は不可）。ループ本体ブロックでは `give` も使えません（`give` は通常ブロックと `if`／`match` アーム専用）。ラベル付きループ・多重 `break` は持ちません。
- `for` が回すのは **`Iterable`**（`fn iterator() -> Iter` で毎回新しいカーソルを産む）で、`Array`・辞書・レンジ等が準拠します。実際に値を出すのは **`Iterator`**（`inout fn next() -> Optional[Item]` の消費カーソル）で、両者は別トレイト（コレクションは値意味論上「自分のカーソル」を持てず、多重走査もしたいため）。`Iterator` 自身も `Iterable`（自分を返す）なので iterator を直接 `for` できます。正確なシグネチャは下記[イテレータ・プロトコル](#イテレータプロトコル)。
- `for (val key, val value) in dict` は、辞書が要素 `(key: K, value: V)`（ラベル付きタプル）の `Iterator` を産み、それを分解しています。

### イテレータ・プロトコル

```plew
trait Iterator {
    type Item
    inout fn next() -> Optional[Item]      // 自分のカーソルを inout で前進・尽きたら None
}

trait Iterable {
    type Item
    type Iter: Iterator[Item = Item]       // 産むカーソルの具象型（any で包まない＝単相化・boxing なし）
    fn iterator() -> Iter                   // self を借用し新しいカーソルを値で返す
}
```

- **`for val x in e` の脱糖**：

```plew
mut val it = e.iterator()                  // next が inout ゆえ mut val
loop {
    match it.next() {
        Optional.Some(value: val x) => { /* 本体 */ }
        Optional.None                  => break
    }
}
```

- **`Iterator` 自身が `Iterable`**：すべての `Iterator` を `Iterable` にする blanket（[`impl B as A`](../02-type-system/09-extensions.md) 形・core 提供）が `iterator()` で自分を返します。

```plew
impl Iterator as Iterable {
    type Item = Self.Item
    type Iter = Self
    fn iterator() -> Self { self }          // 値で返す＝コピー。`for x in someIter` は元の束縛を消費しない
}
```

- **`next` は `inout fn`**（カーソルを書き換える＝破壊操作の命令形メソッド）、**`iterator()` は名詞アクセサのメソッド**、要素を産む `Item`/`Iter` は[関連型](../02-type-system/08-traits.md)。`type Iter: Iterator[Item = Item]` は**境界の中で関連型を束縛**する記法（`any Iterator[Item=I32]` と同形で、存在型に限らず境界一般で使える）。
- v1 はイテレータ＝コピー可能な値（unique イテレータは `allowUnique` 待ちで additive）。提供メソッド（`map`/`filter`/`enumerate`/`zip`/`count` 等）は[`Iterator` のベアな `impl Iterator` に置く](../02-type-system/08-traits.md)（準拠 `impl T as Iterator` で自動的にベアに載り、境界 `where T: Iterator` 越しにも使える・正確な署名は core-library）。
- **レンジの反復**は要素が `Step` のときだけ `Iterable` になる（条件付き準拠 `where T: Step`）→ [レンジ](../01-basics/02-basic-types.md#レンジhalfopenrange--closedrange)。

## ガード文

条件が満たされない場合に早期リターンやブロック脱出を行います。条件は他の制御構文と同じ[条件チェーン](#条件チェーン束縛つき条件)（`&&` で Bool 節と反証可能束縛 `PATTERN = expr` を連結）で、束縛した変数は guard を抜けた後で使えます。

v1 の guard 条件チェーンは `unique` storage place を消費する構文を持ちません。`Optional[File]` のような by-value `unique` generic も v1 では書けないため、通常の unwrap は `Optional[Ref[File]]` などコピー可能値で行います。将来 `allowUnique` と借用束縛を導入する場合に、`guard move ...` などの consuming guard 構文を改めて設計します。

```plew
// 基本的な条件チェック
guard user.isAuthenticated() && user.hasPermission(name: "read") {
    return <Error.Unauthorized />
}
// ここに到達するのは guard 条件が true の場合のみ

// 列挙型のunwrapと変数代入
guard Result.Ok(value: val value) = someResult {
    return <Error.Failed />
}
// ここでは value が使用可能

// 複数の条件を組み合わせ
guard Optional.Some(value: val data) = maybeData && data.isValid() {
    return <Error.Invalid />
}
// ここでは data が使用可能
```

## panic と発散

`panic "メッセージ"` はプログラムを停止させる**文**です（`return`/`break` と同じく、その先へ進まない＝発散する制御フロー）。回復可能な失敗には使わず（それは `Result`/`try`）、**回復不能なバグ**を即座に・大きな声で落とすために使います。catch はできません。

```plew
guard Optional.Some(value: val config) = maybeConfig {
    panic "config is missing"   // guard 本体は発散する必要がある → panic で満たす
}

val config = match maybeConfig {
    Optional.Some(value: val v) => v
    Optional.None                  => { panic "config is missing" }  // 発散アーム
}
```

- メッセージは診断用の `String`（文字列展開可）。型付きの値は運ばない。
- **`panic` は abort＝即座にプロセスを停止し、スタックを巻き戻さない**。したがって**`deinit` は走らない**。catch できない以上、巻き戻しても回復はできず「死ぬまでの掃除」にしかならない上、`panic` は不変条件が壊れた状態なので、そこでユーザーコード（`deinit`）を走らせると二次災害（壊れたデータの書き込み・二重 panic）を招く。OS がメモリ・ファイルディスクリプタ・ソケットを回収するので、資源リークにもならない。`deinit` の「決定的資源解放」契約は**正常終了パス**（スコープ離脱・`return`・`try`/`Result` でのエラー伝播）で保証されるもので、`panic` はその契約の外にある明示的な脱出口。失敗し得るが後始末したい処理は `panic` ではなく `Result`＋明示 flush で書く。
  > 実装：unwind テーブル／landing pad を持たず trap 一発（Rust の `panic = "abort"`・Swift の `fatalError`/`precondition` trap と同じ）。「死ぬ前にログだけ吐く」等が要れば、将来 unwind ではなく atexit 風フックとして additive に足せる。
- **式ではない**ので、式の位置には置けない（`match x { None => panic "..." … }` のように `guard`/`match` を使う）。
- **発散規則**：全経路が `panic`/`return`/`break`/`continue` で抜けるブロックは値を生まないので `give` が不要で、`if`/`match` のアームなどで**任意の期待型と適合**する（式全体の型は発散しないアームから決まる）。
- `spawn` スレッド内の panic は**プロセス全体を停止**する。スレッド単位で扱いたい失敗は `Result` を返して `join()` 経由で受け取る（→ [非同期処理とメモリ管理](../04-execution/14-concurrency.md)）。

## assert ── 条件付き panic

`assert(x > 0)` は条件が偽のとき [`panic`](#panic-と発散) する**通常の関数**です（真なら何もしない・戻り `()`）。回復不能なバグ＝**満たされて当然の不変条件**を、破れた瞬間に大きな声で落とすために使います。診断メッセージは任意引数：`assert(x > 0, message: "x must be positive")`。

- **常時 ON（全ビルド共通）**。最適化レベルで意味論は変わりません ── [整数オーバーフロー](../01-basics/02-basic-types.md#整数の実行時セマンティクスオーバーフロー)・0 除算・NaN 比較の panic と同じ「リリースでだけ落ちないバグを作らない」方針（観測挙動は唱えた意味から逸れない）。Rust の `assert!`／Swift の `precondition` に対応します。
- 内部は `panic` ゆえ **abort**（巻き戻さない・`deinit` は走らない・catch 不可）。ただし `panic` と違い**発散文ではなく**、条件が真なら素通りする**ただの関数呼び出し**です（構文の特別扱いは不要）。構文が参照しない＝**lang item ではない**ので、`print` 同様 `import` が要ります。
- **`debugAssert`（最適化ビルドで除去される段）は当面持ちません＝additive 保留**。重い不変条件チェックを本番で外したい需要はありますが、除去段は「観測挙動が唱えた意味から逸れない」方針と**唯一緊張する部分**（壊れたプログラムの loud 化を遅らせる＝リリースでだけ素通りする）。入れるなら *呼び出し位置で除去段と分かる別名* `debugAssert` ＋ ビルドプロファイル定義を伴って後から非破壊で足します。常時チェックが既定で、除去は明示的に opt-in。
