# 型変換と演算子

## 型キャストと From トレイト

`x as T` キャストは、ターゲット型 `T` の `From[Source]` 実装（**無名 factory**、ラベル `from`）を呼ぶシンタックスシュガーです（`x as T` ⟺ `<T from=x />`）。**唯一の例外は [`newtype`](../02-type-system/10-newtype.md) と元の型の間の `as`** で、これは `From` 実装ではなく表現が同一な型どうしのゼロコストな再タグ（構造的に必ず成功）として処理されます。

**`as` は常に infallible（必ず成功して `T` を返す）です。** `From` は「**全域（total）な変換**」── 無損失（`I32→I64`）も、浮動小数の規定された丸め（`I64→F64`・`F64→F32`）も含め、**必ず値を作れる**変換だけが `From`＝`as` に乗ります。`as` が panic することも `Result` を返すこともありません（戻り型が `as T` の見た目どおり常に `T` であることを保つ）。

```plew
trait From[Source] {
    factory(from: Source)                // 無名 factory 要求（戻りは暗黙 Self・ラベル from）
}

impl I64 as From[I32] {
    factory(from: I32) {
        return <I64 ... />               // I32 から I64（無損失・全域）
    }
}
impl I64 as From[I16] {                  // 別ソースは別 conformance
    factory(from: I16) { /* ... */ }
}

val x: I32 = 10
val y: I64 = x as I64  // <I64 from=x /> と同等
```

ターゲット型は `as T` の `T`（や `try` の関数戻り型）で常に明示的に決まるので、`From` の無名 factory は**ソース引数の型でオーバーロード解決**されます（→ [メソッドのオーバーロード](../02-type-system/07-methods-impl.md)）。これにより 1 つの型が複数ソースから変換可能（`I64` は `From[I32]` と `From[I16]` の両方に準拠）になります。`try` のエラー変換も同じ `From` を使います（→ [エラーハンドリング](13-error-handling.md)）。

## 失敗し得る変換（TryFrom トレイト）

範囲外になり得る変換（`I64→I8` の縮小、`F64→I32`、文字列パースなど）は**全域でない**ので `From`＝`as` には乗せず、**`TryFrom[Source]` の fallible factory `checked`** で表します。戻りは `Result[Self, Error]` で、失敗を静かに切り詰め／飽和させず（Rust の `as` の silent truncate を採らない）、エラーを値として返します。

`From` は共通ケースなので**無名 factory ＋ `as` 糖衣**、`TryFrom` は稀なので**名前付き factory `checked`** ── 非対称にすることで、同じ `Source` から「失敗しない版（`From`）」と「失敗し得る版（`TryFrom`）」を**両方**書いても衝突しません（無名 `factory(from:)` と名前付き `factory checked(source:)` で別物）。

```plew
trait TryFrom[Source] {
    type Error                                    // 出力＝関連型（impl が一意に決める）
    result[Error] factory checked(source: Source) // fallible factory（→ Result[Self, Error]）
}

impl I8 as TryFrom[I64] {
    type Error = RangeError
    result[RangeError] factory checked(source: I64) {
        guard source >= -128 && source <= 127 { return <Result.Err error=<RangeError /> /> }
        return <Result.Ok value=<I8 ... /> />
    }
}

val r = <I8.checked source=big />        // Result[I8, RangeError]
val n = try <I8.checked source=big />    // try で早期 return（前置 try が JSX 全体に掛かる）
```

