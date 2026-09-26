# 構造体と列挙型

## 構造体

```plew
@[Eq, Hash]  // ディレクティブ（derive・オプション）
pub struct MyStruct[T] where T: SomeTrait {
    pub val field1: String
    pub val readonlyField: I32  // 公開 read-only フィールド
    pub(get) mut val internalSetField: I32  // 読み取り公開・書き込みは定義モジュール内のみ
    mut val field2: T = defaultValue()  // フィールドの宣言時デフォルト（生成時に省略可）
}
```

`pub` / `pub(get)` / 非公開（修飾なし）のメンバ可視性は本章末の[メンバの可視性](#メンバの可視性)を参照。`pub(get)` は `mut val` フィールドの setter だけを内部に閉じる指定です。非公開メンバは定義モジュール内のコードから見えます。

型宣言の修飾子順序は **`pub` → `unique` → `nonsendable` → `struct` / `enum`** に固定します。例えば `pub unique nonsendable struct UiResource` は合法ですが、同じ意味の修飾子を並べ替えた `nonsendable unique struct` は構文エラーです。修飾子を集合として扱わず、同じ宣言に一つの書き方だけを与えます（意味論は [unique](../01-basics/03-values.md#uniqueコピー不可型) / [sendable](../01-basics/03-values.md#sendable--nonsendableスレッド間の移送可能性)）。

拡張を型本体で既定化する構文は持ちません。拡張は常に使用箇所の `#Ext` で opt-in します。型の作者が拡張由来の振る舞いを型のベア表面に採用したい場合は、型を所有するモジュールの `pub impl Type` / `pub impl Type as Trait` に明示的な forwarding メソッドや準拠を書きます。トレイト自身の公開提供メソッドは準拠（`impl Type as Trait`）で自動的にベアに載ります（→ [トレイト](08-traits.md)、[拡張](09-extensions.md)）。

**構造体ディレクティブ**: 詳細は今後決定予定（[メタプログラミング](../04-execution/16-metaprogramming.md) 参照）

## 列挙型

```plew
@[All, Eq, Hash]  // ディレクティブ（オプション）
pub enum Color[T] where T: Format {
    Red(intensity: F64)
    Green
    Blue
}
```

**列挙型ディレクティブ**: 詳細は今後決定予定（[メタプログラミング](../04-execution/16-metaprogramming.md) 参照）

**`unique enum`**：struct と同様、`unique` を前置した enum は[コピー不可・move 専用](../01-basics/03-values.md#uniqueコピー不可型)（Swift の noncopyable enum と同型）。unique（または unique を推移的に含む型）をペイロードに持つ enum は `unique enum` 明示必須（伝染規則は struct のフィールドと同一）。ただし v1 では enum 型自身の明示 `deinit` 本体は持てません。enum の破棄時は active payload が通常通り破棄されます。

**`nonsendable enum`**：payload の型に依存しないスレッド束縛を表すときは `nonsendable` を前置できます。全バリアントが payload を持たない、または全 payload が sendable でも、この enum 自身は実スレッド境界を越えません。逆に nonsendable な payload を持つ enum は明示がなくても構造的に nonsendable です。肯定形の `sendable enum` はなく、通常の enum は payload から sendability を自動導出します（→ [sendable / nonsendable](../01-basics/03-values.md#sendable--nonsendableスレッド間の移送可能性)）。

## フィールドの統一原則

構造体と列挙型バリアントは、どちらも**名前付きフィールド**のみを持ちます。位置指向の無名ペイロード（`Some(T)` のようなもの）は書けません。生成と分解は両者で同じ構文に従います。**宣言だけは異なり**、バリアントのフィールドは修飾子なしの `field: Type` です（理由は下記）。

| | 構造体 | 列挙型バリアント |
| --- | --- | --- |
| 宣言 | `struct S { val field: T }` | `enum E { V(field: T) }` |
| 生成 | `<S field=expr />` | `<E.V field=expr />` |
| 分解 | `S { field: val binding }` | `E.V(field: val binding)` |

- 生成は JSX ライク構文のみ（下記 [インスタンス生成](#インスタンス生成) 参照）。
- **構造体は `{ }`・バリアントペイロードは `( )`**：構造体本体は記憶域（`val`/`mut val`・`pub`・メソッド）を持つ宣言ブロックなので `{ }`、バリアントのペイロードは**ラベル付きレコード `(field: Type)` そのもの**なので `( )`。この区別が「これはレコードだ」を構文で明示します。分解側も同じ括弧に従う（構造体 `S { … }`／バリアント `E.V(…)`）ため、`E.V(` の `(` でレコードを開けていることが読み手に分かります。
- 分解は必ず**型名を先頭に置く**ため、ブロックと曖昧になりません（[制御構造](../03-expressions/11-control-flow.md) 参照）。

> **バリアントのフィールドに修飾子（`val`/`mut val`/`pub`）を書かない**：バリアントのフィールドは **`match` で取り出すしかなく**（`e.V.field = …` のような場所書き換えはできず、変更はバリアントを作り直す）、可視性も列挙型単位（バリアントが見えれば全フィールドが見える）。だから個別の可変性/可視性修飾は意味を持たず、`field: Type` だけ＝**ラベル付きレコード `(field: Type)` と同形**です（だから `( )` で書く）。分解時の束縛 `(val binding)` の `val` は**新しい変数の束縛**を表す別物で、こちらは従来どおりです。

## 標準の Optional / Result

オプショナル値とエラーは特別な構文ではなく、通常の列挙型として定義されます。

```plew
@[All, Eq]
pub enum Optional[T] {
    Some(value: T)
    None
}

@[All, Eq]
pub enum Result[T, E] {
    Ok(value: T)
    Err(error: E)
}
```

```plew
val some = <Optional.Some value=42 />
val none = <Optional.None />

match maybeValue {
    Optional.Some(value: val v) => v
    Optional.None               => 0
}
```

オプショナルチェーン（`?.`）は `Chain` トレイトの実装によって有効になります。nil 合体演算子（`??`）は持たず、フォールバックは `Optional.unwrapOr(fallback:)` メソッド（eager 値／lazy クロージャのオーバーロード）で書きます（[型変換と演算子](../03-expressions/12-operators.md) 参照）。

v1 の `Optional[T]` は通常の generic enum なので、`T` はコピー可能型に限ります。`Optional[File]` のように by-value の `unique` 型を直接入れることはできません。`Ref[File]` / `MutableRef[File]` で包めば `Optional[Ref[File]]` や `Optional[MutableRef[File]]` は書けますが、これは共有 identity を導入する別の意味であり、「optional な唯一所有資源」ではありません。by-value の `Optional[unique]` は [`allowUnique`](06-generics.md) で generic container の move-only payload を明示的に許す設計を入れた後の将来機能です。v1 で禁じる理由は、通常 generic enum に move-only payload を入れると、copy / drop / match move / partial initialization の規則が型引数の能力に依存し、すべての generic container が所有権 IR を要求するためです。

> トレイトもカスタム型の一種ですが、要求・関連型・継承・準拠と `via` の意味論は独立章の[トレイト](08-traits.md)にまとめています。

## 再帰的な値型

構造体・列挙型は**自分自身を含む**ように書けます。木・連結リスト・構文木（AST）など、ありふれた形です。

```plew
struct TreeNode {
    val value: I32
    val left:  Optional[TreeNode]   // 自己参照
    val right: Optional[TreeNode]
}

enum List[T] {
    Cons(head: T, tail: List[T])   // 自己参照
    Nil
}
```

値型がそのまま自身を含むと、素朴にはサイズが無限に発散します（`size(TreeNode) ≥ size(TreeNode) + …`）。Plew では**コンパイラが再帰を検出し、必要なフィールドをヒープ間接で自動的に箱化**してレイアウトを有限化します。これは Array / String の CoW バッファと同じ「ヒープ＋参照カウント＋CoW」機構の再利用で、**ユーザーには見えません**（ヒープ確保はランタイム実装の詳細＝CoW と同じく明示の対象外）。

重要なのは、**書いた通りの型・値意味論がそのまま保たれる**ことです。

- フィールドの型は書いた通り（`left` は `Optional[TreeNode]`）。`node.left` も `Optional[TreeNode]` を返す。間接を表す**ラッパー型は表面に一切現れない**。
- コピーは独立（CoW で遅延）。`val a = tree; val b = a` の後に `b` 側を変更しても `a` には伝播しない。spawn を越え得るヒープセルは allocation 時点から atomic カウントになり、境界でも CoW を維持したまま物理共有できる（[非同期](../04-execution/14-concurrency.md#参照カウント方式の静的選択) 参照）。
- 間接の挿入箇所（相互再帰でどの辺を箱化するか）はコンパイラが選ぶ。正しさは選び方に依らないため、**ユーザーが指定する必要はない**（`indirect` のような修飾語は持たない）。

帰結として、**値意味論の再帰型は循環を作れません**。親へ貼り戻すには共有 identity が要りますが、それは `Ref` / `MutableRef` を書いたときだけ得られるからです。よって自動箱化された再帰型は常に木／DAG どまりで、循環参照・弱参照・リークの問題は生じません。

**共有したいときは明示的に `Ref` / `MutableRef` を書きます**。意図が型に出る形で住み分かります。`MutableRef` は構築・更新に使う共有可変グラフ、`Ref` は読み取り専用ビューとして公開したい共有グラフです。`MutableRef` から `Ref` への変換はできますが、逆向きはできません。したがって作成側が可変ハンドルを外へ出さなければ、利用側は実用上の不変循環グラフとして扱えます。ただし `Ref` は deep freeze ではなく読み取り capability なので、どこかに同じ cell への `MutableRef` が残っていれば、その変更は `Ref` からも観測されます。

```plew
val l: Optional[TreeNode]             = …   // 値の木。コピーで独立
val g: Optional[Ref[TreeNode]]        = …   // 共有読み取りグラフ。コピーでノード共有・循環可（→ WeakRef）
val m: Optional[MutableRef[TreeNode]] = …   // 共有可変グラフ。コピーでノード共有・循環可（→ WeakMutableRef）
```

`MutableRef` は nonsendable であり、それを含む型にも nonsendable が自動伝播して spawn 境界を越えられません。`Ref[T]` は読み取り専用なので、`T` が sendable なら `Ref[T]` も sendable です（[値・変数・所有権](../01-basics/03-values.md) / [非同期](../04-execution/14-concurrency.md)）。素の再帰値型は共有参照を含まないため、木や AST をそのまま別スレッドへ送れます。

> 終端ケース（`Optional` の `None` 等）を持たない再帰（`struct Node { val child: Node }`）も**型としては合法**です。レイアウトは有限になりますが、有限の値を構築する手段がないだけです（Rust の `Box` 版と同じ状況）。これを禁じるのはコンパイラに不要なチェックを課すだけなので、特別扱いしません。

## インスタンス生成

全ての構造体および列挙型バリアントは**JSX ライクな構文でのみ**インスタンス化できます。位置引数による生成（`Color.Red(5)` のような形）はありません。

```plew
struct Person {
    val name: String
    val age: I32
}

// フィールド指定による生成
val person = <Person name="Alice" age=30 />

// 子要素を持つ構造体（UIコンポーネントなど）
struct Container {
    val title: String
    val children: Array[Widget]
}

val container = <Container title="Main">
    <Button text="OK" />
    <Label text="Hello World" />
</Container>
```

**子要素の対応づけ（Plew で最も暗黙的な箇所）**：`<Tag>…子…</Tag>` の子要素は、フィールド名がちょうど **`children`** のフィールドへ渡されます（属性で `children=…` と書くのと等価）。`children` の型は、子要素の並び（イテレータ）から構築できることを宣言するトレイト（暫定 `FromIterator[T]`・名称未決）に準拠していなければなりません。

### 既定 factory（memberwise）とフィールドの既定値

`<Type field=… />` でフィールドをそのまま渡す生成は、コンパイラが各構造体に自動で用意する**既定 factory（memberwise factory）**です（リテラルと同じくコア言語の構築意味論で、derive ではありません）。

**フィールドは宣言時に既定値を持てます**（`val age: I32 = 0`）。これは既定 factory の**デフォルト引数**の糖衣で、意味論は[関数のデフォルト引数](../01-basics/04-functions.md#デフォルト引数)と同一です ── 省略のたびに**呼び出しごとに再評価**（`= []` は毎回フレッシュ）・**定義側モジュールのスコープで評価**・任意の実行時式可。ただし既定値式は **suspend 不可**（`await` 不可）で、評価順は**フィールド宣言順**（＝既定 factory のパラメータ宣言順）です。**他フィールド・`self` は参照できません**（`val area: I32 = width * height` は不可。シグネチャ単体で評価できる自己完結な値だけ。許可方向は後から additive）。既定値を持つフィールドは生成時に省略でき、省略の組み合わせごとに生成セレクタが増えます。

```plew
struct Person {
    pub val name: String
    pub val age: I32 = 0
}

val a = <Person name="Alice" age=30 />   // 全指定
val b = <Person name="Bob" />            // age 省略 → 0
```

**既定 factory の既定可視性はモジュール内**です。factory の宣言を何も書かなくても、同じ定義モジュールの root と part にある自由関数・別型の impl・名前付き拡張から利用できます。非公開を含む全フィールドを指定でき、既定値のあるフィールドは省略できます。モジュール外への公開は暗黙に行いません。

公開したいときは **`pub impl` 内に裸の `factory` を 1 行**書きます。これは全フィールドを引数に取る memberwise factory の明示的な公開です。メンバの公開指定は impl 単位であり、メンバ単体の `pub factory` は書きません。

```plew
pub struct Credentials {
    val token: String
    val label: String = "default"
}

pub impl Credentials {
    factory
}

// 別モジュールで Credentials を import 済み
val credentials = <Credentials token="secret" /> // label は既定値
// credentials.token                            // エラー: 読み取りは定義モジュール内のみ
```

- **引数はフィールドの可視性によらず全フィールド**。非公開フィールドや `pub(get)` フィールドも、生成時の引数として指定できます。生成後の読み取り・書き込み権限は変わりません。
- **全フィールドが `pub` である必要はありません**。非公開フィールドに既定値がある必要もありません。既定値のある引数だけを省略でき、省略時に既定値を評価します。
- **全フィールドの名前と型が構築 API の一部**になります。公開シグネチャは通常の[公開 API 閉包性](../04-execution/15-modules.md#公開-api-閉包性)を満たす必要があり、パッケージ外に公開する factory の引数へ非公開型を漏らしてはいけません。既定値があってもこの条件は免除されません。
- 利用者に指定させたくないフィールドや入力検証がある場合は、裸の公開 factory を宣言せず、カスタム factory で引数と構築処理を定義します。
- 書かなければ既定 factory はモジュール内のままです。カスタム無名 `factory(...)` とのラベル集合衝突は通常の[オーバーロード](07-methods-impl.md)規則で判定します。

### factory

カスタムの生成ロジックは `impl` 内に `factory` で定義します（`impl` は [メソッドと impl](07-methods-impl.md)）。古典的なコンストラクタと違い `self` を初期化するのではなく、**完成したインスタンスを `return` で返します**（＝ファクトリ）。`return` は必須で、キャッシュ済みの値など `<Type … />` 以外を返してもかまいません。

- 無名 `factory(...)` → `<Type … />` で呼び出す。ラベル集合が異なれば複数定義（[オーバーロード](07-methods-impl.md)）できる。
- 名前付き `factory name(...)` → `<Type.name … />` で呼び出す（列挙型バリアント生成と同じ形）。名前は camelCase。
- 属性ラベルは factory の引数ラベル（必ずしもフィールド名と一致しなくてよい）。呼び出し時はラベル必須。**factory ではラベル抑制 `~:`（[関数](../01-basics/04-functions.md#ラベルの抑制) の無ラベル引数）は使えません** ── 生成は JSX でラベル付きであることが生成可視性の根幹だからです。

引数なしや同一ラベル集合の生成は無名では1つしか作れないため、それらは**名前付き factory** にします。これにより `assoc fn` を生成用に流用せずに済み、生成が常に JSX で明示されます。

```plew
struct Celsius {
    val degree: F64
}

impl Celsius {
    // 名前付き factory → <Celsius.fahrenheit … />
    factory fahrenheit(value: F64) {
        return <Celsius degree=((value - 32.0) / 1.8) />
    }

    // 引数なしでも名前を付ければ何個でも定義できる
    factory zero() {
        return <Celsius degree=0.0 />
    }
}

val a = <Celsius degree=20.0 />                      // フィールド指定
val b = <Celsius.fahrenheit value=68.0 />           // 名前付き factory
val z = <Celsius.zero />                             // 引数なし factory
```

factory の戻り型は既定で暗黙の `Self` です（fallible な場合のみ下記の修飾語で `Optional[Self]`／`Result[Self, E]` にできる）。本体での `<Type field=… />`（全フィールド指定）はフィールド初期化（＝ `Self` の生成）を指し、自分自身を再帰呼び出ししません。この `Self` 規約により、[`newtype`](10-newtype.md) は元の型の factory をそのまま継承して JSX 構文で生成できます。

#### ファクトリ名の指針（規約）

ファクトリ名は、暗黙の「生成（create）」を**修飾する語**です。`<Color.red />` は「**赤い** Color を生成」、無名 `<Struct />` は「Struct を生成」（修飾語なし）と読めます。だからファクトリ名は **結果や生成のしかたを表す形容詞・名詞**であるべきで、**単独の動詞は避けます**（動詞は別の主語のメソッド `x.convert()` を連想させ、生成として読めません ── 「create を修飾する語」になりません）。呼び出し側でも `<Type.name … />` の形から factory だと一目で分かります。

- ○ 形容詞・名詞：`<Color.red />`・`<Date.now />`・`<I32.checked source=x />`（「**検証済みの** I32 を x から生成」）。
- ✗ 単独動詞 `convert`（source 側メソッド `x.convert()` を連想）・目的語のない動詞 `try`（何を try するか不明）。

**名前とラベルで重複しない。** ラベルがオペランドを、型が結果を既に運ぶので、ファクトリ名は **残り（生成の手段・種別）だけ** を運びます。たとえば変換元を表すラベルが `source`（＝「from」を含意）なら、名前に重ねて「from」を書きません（`checked source=x` ○ ／ `checkedFrom source=x`・`tryFrom source=x` は "from" の重複で ✗）。

**簡潔に。名前とラベルが合わさって「何を生成するか」が過不足なく伝わること** ── これがファクトリ名の良し悪しの基準です。

### 失敗し得るファクトリ（fallible factory）

生成は失敗し得ます（範囲外の数値・不正な入力など）。Plew に例外（throw/catch）は無いので、**失敗を表す lang item で包んで返すファクトリ**を、factory 宣言への**前置修飾語**で表します（戻り型は手書きしません）。

| 修飾語 | 戻り | 意味 |
| --- | --- | --- |
| （なし） | `Self` | 必ず成功する全域ファクトリ |
| `optional` | `Optional[Self]` | 値ができる／できない（理由は運ばない） |
| `result[E]` | `Result[Self, E]` | 成功 or エラー `E` |

包めるのは `Optional`／`Result` の **2 つだけ**（言語の可謬 lang item）。任意のラッパー（`Array[Self]`・`Promise[Self]` 等）は対象外で、`<Type.name … />` は常に「**1 個の Self を作る試み**」として読めます。修飾語は能力語 + 宣言子（`move fn`・`async fn`）と同じく前置で、`result` の `[E]` がエラー型を運びます。

```plew
struct Temperature {
    pub val celsius: F64
}

impl Temperature {
    // 全域（従来どおり）
    factory zero() {
        return <Temperature celsius=0.0 />
    }

    // Optional[Self]：パースできなければ None
    optional factory parse(text: String) {
        guard Optional.Some(value: val c) = parseF64(text: text) {
            return <Optional.None />
        }
        return <Optional.Some value=<Temperature celsius=c /> />
    }

    // Result[Self, E]：物理的にあり得ない値はエラー
    result[RangeError] factory kelvin(value: F64) {
        guard value >= 0.0 { return <Result.Err error=<RangeError /> /> }
        return <Result.Ok value=<Temperature celsius=(value - 273.15) /> />
    }
}

val z = <Temperature.zero />                       // Temperature
val a = <Temperature.parse text="20.0" />          // Optional[Temperature]
val b = <Temperature.kelvin value=300.0 />        // Result[Temperature, RangeError]
val c = try <Temperature.kelvin value=300.0 />    // try と合成（前置 try が JSX 全体に掛かる）
```

- **自動ラップはしない**（明示 > 暗黙）。本体は `Optional.Some`/`None`・`Result.Ok`/`Err` を **JSX で明示的に返す**。成功路の内側 `Self` も `<Type … />` で組むので、**構築点が二段とも可視**になる（JSX の目的どおり）。
- **呼び出しは通常の JSX `<Type.name … />`**。返るのは宣言した `Optional[Self]`／`Result[Self, E]`。`as` のような糖衣は持たない（→ [型変換と演算子](../03-expressions/12-operators.md) の `TryFrom`）。
- **`optional`／`result` は文脈依存キーワード**（factory 修飾位置でのみ予約）。`val result = …` のような**ごく普通の識別子としての `result`/`optional` は従来どおり使えます**（予約語にすると `result` という頻出変数名を潰すため → [概要](../01-basics/01-overview.md#キーワード)）。
- **newtype 継承**：fallible factory も継承され、`Self` 置換がラッパー内に効く（`Result[Self, E]` → `Result[Brand, E]`。→ [newtype](10-newtype.md)）。

### 列挙型バリアントの生成

列挙型のバリアントも同じ構文で生成します。`Enum.Variant` を型として指定し、フィールドを持たないバリアントは要素なしで生成します。

```plew
val some = <Optional.Some value=42 />
val none = <Optional.None />
val ok   = <Result.Ok value=data />
val err  = <Result.Err error=parseError />
```

enum にも camelCase の名前付き factory を定義できます（PascalCase のバリアント名と衝突しません）。

### 型を文脈から省く（leading-dot 構築）

構築する型が**文脈から一意に定まる**ときは、`<Type.Member …/>`・`<Type.member …/>` の **`.` の前の型名だけを省いて** `<.Member …/>`・`<.member …/>` と書けます（列挙型バリアント・名前付き factory）。

```plew
val a: A = <.foo n=1 />                     // <A.foo n=1 /> の省略（名前付き factory）
val s: Optional[I32] = <.Some value=42 />   // <Optional.Some value=42 /> の省略
val n: Optional[I32] = <.None />            // ペイロードなしバリアントも同様
```

- **leading-dot は「既にある `.` の前の型を落とす」操作**であって、`.` を新たに足すものではありません。**memberwise（`<Type field=…/>`）には元々 `.` が無い**ので、`<. field=…/>` は省略形ではなく `.` の捏造になります ── だから書けません（型名を残す）。
- **marker は `<.Name`**。式が `.` で始まるのは**常に `<.` に続けてメンバ名がある形に限られ**、裸の `.foo` が浮くことはありません。これにより「型省略の構築」だと構文で一意に分かり、Swift/Zig が必要とする「期待型まで名前解決を遅延する」曖昧さが出ません。出どころ（型名）は左辺の注釈や受け側のシグネチャに**ローカルに明記**されているので ambient ではなく、provenance は回収可能です。
- **使えるのは期待型が一意に定まる位置だけ**：明示注釈（`val a: A = …`）・宣言済みの戻り型（`return …`）・型が単一に決まる引数位置。**leading-dot 式そのものを使って期待型を決めることはしません**（鶏卵を避ける）。
- **型オーバーロードでは使えない**：同一セレクタで引数型だけ違う候補がある引数位置は期待型が割れるので `<.…/>` 不可 ── `<A.foo …/>` のように型を明示する。**ラベルオーバーロードは可**：型は一意のままラベルが候補を絞るので `<.…/>` で書ける。
- **パターンには波及しない**：[パターン](../03-expressions/11-control-flow.md#パターンマッチング)のバリアントは `Enum.Variant(…)`（JSX ではない）で書き、leading-dot の対象外です。`.` が `<.` 以外で始まらない不変条件を保つため、パターン側は型名を省きません。

## メンバの可視性

可視性は **モジュール内 / 公開の二段階**です。型だけに閉じる private や、パッケージ内公開の中間段は持ちません。モジュールは定義 root と推移的な part 全体です。

- **修飾なしのフィールドと `impl`**：定義モジュール内から利用可能。同じモジュールの自由関数・別型の impl・名前付き拡張も利用できます。
- **`pub` フィールドと `pub impl`**：その型に到達できる別モジュールからも利用可能。
- **`pub(get) mut val`**：読み取りは公開、書き込みは定義モジュール内のみ。既定 factory の明示公開による構築時の値指定とは独立です。
- **`pub mut val`**：読み書きとも公開。ただし書き込みには通常の可変性・借用規則を満たす必要があります。
- **`val`**：公開範囲によらず構築後の再代入は不可。`pub(get) val` は不可で、読み取りを公開するなら `pub val` を使います。

メソッド・関連関数・関連値・factory の公開指定は **impl ブロック単位**です。`pub impl` は全メンバを公開し、修飾なし `impl` は全メンバをモジュール内にします。個別の `pub fn` / `pub factory` は書けません。公開と内部用は別の impl ブロックに分けます。

型そのものも `pub` で公開します。型が非公開なら `pub` メンバを宣言しても、その型を外部に公開することにはなりません。公開する型・メンバの両方が見えるときに外部からアクセスできます。パッケージ外の到達には、さらに[ルート公開面](../04-execution/15-modules.md#公開面と外部到達)に型が載っている必要があります。

```plew
pub struct Account {
    pub val id: I32
    pub mut val nickname: String
    pub(get) mut val balance: I32
    mut val secretKey: String
}

impl Account {
    fn helper() -> I32 { return self.balance + 1 }
}

// 同じモジュールの自由関数も内部状態を扱える
fn replaceKey(account: inout Account, key: String) {
    account.secretKey = key
}

extension Audit {
    impl Account {
        fn key() -> String { return self.secretKey } // 同じ定義モジュールなので可
    }
}
```

別モジュールで定義した拡張からは、その型の非公開メンバにはアクセスできません。同じパッケージ・ディレクトリにあること、型を import / 再公開したこと、view の切替えでは権限は増えません。可視性はアクセスするコードと宣言の定義モジュール identity で判定し、generic の実体化先や呼出し元へ置き換えません。型の不変条件を共同で守る責任は定義モジュール全体にあります。

### トレイト準拠の可視性

トレイト準拠 `impl Type as Trait { … }` も同じくブロック単位で、**`pub impl Type as Trait { … }` が公開準拠**、修飾なし `impl Type as Trait { … }` が**内部準拠**です。Plew はトレイトメソッドを**レシーバ型だけで解決**し、呼び出し側にトレイトを `import` させません（Rust と違い「トレイトがスコープに無い＝メソッドも呼べない」が自然には成立しない → [モジュール § 言語アイテム](../04-execution/15-modules.md#言語アイテムは常にスコープにあるimport-不要)）。そのため**準拠メソッドを隠す軸はメンバ可視性（`pub impl` か否か）**になります：

- **`pub impl Type as Trait`（公開準拠）** — `Type` が見える所どこでも、その準拠が与えるメソッドを（トレイト名を書かずに）呼べる。
- **`impl Type as Trait`（内部準拠）** — 準拠という**事実**はモジュール内で成立し `where T: Trait`／`any Trait` に使えるが、**準拠が与えるメソッドを外部から呼ぶと可視性エラー**。内部だけで使う準拠（内部用ヘルパトレイトへの準拠など）はこちらで閉じる。

外部到達は「型の [`pub`](../04-execution/15-modules.md#公開と再公開)」と「`pub impl`」の組で決まります（型が `pub` されていなければ、`pub impl` でもそもそも外から型に届きません）。準拠は coherence の大域事実なので「準拠そのものの隠蔽」はせず、隠すのは**メソッドアクセス**です。
