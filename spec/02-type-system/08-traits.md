# トレイト

トレイトは、型が満たすべき**要求**（メソッド・フィールド・関連値・関連型・factory）を束ねる抽象です。トレイト本体には**要求だけ**を書きます。要求の上に組む**提供メソッド**（`Iterator` の `map`/`filter` を `next` の上に作る類）は、トレイトと同じモジュールの**ベアな `impl Trait { … }`** に書きます（型の inherent `impl Type { }` と対称）。型への準拠 `impl Type as Trait` を書けば、その型は**提供メソッドを自動でベア表面に得**、境界 `where T: Trait` 越しにも使えます。準拠は `impl Type as Trait` で**明示的に宣言**し、各要求を `via`／本体で witness します（暗黙の準拠は起きません）。トレイトは[型引数](#トレイト名の-型引数と関連型束縛)（多重 conformance）と[関連型](#関連型associated-type)（出力）を持てます。

トレイトの宣言は型システムの[カスタム型](05-structs-enums.md)の一種で、`export trait Shape { … }` のように書きます。本章では定義・関連型・継承・準拠の意味論、トレイトを値の型として使う**存在型**（`any`）、そして標準で提供されるトレイトのカタログをまとめます。

## トレイトの定義（要求と提供メソッド）

トレイト本体には型が満たすべき**要求**だけを書きます（すべて本体なし）。フィールド要求・メソッド要求・関連値要求・関連型・**factory 要求**があり、各型は準拠時に `via`／本体で必ず witness します（後述「トレイト準拠と via」）。

factory 要求は、トレイトが「**この型を構築する手段**」を求めるものです（`From` の変換・`Chain` の再構築など）。本体なしの factory シグネチャ（[fallible factory](05-structs-enums.md#失敗し得るファクトリfallible-factory) も可）を書き、準拠側 impl が factory 本体を定義して witness します。生成が常に JSX `<Type.name … />`／factory で起きる原則を、トレイト要求でも保ちます。

```plew
trait Default {
    factory default()                 // factory 要求（全域）
}
impl Temperature as Default {
    factory default() { return <Temperature celsius=0.0 /> }  // 本体で witness
}
```

要求の上に組む**提供メソッド**（要求だけを使って組み立てる共有メソッド。`Iterator` の `map`/`filter` を `next` の上に作る類）は、**トレイトと同じモジュールのベアな `impl Trait { … }`** に書きます。トレイト本体には要求（本体なし）、`impl Trait` には提供メソッド（本体あり）── 型の `struct`（フィールド宣言）と `impl Type`（メソッド本体）と同じ住み分けです。

```plew
trait Stepper {
    fn step() -> I32                 // 要求：本体なし。各型が実装する
}

impl Stepper {                       // 提供メソッド（トレイトと同じモジュール・ベア）
    fn doubleStep() -> I32 {         // self: Self: Stepper。要求 step() の上に組む
        return self.step() + self.step()
    }
}

struct Counter {
    mut val n: I32
}
impl Counter as Stepper {
    fn step() -> I32 { return self.n }   // 要求を witness するだけ
}
// counter.step() も counter.doubleStep() も呼べる。
// fn f[T](t: T) where T: Stepper の本体でも t.doubleStep() がベアで呼べる。
```

- **`impl Trait` の `self` は準拠型**（`Self: Trait`）。提供メソッドは `self` 経由で要求や同じ `impl Trait` 内の他メソッドを呼べる。本体はトレイトのインターフェースに対して**一度だけ**型検査される。
- **ベアな `impl Trait` を書けるのはトレイトを所有するモジュールだけ**（型の inherent `impl Type` と同じ「所有者のみ」コヒーレンス）。第三者がトレイトに提供メソッドを足す（`itertools` 的）のは**名前付き拡張** `extension X { impl Trait { … } }` で行い、使用箇所で `#X` を明示して opt-in する（→ [拡張](09-extensions.md)）。
- **トレイト間準拠 `impl B as A` も同じ**：`B` の全準拠型を別トレイト `A` へ準拠させる宣言は、`A`／`B` を所有するモジュール内なら**ベアに書け**（準拠は対象型のベア表面に自動随伴）、第三者は `extension` 経由で `#Ext` opt-in する。複数経路で同じトレイトに届く衝突は `#` で経路を選ぶ（→ [拡張：トレイト間準拠の衝突](09-extensions.md#トレイト間準拠の衝突は-でパスを選ぶ)）。頭なし型変数 `impl[T] T as A where T: B` は禁止（錨が無く列挙不能）で、主語トレイト `B` を錨にした `impl B as A` で書く。
- **準拠で自動随伴**：`impl A as Trait` を書けば A は提供メソッドをベア表面に得、`where T: Trait` 越しにも使える。出どころは「明示した準拠／境界のトレイト」なので provenance は追える（ambient ではない）。
- **上書き不可**：準拠側 `impl Type as Trait`（`as` あり）が witness するのは**要求だけ**。提供メソッドに別の本体を与える（上書き）ことはできない。挙動を変えたい型は名前付き `extension` に別実装を置き `#Ext` で選ぶ（Swift の protocol extension に伴う静的／動的ディスパッチの食い違いを避けるため）。

### 提供メソッドの衝突（曖昧はエラー）

トレイト `P` と `Q` が同じ `foo()` を提供し、型 A が**両方に準拠**すると、A のベアに同名 `foo` が 2 つ載ります。**準拠の併存は許されます** ── ベアの `a.foo()` だけが**曖昧でエラー**になり、`a#P.foo()`／`a#Q.foo()` で源を選びます（[メソッド源の選択](09-extensions.md#メソッド源の選択ap)・generic の `where T: P + Q` と完全に同じ）。

```plew
trait P {}
impl P { fn foo() {} }
trait Q {}
impl Q { fn foo() {} }

struct A {}
impl A as P {}
impl A as Q {}        // OK：併存できる

a.foo()               // エラー：P.foo と Q.foo で曖昧
a#P.foo()             // P の foo
a#Q.foo()             // Q の foo
```

> 「同名同シグのベア宣言不可」は**同一の出どころで二重定義する**ことを禁じる規則で、別トレイト由来の 2 つが鉢合わせるのは*曖昧な呼び出し*の方をエラーにします（**曖昧はエラーを use site で適用**＝`any P` のメンバ診断と同じ）。要求（本体なし）が同名同シグなら衝突しません（1 witness で両方満たす＝define-once・後述）。

**ベアの `a.foo()` を使えるようにしたい（既定を 1 つ選ぶ）なら**、片方を privileged にします（いずれも任意・強制ではない）：

```plew
// (1) 片方の準拠を拡張に閉じる → もう片方がベア既定
impl A as P {}
extension AWithQ { impl A as Q {} }
// a.foo() → P・a#AWithQ.foo() → Q

// (2) inherent メソッドでベアを奪う
impl A { fn foo() { … } }
// a.foo() → inherent・a#P.foo()/a#Q.foo() で各トレイト版
```

### 関連型（associated type）

トレイトは**関連型**を要求できます。`type Name` で宣言し、シグネチャ中ではベア名で参照します。準拠側の impl が具体型に束ねます（`type Name = ConcreteType`）。

```plew
trait Iterator {
    type Item                       // 関連型の要求
    fn next() -> Optional[Item]     // シグネチャ中はベアで参照
}

impl Counter as Iterator {
    type Item = I32                 // impl が具体型に束ねる
    fn next() -> Optional[I32] { /* ... */ }
}
```

- **型引数（多重 conformance）と関連型（出力）の使い分け**: 入力＝呼ぶ側が選び 1 型に複数 conformance あり得るものは**型引数**にする（`Add[Rhs]`：`Vec` は `Add[Vec]` と `Add[F64]` の両方に準拠できる）。出力＝impl が一意に決めるものは**関連型**（`type Output`・`Iterator.Item`）。1 つのトレイトが両方を持ってよい（`Add[Rhs] { type Output }`）。複数 conformance のメソッドは引数型で区別される**オーバーロード**になる（→ [メソッドのオーバーロード](07-methods-impl.md)、[型変換と演算子](../03-expressions/12-operators.md)）。
- 関連型は**型の名前空間**に属し、メソッド・フィールドのオーバーロード集合とは別。関連型名がメソッド名と衝突することはない。`type Name = …` の充足に `via` は使わない（直接束ねる）。
- **制約を付けられる**: `type Item: Format` のように要求側で制約を課せる。impl はその制約を満たす型で束ねる。ベア `type Item` は制約なし。
- **外部からの射影は `T.Item`**: 型変数経由で関連型を名指すときは `.` で射影する（値のメンバアクセスと同じ区切り）。

```plew
fn first[T](it: T) -> Optional[T.Item] where T: Iterator {   // T.Item で射影
    return it.next()
}
```

#### トレイト名の `[...]`（型引数と関連型束縛）

トレイト名のあとの `[...]` には、**位置型引数**（`Add[Vec]`）と**関連型束縛**（`Iterator[Item = Foo]`）を書けます（Rust の `Trait<Arg, Assoc = T>` と同形で、記号だけ `[]`）。両者は混在可。supertrait や `where` 句のトレイト制約で使えます。

```plew
fn sum[T](it: T) -> I32 where T: Iterator[Item = I32] { /* ... */ }   // 関連型を束縛
fn g[T](x: T) where T: Add[Vec] { /* ... */ }                          // 位置型引数で右オペランドを指定
```

### トレイトの継承（supertrait）

`trait Sub: Super` は、`Sub` への準拠に `Super` への準拠を要求します（`where T: Trait` と同じく `:` は制約）。複数指定は **`+`** で連結します（`trait Sub: A + B`）。`where T: A + B` と同じく、`+` は**同一の主語（ここでは `Self`）に制約を重ねる**意味で、別々の主語を並べる `,` とは役割が違います。関連型の束縛も同じ記法で書けます（`trait FooIterator: Iterator[Item = Foo] + Format {}`）。

```plew
trait BoundedStepper: Stepper {
    fn limit() -> I32                    // 要求
}

impl BoundedStepper {                    // 提供メソッド：親 Stepper の step() を使える
    fn cappedStep() -> I32 {
        val s = self.step()
        if s > self.limit() { return self.limit() }
        return s
    }
}
```

- **継承は制約であって自動実装ではない**: `impl Type as BoundedStepper` を書くには、別途 `impl Type as Stepper` も存在しなければならない（無ければエラー）。親準拠が暗黙に生成されることはない（暗黙準拠を持たない方針の一貫）。
- `Sub` の提供メソッド（`impl Sub`）は、`self` 経由で `Super` の要求を呼べる（継承で準拠が保証されるため）。
- ダイヤモンド継承で同名の**要求**が複数経路から来ても、同セレクタ・同シグネチャならオーバーロード集合の 1 つへ畳まれる（シグネチャ不一致はオーバーロードとして共存、または衝突ならエラー）。**提供メソッド**は準拠で随伴するので、別経路由来の同名・同シグネチャ提供メソッドが衝突するのは「型が両トレイトに準拠したとき」で、**呼び出し時に曖昧**になる（[提供メソッドの衝突](#提供メソッドの衝突曖昧はエラー)と同じく、`self#P.foo()` 等で源を選ぶ・ベア既定が要れば片方を拡張に閉じる）。トレイト定義そのものが衝突を強制することはない。

## トレイト準拠と via

型のメソッド・フィールドはセレクタ（名前＋ラベル）ごとの**オーバーロード集合**を成します（→ [メソッドのオーバーロード](07-methods-impl.md)）。同一シグネチャで別物を作ることはできません。トレイトへの準拠は `impl Type as Trait` で**明示的に宣言**し、その中で各要求を **`via` で実体メンバに束ねる**か、メソッドならその場で本体を定義します。**準拠は暗黙には成立しません** — 名前が一致するだけのメソッドが勝手に witness になる「うっかり準拠」は起きません。

```plew
trait Drawable {
    val width: I32         // フィールド要求
    fn draw() -> String    // メソッド要求
}

struct Sprite {
    val w: I32
}

impl Sprite {
    fn render() -> String { return "..." }  // 実体メソッド
}

impl Sprite as Drawable {
    val width: I32 via w               // 要求 width（型つき）を実体 w に束ねる
    fn draw() -> String via render     // 要求 draw（完全シグネチャ）を実体 render に束ねる
}
```

`via` の左辺は**常に要求の完全なシグネチャ**（メソッドは引数ラベル・型・戻り型、フィールドは型）を書きます。本体定義形（`fn draw() -> String { … }`）と対称で、トレイトを見なくても各行が何を満たすか分かり、同名要求が複数あっても一意に特定できます。

実体に対応するメソッドが無い要求は、impl 内でそのまま本体を定義します（新しい実体メソッドになります）。

```plew
impl Sprite as Drawable {
    val width: I32 via w
    fn draw() -> String { return "drawn" }  // 実体が無いのでここで定義（via 形と対称）
}
```

同名の要求が複数ある（要求がオーバーロードされている）トレイトも、左辺のシグネチャで一意に witness できます。

### 準拠の可視性（`pub impl`）

準拠も他の `impl` と同じく**ブロック単位の可視性**を持ちます（→ [メンバの可視性](05-structs-enums.md#トレイト準拠の可視性)）。**`pub impl Type as Trait { … }` は公開準拠**で、`Type` が見える所どこでもその準拠が与えるメソッドを（トレイト名を書かずに）呼べます。修飾なし `impl Type as Trait { … }` は**内部準拠**で、準拠の事実はモジュール内で成立し `where T: Trait`／`any Trait` に使えますが、外部からメソッドを呼ぶと可視性エラーになります（内部用ヘルパトレイトへの準拠を閉じるのに使う）。Plew はトレイトメソッドをレシーバ型だけで解決し import を要さないため、Rust の「トレイトがスコープに無ければ呼べない」が自然には効かず、**隠す軸はこの `pub impl` か否か**になります。`export impl` は無く、外部到達は「型の `export`」＋「`pub impl`」で決まります（coherence の大域事実である準拠そのものは隠さず、隠すのはメソッドアクセス）。

```plew
trait Writer {
    fn write(data: Bytes) -> Result[I32, Error]    // 同名・別シグネチャの要求
    fn write(data: String) -> Result[I32, Error]
}

impl File as Writer {
    fn write(data: Bytes) -> Result[I32, Error] via writeBytes
    fn write(data: String) -> Result[I32, Error] via writeText
}
```

### ルール

- **全要求を明示的に束ねる**: フィールド・メソッドを問わず、各要求は `via` で実体メンバを指すか、メソッドなら impl 内で本体を定義する。**`via` 左辺は常に完全なシグネチャ**（名前一致でも `val width: I32 via width` のように書き、省略しない）。空の `impl Type as Trait {}` は準拠失敗。
- **define-once**: 実体メソッドの**本体定義は一箇所のみ**（inherent な `impl Type {}` か、いずれか 1 つの trait impl）。他からは `via` で参照する。フィールドはストレージなので struct 宣言で定義し、trait impl からは `via` で束ねるだけ（trait impl はフィールドを新設しない）。
- **別名はベアでも呼べる**: `fn draw() -> String via render` のように要求名と実体名が違う場合、要求名（`draw`）も**ベアの呼び名として加わる**。`sprite.draw()` も `sprite.render()` も同じメソッドを指す（トレイト経由で呼べてベアで呼べないのは不自然、という方針）。別名が同セレクタ・同シグネチャの既存メンバと衝突すればコンパイルエラー（引数が違えばオーバーロードとして共存）。
- **シグネチャ一致**: `via` 先のシグネチャは要求と一致しなければならない。合わなければエラー（改名するか、`#Ext` 拡張に分離）。
- **1 メンバで複数トレイトを witness 可**: 同じ実体メソッドを複数の trait impl から `via` で束ねてよい。シグネチャが噛み合う限り、1 つの `Foo.bar` が複数トレイトの要求を同時に満たせる。
- **同名・同シグネチャで別挙動が必要なら拡張**: 引数型が違えばオーバーロードとして同居できるが、**同じシグネチャ**で別実装は作れない。本当に必要なら名前付き `extension` に置き、呼び出し位置の `#Ext` で明示する（[拡張](09-extensions.md)）。

## トレイトを値の型として使う（存在型 `any`）

トレイト名はそのままでは値の型になりません。**`any P` と書いたときだけ**、「P に準拠する*任意の*具体型」を型消去して持つ**存在型**になります。要素ごとに別の具体型を入れられる（**異種混在**）ので、「準拠さえしていれば何でも入る」コレクション（React 風のツリーなど）を素直に書けます。

```plew
trait Drawable {
    fn draw() -> String
}

// 別々の具体型を 1 つの配列に混在できる
val shapes: Array[any Drawable] = [circle, square, label]
for val s in shapes {
    print(s.draw())                 // 動的ディスパッチ
}
```

`any P` は**第一級の型**で、型を書ける場所すべて（変数・フィールド・引数・戻り値・ジェネリック引数 `Array[any P]`・関連型束縛・[newtype](10-newtype.md) の underlying）に置けます。戻り値にも使えるので、具体型を隠す opaque な戻り（Swift の `some`）は別に設けず、存在型 `any` に一本化します。

### 存在型の sendability

無印の **`any P` は常に nonsendable** です。実行時にたまたま sendable な具体値だけが入っていても、型消去時に sendability 保証を保持していないため、whole-program の流入集合から保証を暗黙に回復しません。別スレッドへ送れる存在型は **`any sendable P`** と明示します。

```plew
val local: any Drawable = circle
val transferable: any sendable Drawable = circle // Circle が sendable なら OK

spawn { render(local) }        // Error：any Drawable は nonsendable
spawn { render(transferable) } // OK
```

`any sendable P` へ具体値を注入するとき、その具体型が sendable でなければコンパイルエラーです。保証の変換は [`sendable fn` と通常の `fn`](../01-basics/04-functions.md#sendable-クロージャ) と同じ向きに限定します。

- `any sendable P` → `any P` は、保証を捨てるだけなので暗黙変換できる。
- `any P` → `any sendable P` は不可。中身が実際には sendable でも、消去済みの保証を後から回復しない。
- `any sendable P` は `[sendable T]` を満たすが、`any P` は満たさない。
- `any P` をフィールド/payloadに持つ外側は nonsendable。`any sendable P` は sendable な構成型として通常の構造的導出に参加する。

`any sendable P` が保証するのは、型消去された具体値と existential storage の**移送可能性**です。動的に選ばれる witness が可変／nonsendable なトップレベル状態へ到達しないかは別軸であり、[`spawn` の実行依存グラフ](../04-execution/14-concurrency.md#spawn-からのトップレベルアクセス)が候補 witness の effect summary まで推移検査します。`Self` を返すメンバを `any sendable P` 越しに呼んだ場合、結果も保証を保って `any sendable P` へ再消去されます。

### 存在型とジェネリックは別物

同じ「P に縛る」でも、ジェネリック境界と存在型は能力が違います（横着すると後者を書きがちですが、取り違えないこと）。

| | `fn f[T](x: T) where T: P` | `fn f(x: any P)` |
| --- | --- | --- |
| 要素の型 | 全部同じ具体型 `T`（**同種**） | 各値が別の具体型でよい（**異種**） |
| ディスパッチ | 静的（単相化） | 動的 |
| 使える要求 | **すべて**（`Eq`/`Ord`/`From` も） | 制限あり（下記） |

「1 種類で十分・最大限に操作したい」ならジェネリック、「異種を混ぜたい」なら `any` を選びます。

### 形成の条件：型引数・関連型をすべて束縛する

`any P` を作るには、P の**型引数と関連型をすべて指定**しなければなりません（未指定は禁止）。

```plew
any Iterator                        // Error：関連型 Item が未束縛
any Iterator[Item = I32]            // OK
fn f[T](i: any Iterator[Item = T])  // OK（T は呼び出しで具体化される）
any Add                             // Error：型引数 Rhs・関連型 Output が未指定
any Add[I32, Output = I32]          // OK
```

この縛りにより、`any P` の中で**まだ消去されているのは `Self` だけ**になります（関連型は具体型に確定済み）。次の呼び出し規則がこの一点だけで決まります。

### 呼べるメンバ・呼べないメンバ（使用箇所で診断）

`any P` という**型を作ること自体は常に valid** です。「このトレイトは存在型にできない」という型レベルの一律禁止は持ちません。代わりに**個々のメンバが呼べるかを使用箇所で判定**します。直感は「自分自身への操作・結果を返す操作は OK、自分と同じ未知の型の値を受け取る操作は NG」。

- **呼べる**：`self` を受け取り、`Self` が**出力位置にしか出ない**メンバ。出力の `Self` はレシーバと同じ保証の存在型へ再消去される（`any P` 越しなら `any P`、`any sendable P` 越しなら `any sendable P`）。関連型は具体型に確定しているので入力に現れても問題ない（`any Add[I32, Output = I32]` で `add(lhs~: any, rhs~: I32)` は呼べる）。
- **呼べない**：
  - `Self` が**非レシーバの入力位置**に現れるメンバ（`Eq` の `eq(lhs~: Self, rhs~: Self)` の `rhs`）。2 つの `any Eq` の具体型が一致する保証がないため、**その呼び出し行で**エラー。
  - **関連関数・factory**（`self` を受けない＝witness を運ぶレシーバが無い。`From.from` など）。

```plew
fn f(a: any Eq, b: any Eq) {        // 型として持つ・配列に入れる・渡すのは自由
    a == b                          // Error：Self 入力。型ではなくこの行で弾く
}
```

全要求が呼べない `any P`（`any Eq` 単独など）も型としては valid ですが実質使い道が無いので、lint で気づかせます（ハードエラーにはしません）。

> **実装は最終フェーズ。** 存在型は動的ディスパッチ機構（witness を実行時に持ち回る）を要するため、コンパイラの他部分が固まってから実装します。ただし**言語仕様としては確定**です（Plew = *Programming Language for Everyday Wizard*。トレイトの配列を Swift のように素直に書けることを優先する）。

### ダウンキャストは無い

`any P` から元の具体型や別トレイトへ取り出す**ダウンキャストは提供しません**（Rust と同じ姿勢。Swift の `as?`・Go の type assertion に当たるものは無い）。`any P` は「P のインターフェースだけを使う」という意図を表す道具で、具体型を回収したくなったらその意図に反しています。**「どれか」を知りたい閉じた集合は[列挙型](05-structs-enums.md)＋網羅 `match`** が担い、`any P` は**開いた集合をインターフェース越しにだけ扱う**ために分けてあります。

設計上の理由：

- **開いた存在型は網羅できない**。任意の具体型が後から準拠し得るので、型での分岐は必ず `_` フォールバックを伴う可謬チェックにしかならず、「match は網羅必須」という Plew の原則と噛み合わない。
- **`as` は infallible 固定**（[型変換](../03-expressions/12-operators.md)）で可謬なダウンキャストを載せられない。`TryFrom` は値変換であって型同一性クエリではない（別カテゴリ）。
- ダウンキャストの定番動機＝エラー検査は、Plew では[エラーが値型 enum](../03-expressions/13-error-handling.md) で集約 From 済みのため**既に具体 `match` で見る**。最大の使い所が存在型側に無い。

> 欲しくなったら設計の誤りを疑う、という立場を既定にします。将来 additive で入れるとしても、`Any` 風の opt-in 定型ではなく `match` の型パターン（`_` 強制で開放性に正直・具体型のみ・`any P → any Q` のグローバル準拠探索は採らない）を第一候補とします。

## 標準トレイト

言語・コアライブラリが提供する主要トレイトです。**演算子・変換・Optional 系トレイトの意味論（シグネチャ・脱糖・型引数の有無・NaN の扱い）の正典は [型変換と演算子](../03-expressions/12-operators.md)** にあり、ここでは所在の一覧だけを示します。型引数を持つか関連型にするかの判断基準は[上記の使い分け](#関連型associated-type)（入力＝型引数で多重 conformance／出力＝関連型）に従います。

### 基本トレイト

演算子に対応しない純粋なトレイトで、本章が正典です。

> **`Clone` トレイトは持ちません**：値意味論（CoW）では代入・受け渡しが独立コピー（`mut val b = a`）なので明示的な複製は不要です。共有が要るときだけ [`Ref`](../01-basics/03-values.md#ref--weakref共有可変)（コピーで共有）→ [値・変数・所有権](../01-basics/03-values.md)。

- `Hash: Eq`: ハッシュ値計算。`Dictionary`/`Set` のキー境界で、衝突解決に等価比較が要るため `Eq` をスーパートレイトに持つ（`Ord: Eq` と同じ形）。算法は Rust 流の `Hasher` ストリーミング方式に倒すが、正確なシグネチャはコアライブラリ設計時に確定（`@[Hash]` で導出可）
- `Format`: 表示・変数展開用フォーマット（→ [文字列](../01-basics/02-basic-types.md)）

### 演算子・変換・Optional 系トレイト

各演算子は対応トレイトのメソッドの糖衣です。詳細な意味論は [型変換と演算子](../03-expressions/12-operators.md) 章を参照（一覧のみ）。

| 種別 | トレイト | 演算子・用途 |
| --- | --- | --- |
| 変換（全域） | `From[Source]` | `x as T`（無名 factory `(from:)`・必ず成功。`try` のエラー変換も。newtype⇔元型の `as` だけは From でなくゼロコスト再タグ） |
| 変換（可謬） | `TryFrom[Source]` | `<T.checked source=… />`（fallible factory `checked`・関連型 `Error`・`Result[Self, Error]`。`as` 糖衣なし） |
| 算術 | `Add[Rhs]` / `Sub[Rhs]` / `Mul[Rhs]` / `Div[Rhs]` | `+ - * /`・結果型は `type Output` |
| 等価 | `Eq`（型引数なし・右辺 `Self`） | `== !=` |
| 順序 | `Ord: Eq`（型引数なし） | `< <= > >=` |
| 単項 | `Not` | 前置 `!`（eager） |
| 添字 | `Index[Key]` | `collection[key]` |
| Optional チェーン | `Chain` | `?.` |

> **`&&` / `||` はトレイトではありません** — Bool 限定・オーバーロード不可の、短絡する制御フロー（`if` の糖衣）です。`??`（nil 合体）は演算子として持たず（右オペランドの暗黙遅延を避けるため）、`Optional.unwrapOr(fallback:)` メソッドで代替します（→ [論理結合子・Nil 合体演算子は持たない](../03-expressions/12-operators.md)）。