- **`as` の糖衣は持たない**：fallible 変換は `<T.checked source=… />`（factory）で明示する（infallible な `as` と取り違えないよう、可謬は常に factory 形）。
- **`Error` は関連型**（impl が一意に決める出力なので・`From` の `Source` が型引数なのと対）。`checked` のエラーは `try` の `From` 変換に乗って関数の戻りエラー型へ集約できる（→ [エラーハンドリング](13-error-handling.md)）。
- **判定基準＝全域か可謬か**：必ず値を作れる（無損失 or 規定の丸め）＝`From`／`as`。表現できない入力があり得る＝`TryFrom`／`checked`。浮動小数の丸め（精度欠落）は「型を取り違える wrap」ではなく IEEE の規定動作なので `From` 側（静かな嘘ではない）。
- **浮動小数→整数は `TryFrom[F64]`**（`<IX.checked source=f />`）：小数部があり得る（全域でない）ので `as` には乗らない。変換は **0 方向への切り捨て**（C/Rust 慣習・`3.7→3`・`-3.7→-3`）で、整数部が**ターゲットの範囲を外れる／`NaN`／±inf** のときだけ `Err`（`RangeError`）を返す。範囲内なら小数部を捨てて必ず成功する（精度欠落＝丸めの規定動作で、これは「嘘」ではない）。`NaN` 判定を順序比較より**先**に行う（float の `==`/`<` は NaN で panic するため → [基本型](../01-basics/02-basic-types.md#浮動小数の実行時セマンティクスnan--inf)）。
- **命名**：factory 名 `checked` は暗黙の「生成（create）」の修飾語＝「範囲を検査して作る」を表す**過去分詞形容詞**（`<I8.checked source=… />`＝"checked construction"）。ラベル `source` が検査対象の入力を名指す（[ファクトリ名の指針](../02-type-system/05-structs-enums.md#ファクトリ名の指針規約)）。退けた候補：単独動詞 `convert`（生成でなく「ソース側のメソッド」に読める）・`checkedFrom`／`tryFrom`（ラベル `source` と "from" が重複）・`fit`／`parse`／`coerce`（範囲・文字列・成功含意に寄り過ぎ）。可謬性は語ではなく `Result` 戻り＋呼び出し側の `try` が担う（Plew は fallible 関数に `try_` 接頭辞を付けない → [概要](../01-basics/01-overview.md)）。

**暗黙変換は持ちません。** 型変換は常に明示の `as`（数値の幅変更 `I32→I64` 等も `x as I64`）か、可謬なら `<T.checked source=… />`。例外は `try` のエラー変換（`From` を暗黙挿入）と数値リテラルの多相だけ。これによりオーバーロード解決は「ラベル＋具体型の完全一致（リテラルは互換候補に絞り一意か否か）」に保たれ、変換ランク付けの曖昧さが生じません。なお[拡張ビューの変更](../02-type-system/09-extensions.md#拡張ビューの変更は明示暗黙キャストなし)（`A`↔`A#P`）は値の表現を変えない**ビューの再解釈**で型変換ではありませんが、これも暗黙には起きず常に `#`／`#!` を明示します。

## 演算子システム

全ての演算子は、対応するトレイトのメソッド呼び出しのシンタックスシュガーです。**演算子は、オペランドの型に対応するトレイトが実装されている場合にのみ呼び出せます。** 未実装の型に演算子を適用するとコンパイルエラーになります（これは二項演算子・単項演算子・添字アクセスすべてに共通します）。なお **`&&` / `||` は例外**で、短絡評価する制御フロー糖衣でありトレイトではありません（後述「論理結合子」）。（`??` は演算子として持ちません ── 後述「Nil 合体演算子は持たない」。）

```plew
trait Add[Rhs] {
    type Output
    assoc fn add(lhs~: Self, rhs~: Rhs) -> Output
}

struct Vector {
    val x: F64
    val y: F64
}

impl Vector as Add[Vector] {
    type Output = Vector

    assoc fn add(lhs~: Vector, rhs~: Vector) -> Vector {
        return <Vector x=(lhs.x + rhs.x) y=(lhs.y + rhs.y) />
    }
}

val v1 = <Vector x=1.0 y=2.0 />
val v2 = <Vector x=3.0 y=4.0 />
val result = v1 + v2  // Vector.add(v1, v2) と同等
```

> **演算子・比較トレイトの要求は `assoc fn`（メソッドではない）**。`add`/`sub`/`compare` などは**自分を書き換えず新しい値を産む対称演算**で、両オペランドは対等な値（どちらも「主語」ではない）。これをメソッド `lhs.add(rhs:)` にすると、命令形動詞なのに非破壊＝「`add` したら `lhs` が変わる」と誤読させる（Swift の `sort`/`sorted` 規約に反する）。`assoc fn add(lhs~:, rhs~:)` なら受け手を演じず、操作名として正直（Swift の演算子 `static func +`、Rust の `Step` と同形）。**オペランドラベルは抑制**（`~:`）：対等ピアの `lhs`/`rhs`（単項は `value`）は位置以上の情報を持たない位置名に堕ちているため、[引数ラベルの指針](../01-basics/04-functions.md#引数名とラベルの指針規約)に従って落とす（明示呼び出しは位置引数 `T.add(a, b)`／演算子脱糖は名前＋位置で解決するので透過）。一方**コンテナアクセサ**（`Index`/`IndexSet`）と**分解アクセサ**（`Chain.chain()`）は受け手（コンテナ／自分自身）が主語なのでメソッドのまま、**破壊する反復子**（`Iterator.next` の `inout fn`）は命令形メソッドで規約どおり。

算術二項演算子はそれぞれ対応トレイトの糖衣です（いずれも型引数 `[Rhs]` と `type Output` を持ち、右オペランド型でオーバーロードできます）。

| 演算子 | トレイト | 脱糖 |
| --- | --- | --- |
| `+` | `Add[Rhs]` | `T.add(a, b)` |
| `-` | `Sub[Rhs]` | `T.sub(a, b)` |
| `*` | `Mul[Rhs]` | `T.mul(a, b)` |
| `/` | `Div[Rhs]` | `T.div(a, b)` |
| `%` | `Rem[Rhs]` | `T.rem(a, b)` |

（`T` は左オペランド `a` の型。`a + b` は `b` の型で `Add[…]` の conformance を選ぶ。）

冪乗 `**` は持ちません（`Pow[Exp]` トレイトの `assoc fn pow(base: Self, exp: Exp) -> Output`・`Add[Rhs]` と同形で多重 conformance＝`pow(base: x, exp: 2)`）。二項ビット演算 `&` / `^` / `|` / `<<` / `>>` と単項 `~` は、それぞれ `BitAnd` / `BitXor` / `BitOr` / `Shl` / `Shr` / `BitNot` トレイトの糖衣で、いずれも上と同じく **`assoc fn`**（二項は `op(lhs~:, rhs~:)`・単項 `~` は `bitnot(value~:)`・ラベルは抑制）です（優先順位は → [優先順位と結合性](#優先順位と結合性)）。

> **`/`・`%` の実行時意味**（Rust/Go/Swift に合わせる）：
> - **整数の 0 除算**（`a / 0`・`a % 0`）は **panic**（回復不能なトラップ。静かな値を返さない）。
> - **整数剰余 `%` は被除数の符号に従う**（truncated 除算。`-7 % 3 == -1`）。
> - **`%`（`Rem`）は整数型のみ**：浮動小数は `Rem` を実装せず `1.5 % 0.5` は**コンパイルエラー**（剰余が要るときは名前付きメソッド。Go/Swift に合わせる ── float の `%` は罠が多いため非提供）。
> - **浮動小数の除算**は IEEE 据え置きで `1.0 / 0.0` は `inf`（panic しない。→ [基本型](../01-basics/02-basic-types.md#浮動小数の実行時セマンティクスnan--inf)）。
> - 整数オーバーフロー（`I32.MIN / -1` 等）も **panic**（→ [整数の実行時セマンティクス](../01-basics/02-basic-types.md#整数の実行時セマンティクスオーバーフロー)）。

### 演算子のオーバーロード（右オペランド型ごとの conformance）

演算子は対応トレイトの `assoc fn` の糖衣です（`a + b` ⟺ `T.add(a, b)`）。トレイトは右オペランド型を型引数に持つので（`Add[Rhs]`）、**1 つの型が複数の右オペランド型に準拠**でき、`add` は引数型で区別される[オーバーロード](../02-type-system/07-methods-impl.md)として解決されます。

```plew
// Vector * Vector
impl Vector as Mul[Vector] {
    type Output = Vector
    assoc fn mul(lhs~: Vector, rhs~: Vector) -> Vector { /* 要素ごとの積など */ }
}
// Vector * F64（スカラー倍）
impl Vector as Mul[F64] {
    type Output = Vector
    assoc fn mul(lhs~: Vector, rhs~: F64) -> Vector { /* スカラー倍 */ }
}

val a = <Vector x=1.0 y=1.0 />
val b = a * <Vector x=2.0 y=2.0 />   // Mul[Vector]
val c = a * 2.0                       // Mul[F64]
```

`a * b` は `b` の型で `Mul[…]` の conformance を選びます。**同じ右オペランド型に複数の conformance は書けません**（同セレクタ・同シグネチャで衝突）。blanket なジェネリック impl（`Mul[T] where T: Scalar`）と具体 impl（`Mul[F64]`）の併存も不可（引数位置が generic vs 具体で混在＝オーバーロードの形不一致。specialization は無し）。

#### リテラルオペランドの型解決（型付きオペランドが決める・曖昧はエラー）

`Add[Rhs]` では `Self`（左）・`Rhs`（右）・`Output`（結果）は独立し得る。裸の整数/浮動小数リテラルは文脈から型を取るが、その規律は一つ：

1. **型を持つオペランドが演算子の型を決める（左右どちら側でも）。** 組み込み数値は同次（`impl T as Add[T] -> T`）なので両辺＋結果が同じ `T`。`big + 5` も `5 + big`（big: I32）も `5` は I32。
2. **ヘテロなトレイト witness**（`impl Money as Add[I8] { type Output = U64 }` など）では、右リテラルは **witness の `Rhs`（I8）で型付け・範囲検査**する。**周囲の文脈は `Output`（結果）を型付けするのであって、オペランドには降りない** ── `val r: U64 = money + 200` は `200` を I8 として検査し（U64 ではない）、収まらなければエラー（裏で i8 に切り詰めない）。ヘテロ witness を持てるのは非数値の左オペランド（struct/newtype/String）で、リテラルにはなり得ないので、解決は常に左→右でよい。
3. **どちらのオペランドも型を持たない**（両方裸リテラル）ときだけ、期待される結果型（注釈・引数・戻り）を共有型として使う。`val x: U64 = 1 + 2` は両方 U64。期待型も無ければ曖昧 → **エラー**（`val z = 1 + 2` は不可・サフィックスか注釈を要求）。
4. **裸リテラルの右オペランドが整数オーバーロードを複数同時に充たす**（`Add[I32]` と `Add[I64]` の両方がある）ときは**曖昧 → エラー**。宣言順で黙って選ばない。サフィックスで一意化する（`a + 10I64`）。`Rhs` を**右→左に逆探索することはしない**（全型からの探索になり曖昧を生むため）：左がリテラルの左スカラ用法（`5 + vec`）はサフィックス必須。

> 型引数 `[Rhs]` を持つのは **両辺が別型になり得る演算**（算術・添字・nil 合体・変換）です。**等価・順序は同一型上の関係なので型引数を持ちません**（次節）。

## 等価（Eq トレイト）

`==` / `!=` は `Eq` トレイトの糖衣です。等価は**同一型上の同値関係**（反射・対称・推移）なので、`Eq` は**型引数を持たず**右辺は常に `Self` です。異種比較 `impl A as Eq[B]`（A≠B）は対称律（`B as Eq[A]` が無いと壊れる）・反射律を型で保証できないため持ちません。幅の違う数値などは明示 `as` で型を揃えてから比較します。

```plew
trait Eq {
    assoc fn eq(lhs~: Self, rhs~: Self) -> Bool
}

val same = a == b   // T.eq(a, b)
val diff = a != b   // !(a == b)
```

`@[Eq]` ディレクティブはフィールドごとの `impl T as Eq` を合成します（→ [メタプログラミング](../04-execution/16-metaprogramming.md)）。

## 順序（Ord トレイト）

`< <= > >=` は `Ord` トレイトの糖衣です。`Ord` も同一型上の**全順序**なので型引数を持たず（`Ord: Eq`）、比較結果を `Ordering` で返します。4 つの演算子はすべて `compare` に展開されます。

```plew
enum Ordering {
    Less
    Equal
    Greater
}

trait Ord: Eq {
    assoc fn compare(lhs~: Self, rhs~: Self) -> Ordering
}

a < b    // match T.compare(a, b) { Less => true,     _ => false }
a <= b   // match T.compare(a, b) { Greater => false, _ => true }
a > b    // match T.compare(a, b) { Greater => true,  _ => false }
a >= b   // match T.compare(a, b) { Less => false,    _ => true }
```

`F32`/`F64` も `Ord`・`Eq` に準拠しますが、**NaN を比較すると panic** します（IEEE の「NaN はどの値とも順序が付かない」を静かな `false` で返さず落とす）。NaN 判定は `isNan()`。算術自体は IEEE 据え置きで NaN/inf を生成します（→ [基本型](../01-basics/02-basic-types.md)）。

## 論理結合子（`&&` / `||`）

`&&` / `||` は**短絡評価する制御フロー**で、`if` の糖衣です（演算子トレイトではありません）。両オペランドは `Bool` 限定で、オーバーロードできません。

```plew
a && b   // ≡ if a { give b } else { give false }
a || b   // ≡ if a { give true } else { give b }
```

右オペランドは左の結果次第で評価されません（`b` は `a` が真のときだけ走る）。これは「**右辺は左辺が成り立つときだけ安全に評価したい**」ガード（範囲チェック後の添字、存在確認後のアクセス等）を 1 式で書くためで、`while i < n && p(arr[i])` のようなループ条件で特に効きます。`if` の糖衣として定義されるので、非評価は隠れた魔法ではなく明示的な意味です。

> 前置 `!`（`Not`）は eager（必ずオペランドを評価する値演算）なので、短絡する `&&`/`||` と違いトレイトのままです（`Bool` 専用の論理否定）。整数のビット反転は**別演算子** `~`（`BitNot`）で、二項ビット演算 `& ^ | << >>` ともども対応トレイトの糖衣です（→ [優先順位と結合性](#優先順位と結合性)）。

> **条件位置での `&&`**：`if`/`while`/`guard` の条件では、`&&` は Bool 節に加えて反証可能束縛 `PATTERN = expr` も連結します（[条件チェーン](11-control-flow.md)）。値レベルの短絡と同じ概念ですが脱糖は別物で、`||` は束縛節を連結できません。

## 単項演算子

前置演算子も二項演算子と同様にトレイトベースで、対応するトレイトが実装されている場合にのみ呼び出せます。

前置 `!`（論理否定）は`Not`トレイトのシンタックスシュガーです。

```plew
trait Not {
    type Output
    assoc fn not(value~: Self) -> Output
}

impl Bool as Not {
    type Output = Bool

    assoc fn not(value~: Bool) -> Bool {
        // 否定の実装
    }
}

val flag: Bool = true
val negated = !flag  // Bool.not(flag) と同等
```

前置 `-`（符号反転）は `Neg` トレイトのシンタックスシュガーです（`-x` ⟺ `T.neg(x)`）。

```plew
trait Neg {
    type Output
    assoc fn neg(value~: Self) -> Output
}

impl I32 as Neg {
    type Output = I32

    assoc fn neg(value~: I32) -> I32 {
        // 符号反転の実装
    }
}

val n: I32 = 5
val m = -n  // I32.neg(n) と同等
```

- 符号付き整数（`I8`…`I64`）と浮動小数（`F32`/`F64`）が `Neg` を実装します。
- **符号なし整数（`U8`…`U64`）は `Neg` を実装しない**ので、`-x`（`x: U32` 等）は**コンパイルエラー**です（「演算子は実装トレイトのある型でのみ」の一般則どおり。静かなラップ値を返しません）。
- 二項の `-`（減算）は別トレイト `Sub` で、前置か中置かは構文位置で区別します。

> **負の数値リテラル**は `Neg` ではありません。`-128` のように数値リテラル直前にある `-` は[リテラルの一部](../01-basics/02-basic-types.md#リテラルの型付け多相文脈で確定)として畳み込まれ、リテラル全体で型範囲を検査します（`-128` は `I8` の最小値として有効。`I8.neg(128)` 経由だと `128` が `I8` に収まらず溢れてしまう）。`Neg` は変数・式に対する実行時の符号反転に使います。

## 添字アクセス

`collection[key]`の添字アクセスは`Index`トレイトのシンタックスシュガーです。`Index`を実装した型に対してのみ使用できます。

```plew
trait Index[Key] {
    type Output
    fn index(key: Key) -> Output
}

impl MyArray[T] as Index[U64] {
    type Output = T

    fn index(key: U64) -> T {
        // 要素取得の実装
    }
}

val array = <MyArray ... />
val first = array[0]  // array.index(key: 0) と同等
```

### 添字代入（IndexSet）

`collection[key] = value` の添字**代入**は `IndexSet` トレイトの糖衣です。Rust の `IndexMut` 相当ですが、Plew は参照・借用を持たない（可変な場所を返せない）ので、**可変参照ではなくセッターメソッド**にします。

```plew
trait IndexSet[Key] {
    type Value
    inout fn indexSet(key: Key, value~: Value)
}

impl MyArray[T] as IndexSet[U64] {
    type Value = T

    inout fn indexSet(key: U64, value~: T) {
        // 要素設定の実装
    }
}

mut val array = <MyArray ... />
array[0] = x  // array.indexSet(key: 0, x) と同等
```

- 読み取り（`Index`）と代入（`IndexSet`）は**独立したトレイト**で、読み取り専用コレクションは `Index` だけを実装できます。
- `collection[key] += x` などの複合代入は、**読み取り（`Index`）＋演算＋代入（`IndexSet`）**に展開されるので、両方の実装が要ります（→ [複合代入演算子](#複合代入演算子)）。
- レシーバ自身を書き換えるので `indexSet` は `inout fn`。可変束縛（`mut val`）にしか使えません。
- 添字越しの `inout` メソッド呼び出し（`arr[i].inoutMethod()`）やネストした代入（`a.b[i].c = x`）も、同じ `Index`→（変更）→`IndexSet` の **get-modify-set** に脱糖されます。場所（place）の文法・脱糖・重なる `inout` の禁止は [場所越しの変更](../01-basics/03-values.md#場所place越しの変更) を参照（複合代入はその特殊形）。

## オプショナルチェーン（Chain トレイト）

`?.` は「値を持つ／持たない」のいずれかに分岐する型へのメンバアクセスのシンタックスシュガーです。`Chain` トレイトを実装した型でのみ使用できます（Optional 専用ではなく、`Chain` を実装した任意の型で使えます）。

```plew
trait Chain {
    type Value
    fn chain() -> Optional[Value]        // 値か空かに分解する
    factory fromValue(value: Value)     // 値から再構築する（factory・戻り Self）
    factory empty()                      // 空を再構築する（factory・戻り Self）
}
```

`?.` は**直後の後置メンバアクセス連鎖全体**（`.field` / `.method(args)` / `[i]` / `->` の並び。次の `?.` が現れるまで）を取り込み、それをアンラップした値に適用します（Swift 式の伝播）。`receiver?.<cont>` は次のように展開されます（`receiver: R` は `R: Chain`、式全体の結果型 `O` も `Chain` を実装する型・`<cont>` はアンラップ値 `v` に根ざした後続アクセス）:

```plew
match receiver.chain() {
    Optional.Some(value: val v) => <O.fromValue value=v.<cont> />
    Optional.None                  => <O.empty />
}
```

レシーバが空なら `<cont>` は評価されず、式全体が空になります（短絡評価）。`chain()` が値を返したときだけ後続アクセスを評価し、結果を `fromValue` で包み直します（`fromValue` / `empty` は**factory なので JSX `<O.… />` で生成**し、構築点が見えます）。だから `a?.b.c` は「`a` が空なら全体が空、値があれば `b.c` を辿る」＝末尾の `.c` まで含めて短絡します（`(a?.b).c` ではない）。

```plew
val name = user?.profile.name    // user が空なら name は空。値があれば .profile.name を辿る
val n    = a?.b.c.doubled()      // a が空なら空。値があれば b.c.doubled() まで評価
```

次の `?.` は新しいチェーンを開始します（`a?.b?.c` は各 `?.` が独立に短絡）。中間フィールド自身が Optional のとき（`a?.b?.c` で `b: Optional[…]`）は各段に `?.` を付けます。なお `?.` は**ネストした Optional を平坦化しません**（`v.member` が `Optional[T]` を返せば結果は `Optional[Optional[T]]`）── 平坦化は隠れた意味変換なので採らず、明示に `?.` を重ねます。

```plew
val name = user?.profile?.name
// user または profile が空なら name は空
// 両方とも値を持つ場合のみ name にアクセスする
```

Optional 自身への実装は、値か空かをそのまま返すだけです。

```plew
impl Optional[T] as Chain {
    type Value = T

    fn chain() -> Optional[T] {
        return self
    }

    factory fromValue(value: T) {
        return <Optional.Some value=value />
    }

    factory empty() {
        return <Optional.None />
    }
}
```

> `chain()` の戻り値が空のケース（`Optional.None`）は付随する値を持たないため、`Result` のようにエラー情報を運ぶ型は `?.` の対象外です。エラーの早期リターンは [`try`](13-error-handling.md) を使います。

## Nil 合体演算子（`??`）は持たない

Plew に `??` 演算子はありません（Rust と同じく**あえて入れません**）。Optional のフォールバックは `Optional.unwrapOr(fallback:)` メソッドで書き、eager（既定値）と lazy（クロージャ）を**引数型で明示的にオーバーロード**します：

```plew
val port: I32 = configPort.unwrapOr(fallback: 8080)                       // eager（8080 は常に評価）
val port: I32 = configPort.unwrapOr(fallback: fn() -> I32 { return computeDefault() })  // lazy（None のときだけ走る）
```

`??` を採らない理由は **eager/lazy の遅延を明示にするため**。短絡する `??`（左辺が `Some` なら右辺を評価しない）は、右辺という**ユーザー提供の値**を暗黙に遅延評価します ── これは Plew が却下する `@autoclosure`（暗黙の遅延）と本質的に同じ隠れた挙動です。メソッド形なら lazy は**クロージャ `fn() -> T { … }` として目に見え**、eager との選択がコードに現れます（`&&` / `||` は遅延されるのが安価な `Bool` 節で、どの言語でも短絡と理解される普遍イディオムなので残しますが、任意の値を遅延する `??` は別格です）。

> オプショナルチェーン `?.`（`Chain` トレイト）は残ります。`?.` の短絡は**受け手**側（左辺が `None` なら以降のメンバアクセスを飛ばす）で、遅延部分はコンパイラの `match` 脱糖が制御し、トレイトのメソッド（`chain()` 等）自体は eager です。だから `??`（遅延されるのがユーザー提供の右オペランド）と違ってトレイト化できます。

## 優先順位と結合性

演算子の結合の強さ（番号が小さいほど強く＝先に結合する）と結合性は次の通りです。**`as`・算術・ビット・比較・`&&`/`||` の相対順は Rust と一致**します。Plew 固有の差は ① 前置 `try`/`await`（Rust の後置 `?` と逆）、② 代入は文なので優先順位に乗らない、③ 単項の顔ぶれ（`~` を分離・deref/borrow なし）の 3 点です（`??` は Rust と同じく持ちません）。

| 強さ | 演算子 | 結合性 |
| --- | --- | --- |
| 1（最強） | 後置：メンバ `.`／オプショナルチェーン `?.`／添字 `[]`／呼び出し `f()`／構造体生成 `<T/>` | 左（連鎖） |
| 2 | 前置：`-`(Neg)・`!`(Not)・`~`(BitNot)／前置キーワード `try`・`await` | 前置（右） |
| 3 | `as`（キャスト） | 左 |
| 4 | `* / %`（Mul/Div/Rem） | 左 |
| 5 | `+ -`（Add/Sub） | 左 |
| 6 | `<< >>`（Shl/Shr） | 左 |
| 7 | `&`（BitAnd） | 左 |
| 8 | `^`（BitXor） | 左 |
| 9 | `\|`（BitOr） | 左 |
| 10 | `== != < <= > >=`（Eq/Ord） | **非結合** |
| 11 | `&&` | 左 |
| 12 | `\|\|` | 左 |
| 13（最弱） | `..< ..=`（レンジ） | **非結合** |

### `as` は算術より強い（Lv3）

Plew は暗黙の数値拡幅を持たない（`I32`→`I64` も `x as I64` と明示）ので、`a as I64 + b as I64` が `(a as I64) + (b as I64)` と読まれるために `as` は算術より強い必要があります。Rust も暗黙拡幅が無く、同じ位置・同じ理由です。

### 前置 `try`/`await` は後置チェーン全体に掛かる

`try`/`await` は前置で、**直後の後置チェーン全体**を取ります：`try a().b()` は `try (a().b())`（`a().b()` を評価してから `try`）。Rust の後置 `?`（`a()?.b()` ＝ `(a()?).b()`）とは逆なので、**チェーン途中で取り出すには括弧**を使います：`(try a()).b()`。`try`/`await` は二項演算子より強く（`try f() + 1` ＝ `(try f()) + 1`、`try f() as I64` ＝ `(try f()) as I64`）、後置より弱い位置にあります。`try` の意味論は [エラーハンドリング](13-error-handling.md) を参照。

### 比較・等価は非結合（Lv11）

`a < b < c` や `a == b == c` は**構文エラー**です。後者は `Bool` も `Eq` を持つため、左結合だと `(a == b) == c` が静かに通って「3 つが等しい」と誤読されます。明示の括弧を要求します（Rust の "Require parentheses" と同じ）。

### ビット演算子（シフト ＞ `&` ＞ `^` ＞ `|`）

`& | ^ << >> ~` は `BitAnd` / `BitOr` / `BitXor` / `Shl` / `Shr` / `BitNot` トレイトの糖衣です。位置は **Rust と同じ「シフト ＞ `&` ＞ `^` ＞ `|`、いずれも比較より強い」**。これにより C の有名な罠 `x & 1 == 0` ＝ `x & (1 == 0)` を避け、`(x & 1) == 0` と読めます。シフトは加算より弱い（C/Rust と同じく `1 << 2 + 3` ＝ `1 << 5`）。Plew はジェネリクスを `[...]` で書くので `>>` がジェネリクス閉じと衝突せず、単一トークンにできます。論理否定 `!`（`Not`・`Bool` 専用）とビット反転 `~`（`BitNot`・整数）は別演算子です（Rust は `!` が両用）。

### レンジは非結合・最下位（Lv14）

`a..<b..<c` は構文エラーです。端点には算術が先に結合するので `0..<n + 1` ＝ `0..<(n + 1)`。位置は Rust に合わせて最下位にしています（レンジは `for … in`・`[]`・括弧内でしか現れず、端点が複合式でも括弧不要なため、低く置いても害がありません）。

### 代入は文

単純代入 `=` と複合代入 `+= -= *= /= %=`（およびビット系 `&= ^= \|= <<= >>=`）は式ではなく文なので、優先順位に乗りません（`if x = y` 型の取り違え事故が起きません）。複合代入の脱糖規則は[複合代入演算子](#複合代入演算子)で規定します。

## 複合代入演算子

複合代入 `a OP= b` は **`a = a OP b` への純粋な脱糖**で、専用トレイト（Rust の `AddAssign` 等）を持ちません。対応する二項演算子トレイト（`Add` 等）だけで成立し、`a += b` と `a = a + b` は**定義上つねに一致**します。

> **Rust と違い専用トレイトを持たない理由**：Rust の `AddAssign` は `Add::add(self, …)` が `self` を move 消費するため、借用越しの `self.x += 1`（`&mut self` 経由）を `a = a + b` の脱糖で書けない、という所有権・借用への対処が第一動機です。Plew は所有権・借用を持たず、`mut val` への再代入もフィールド／添字への store も単なる書き込みなので、純粋脱糖で一様に扱えます。in-place 最適化はコンパイラの責務（左辺が以後 dead なら再利用）で、言語の意味論には現れません。

二項演算子と一対一に対応します（算術 5 つ＋ビット系 5 つ）。`&&` / `||` は短絡する制御フローでトレイトを持たないため、`&&=` / `||=` は**ありません**。

| 演算子 | 脱糖 | 対応トレイト |
| --- | --- | --- |
| `+=` | `a = a + b` | `Add` |
| `-=` | `a = a - b` | `Sub` |
| `*=` | `a = a * b` | `Mul` |
| `/=` | `a = a / b` | `Div` |
| `%=` | `a = a % b` | `Rem`（整数のみ） |
| `&=` | `a = a & b` | `BitAnd` |
| `^=` | `a = a ^ b` | `BitXor` |
| `\|=` | `a = a \| b` | `BitOr` |
| `<<=` | `a = a << b` | `Shl` |
| `>>=` | `a = a >> b` | `Shr` |

`%=` は `Rem` が整数のみ提供（→ [演算子システム](#演算子システム)）なので整数型でだけ使えます。

### 成立条件

`a OP= b` は次をすべて満たすときだけ有効です：

1. **`a` が代入可能な場所**：`mut val` 束縛・`mut val` な値のフィールド・`IndexSet` を持つ添字 `coll[k]` など（`a = …` を単独で書けるのと同じ場所）。
2. **`a OP b` が成立**：`a` の型に対応する演算子トレイト（`Add[typeof b]` 等）の実装がある。
3. **結果型が左辺型と一致**：`a OP b` の `Output` が `a` の型と同じ。暗黙変換は無いので、違えば型エラー（複合代入で左辺の型が変わることはない）。

### 場所は 1 回だけ評価する

純粋なテキスト置換ではなく、**左辺の場所を 1 回だけ評価**します。`arr[f()] += x` で `f()` が二重に走ったりしません。添字の複合代入は、レシーバと添字を一時変数に束ねてから読み取り（`Index`）→演算→代入（`IndexSet`）に展開します（→ [添字アクセス](#添字アクセス)）：

```plew
arr[f()] += x
// ≈
val recv = arr
val key  = f()
recv.indexSet(key: key, recv.index(key: key) + x)
```

フィールド `obj.field += x` も同様にレシーバを 1 回評価します。

> **`??=` は持ちません**：`??` 演算子自体を持たない（[Nil 合体演算子は持たない](#nil-合体演算子-は持たない)）ので、複合形もありません。フォールバックは `opt.unwrapOr(fallback:)` を使います。
