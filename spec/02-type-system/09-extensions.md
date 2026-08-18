# 拡張システム

Plew の特徴的な機能として、値に対して動的に拡張を適用できるシステムがあります。拡張は値自体を変更するのではなく、その値の型に対して一時的に機能を追加する仕組みです。

## 拡張で追加できる機能

拡張では以下の要素を追加できます：

- **メソッド** - インスタンスメソッドや可変メソッド
- **関連値** - 関連定数や関連変数（`assoc val`）
- **型エイリアス** - 型の別名定義
- **factory** - 新しいインスタンス生成（factory）
- **viaフィールドエイリアス** - 既存フィールドの別名

`impl` の**主語は型でもトレイトでもよい**：`impl Type { … }` は型に、`impl Trait { … }` はそのトレイトの全準拠型に（提供メソッド）、`impl B as A { … }` はトレイト `B` の全準拠型を別トレイト `A` へ準拠させる（トレイト間準拠）。**ベアに書けるのは surface が増える主語の所有モジュールだけ**です。`pub impl Trait` は trait owner が公開提供メソッドを載せ、修飾なし `impl Trait` は trait owner module 内 helper になります。`impl B as A` は主語トレイト `B` owner だけがベアに書け、`A` owner であるだけでは不可です（型の `impl Type as Trait` が型 owner だけに許されるのと同じ主語所有者コヒーレンス・→ [トレイト](08-traits.md)）。拡張内の `impl Trait`／`impl B as A` は**主語所有者でない側が足す追加分**で、`#Ext` で opt-in します。1 つの拡張に複数の `impl` を束ねられ（型・トレイト混在可）、**拡張の名前からは中身を推定できません**（後述の à la carte 適用）。

## 拡張の定義と使用

```plew
struct Person {
    val name: String
    val age: I32
}

extension GreetingExtension {
    impl Person {
        // メソッドの追加
        fn greet() -> String {
            return "Hello, I'm {self.name}"
        }
        
        // 関連値の追加
        assoc val defaultGreeting: String = "Hi there!"
        
        // 型エイリアスの追加
        type NameType = String
    }
}

extension FormalExtension {
    impl Person {
        fn greet() -> String {  // 同名メソッドでも拡張が違えばOK
            return "Good day, my name is {self.name}"
        }

        fn introduce() -> String {
            return "Allow me to introduce myself: {self.name}"
        }
        
        // factory の追加（→ <Person.formalPerson name=… />）
        factory formalPerson(name: String) {
            return <Person name=name age=0 />
        }
    }
}

// トレイト実装の追加例
extension MathExtension {
    impl Person as Add[Person] {
        type Output = Person
        
        assoc fn add(lhs~: Person, rhs~: Person) -> Person {
            return <Person name="{lhs.name} & {rhs.name}" age=(lhs.age + rhs.age) />
        }
    }
    
    // 複数の型を同一拡張内で拡張可能
    impl String {
        fn shout() -> String {
            return "{self}!"
        }
    }
}

fn example() {
    val person = <Person name="Alice" age=25 />

    // 拡張を明示的に適用してメソッド呼び出し
    val greeting1 = person#GreetingExtension.greet()
    val greeting2 = person#FormalExtension.greet()
    
    // 関連値へのアクセス
    val default = Person#GreetingExtension.defaultGreeting
    
    // 拡張でトレイト実装を追加した場合の演算子使用
    val combined = person#MathExtension + <Person name="Bob" age=30 />
    
    // 拡張なしではメソッド呼び出し不可
    person.greet()  // エラー: greet メソッドは定義されていない
}
```

## 拡張の制約

### 同名メソッドと `#Ext` による曖昧化

拡張は**対象型の既存メソッドと同名のメソッドを定義できます**（旧仕様の「同名禁止」は撤廃）。これは引数ベースのオーバーロードではなく、`#Ext` による明示的なシャドーイングです。

- **ベア `value.foo()` は常に型自身（無名 impl）の `foo`** に解決される。拡張のメソッドはベア解決に参加しない。
- **`value#Ext.foo()` は `Ext` の `foo`**。`#Ext` ビューでは拡張側が同名の型メンバをシャドーする（明示適用したから拡張が優先）。
- 複数拡張を同時適用して同名が衝突する場合はコンパイルエラー。ただし候補が異なるトレイト由来なら `#Trait` で選べる（後述「拡張の型システム統合」）。

外部型を自分のトレイトに準拠させる際、トレイトの要求名が型の既存メソッドと衝突しても（シグネチャ不一致で `via` 束縛できなくても）拡張内で定義でき、`#Ext` で区別できます ── これを禁止すると当該準拠が不可能になるため、同名追加を許容します。トレイト準拠は `via` で実体メンバに束ねます（[トレイト準拠と via](08-traits.md) 参照）。なお拡張の impl からは対象型の **`pub` / `pub(get)` メンバのみ**見えます（非公開は無名 impl 専用 → [メンバの可視性](05-structs-enums.md#メンバの可視性)）。

```plew
struct MyStruct {}

impl MyStruct {
    fn process() -> I32 { return 1 }           // 型自身のメソッド
}

extension Alt {
    impl MyStruct {
        fn process() -> String { return "x" }  // OK: 同名でも #Ext で区別
    }
}

val a = value.process()       // 型自身（-> I32）
val b = value#Alt.process()   // Alt のもの（-> String）
```

### 同一拡張内の解決不能な衝突はエラー

同一 `extension` の中でも、`#Ext` を開いた後に**呼び出し側が一意に選べない**候補だけを定義時点で reject します。通常のオーバーロード規則で区別できるメソッド群は共存できます。さらに、同じセレクタ・同じ呼び出し形でも、異なるトレイト由来なら `#Trait` で選べるので併存できます。これはベア表面で別トレイト由来の提供メソッドを併存させる規則と同じです。

```plew
extension DebugExtra {
    impl User {
        fn debug() -> String { return "a" }
    }

    impl User {
        fn debug() -> String { return "b" }   // エラー: User#DebugExtra.debug が一意でない
    }
}
```

```plew
trait A {}
pub impl A { fn same() -> String { return "bare A" } }
trait B {}
pub impl B { fn same() -> String { return "bare B" } }

struct S {}
pub impl S as A {}
pub impl S as B {}

extension E {
    pub impl A { fn same() -> String { return "E / A" } }
    pub impl B { fn same() -> String { return "E / B" } }
}

val s: S = /* ... */
s.same()        // エラー: ベア A / B が曖昧
s#A.same()      // "bare A"
s#B.same()      // "bare B"
s#E.same()      // エラー: E が加えた A / B が曖昧
s#E#A.same()    // "E / A"
s#A#E.same()    // "E / A"（#E#A と同じ）
s#E#B.same()    // "E / B"
```

一方、同一拡張内の同じ subject に対する同じ trait conformance は重複できません。trait が型引数を持つ場合、重複判定のキーは subject 型・target trait・明示された trait 型引数を正規化したものです。異なる型引数を持つ多重 conformance は、trait 側がそれを許す限り別 conformance として扱います。関連型は conformance が決める出力なので、同じ subject・同じ trait 型引数で関連型だけが違う impl は別 conformance ではなく重複です。

```plew
extension ConvertExtra {
    impl Box as Convert[I64] { /* ... */ }
    impl Box as Convert[I64] { /* ... */ }       // エラー: 同じ conformance の重複
    impl Box as Convert[String] { /* ... */ }    // OK: 別の型引数
}
```

別々の拡張が同じ対象・同じメンバ・同じ conformance を追加することはできます。その場合は `value#A` / `value#B` のように呼び出し側が拡張の出どころを選びます。同一拡張内でも、候補が別トレイト由来なら `value#Ext#Trait` のように trait 出所を選べます。直接メンバ同士、または同じトレイト由来でなお重なる候補は、この追加の選択子でも区別できないため定義時エラーです。

### 拡張メソッド内の `self` の型

`extension Bar { impl Foo { … } }` の中では、`self` の型は **`Foo#Bar`**（対象型 + その拡張）です。本体は一度だけ型検査されるので `self` は固定型でなければならず、拡張 `Bar` が定義時に知っているのは **対象型 `Foo` と自分自身 `Bar` だけ**だからです。呼び出し側がさらに別の拡張を効かせていても（`v: Foo#Bar#A` で `v.m()` を呼ぶ）、`m` の本体内では `self` はあくまで `Foo#Bar` で、**呼び出し側の `#A` は本体からは見えません**（見えるとモジュラに型検査できなくなる）。

この帰結として、`#Bar` ビューでは拡張が元メソッドをシャドーするため：

```plew
impl Foo {
    fn m() {}                 // 元メソッド
}

extension Bar {
    impl Foo {
        fn m() {              // self: Foo#Bar
            self.m()          // ← 拡張メソッドの再帰（#Bar ビューで Bar が元を隠す）
            self#!Bar.m()     // ← 元メソッド（#!Bar で Foo に戻す。"super" 相当）
            // self#A.foo()   // ← 別拡張 A の再利用は #A を明示スタック
        }
    }
}
```

- **`self.m()` は再帰**であって「元メソッド呼び出し」ではない。元（シャドーされた本体）へ到達する唯一の経路は **`self#!Bar.m()`**。
- 可視性は**字句的**で `self` の型に依らない。拡張 impl 内にいる限り対象型の `pub` / `pub(get)` メンバしか見えず、`self#!Bar` は解決先を変えるだけで可視性は広げない（`self#!Bar.m()` が元に届くのは元が公開メンバのときだけ）。

## 拡張の型システム統合

```plew
fn handleFormalPerson(person: Person#FormalExtension) {
    // 拡張適用済みの型として受け取り
    person.greet()      // OK: FormalExtension.greet
    person.introduce()  // OK: FormalExtension.introduce
}

fn convertExtensions(person: Person#GreetingExtension) {
    // 同名メソッドを持つ拡張を同時適用しようとするとコンパイルエラー
    // val conflicted: Person#GreetingExtension#FormalExtension  // エラー: greet メソッドが衝突

    // 拡張を外して別の拡張を適用
    val formalOnly = person#!GreetingExtension#FormalExtension
    formalOnly.greet()  // OK: FormalExtension.greet のみ

    // 拡張の付け替え
    val withGreeting = formalOnly#!FormalExtension#GreetingExtension
    withGreeting.greet()  // OK: GreetingExtension.greet
}
```

### 拡張ビューの変更は明示（暗黙キャストなし）

`A#P` は値の表現を変えない**ゼロコストのビュー**です（C 的には `A` と同一の構造体で、実行時フットプリントを持ちません）。`a#P.m()` は「`P` が `A` 用に定義したメソッド `m` を `a` を self に直接呼ぶ」に落ち、`#P` は「どのメソッド／witness を解決に使うか」を選ぶマーカにすぎません。

一方で**型検査の上では `A#P` は `A` の型細別**として扱い、ジェネリック引数にも流れ、**不変（invariant）に扱います**。帰結は次の 3 点です。

**ビューの変更は常に明示** ── `A`／`A#P`／`A#Q` の間に暗黙キャストはありません。別のビューが欲しければ呼び出し位置で `#`／`#!` を書きます。値が黙って効く拡張を変えることはありません（Plew が他でも暗黙変換を持たない方針の一貫）。

```plew
fn f(a: Person#FormalExtension) { a.introduce() }
val p = <Person name="Alice" age=25 />   // p: Person

f(p)                   // エラー：Person は Person#FormalExtension ではない
f(p#FormalExtension)   // OK：呼び出し位置で Formal ビューを明示
```

**コンテナには伝播しない（不変）** ── `Array[A]` と `Array[A#P]`、`Array[A#Ext1]` と `Array[A#Ext2]` は別の型で、暗黙キャストできません。スカラの再ビューは安全でも、コンテナは impl に依存する**内部不変条件**を抱えるためです。たとえば外部型 `A` に別のハッシュを与える `#Ext1`/`#Ext2` で `Set[A#Ext1]`（Ext1 のハッシュで配置済み）を `Set[A#Ext2]` として読めてしまうと、全ルックアップが空振りする**サイレントな論理バグ**になります。ジェネリクス一般と同じく不変に倒し、付け替えたいときは明示的に組み直します（`Set.from(…)` など）。なお[トレイト制約](08-traits.md)の充足のためにコンパイラが拡張を推論することもありません（`f(x#Ext)` と明示する＝[orphan rule](#外部型への実装は拡張でorphan-rule) の想定された使い方）。

**拡張違いはそのままオーバーロードになる** ── 暗黙キャストが無いので `A` と `A#P` は exact 一致で曖昧なく解決でき、`fn f(a: A)` と `fn f(a: A#P)` は別[オーバーロード](07-methods-impl.md#メソッドのオーバーロード)として共存します（呼び出し側が `#` の有無で選ぶ）。

## メソッド源の選択（`a#P`）

`#` は「**メソッド源の選択子**」です。`a#X.foo()` は「**X が提供する `foo`**」を呼びます ── X は**拡張**でも**トレイト**でもよく、どちらでも操作は同じ（"foo from X"）。**`#Ext`（拡張ビュー）も `#P`（トレイトビュー）も型レベルに現れます** ── `A#X` は[前節](#拡張ビューの変更は明示暗黙キャストなし)のゼロコストなビュー型（`A` の不変な型細別）で、ジェネリック引数にも流れます。`#Ext` はその拡張が加える候補を優先し、`#P` はその中でも `P` 由来の候補を選びます。両者は直交し、並び順に意味はありません：`a#Ext#P.foo()` と `a#P#Ext.foo()` は同じ呼び出しです。`A#P` は「`A`、ただし曖昧なメソッド／準拠は `P` 経由で解決する」という細別で、`A` への準拠は何も主張しません（トレイトを値の型として持つ存在型は [`any P`](08-traits.md#トレイトを値の型として使う存在型-any)・`A#P` とは別物）。`A#P` が**必要**になるのは、`A` が `foo` を `P`・`Q` の双方から得て曖昧なとき（メソッド源選択）か、`A` が[トレイト間準拠](#トレイト間準拠の衝突は-でパスを選ぶ) `impl P as R` を複数経路で得て曖昧なとき（準拠経路選択）です。ただし `#P` は**トレイト相対**で、`A` が `P` に直接準拠していても（曖昧でなくても）書けます ── その場合 `a#P.foo()` は冗長なだけで、ベアの `a.foo()` と**同じ `P` 由来の `foo`** に解決します（`fn f(a: A#P)` のオーバーロードを呼ぶときなど）。つまり `#P` は「曖昧時に必須・常時 valid」です。

**ビュー型の正規形**は「順序なしの extension 集合」と「高々一つの trait source selector」である。extension は候補を加えるため `A#E#F` のように複数を重ねられ、順序は意味を持たない。一方 trait view は候補の**出所を一つに絞る**選択子であり、異なる trait を同時に選ぶ意味を持たない。したがって `A#E#P` と `A#P#E` は同じ型だが、`A#P#Q`（P と Q が異なる）はコンパイルエラーである。P と Q の各メンバーを使うだけなら、呼び出しごとに `a#P.foo()`／`a#Q.bar()` と明示する。 

具体型でも generic でも同じです。型 A が同名 `foo` を提供する P・Q の両方に準拠していると、`a.foo()` は曖昧でエラー、`a#P.foo()`／`a#Q.foo()` で選びます。generic も同形：

```plew
impl A as P {}
impl A as Q {}              // 併存 OK（→ [提供メソッドの衝突](08-traits.md#提供メソッドの衝突曖昧はエラー)）
a#P.foo()                   // P の foo（a#Q.foo() で Q）

fn f[T](a: T) where T: P + Q {
    a.foo()                 // エラー：P.foo と Q.foo で抽象的に曖昧
    a#P.foo()               // P の foo
}
f(a)                        // A は P・Q 両方に準拠＝そのまま渡せる
```

- `where T: P + Q`（P・Q が同名 `foo` を提供）は **P・Q 両方に準拠する型なら満たせます**（A が直接両方に準拠していればそのまま `f(a)`。片方を拡張に閉じた `A#AWithQ` でも可）。
- `a#P.foo()` は**トレイト相対**なので、P を指定しない候補とは混ざりません。さらに `#Ext` が有効なら、同じ P 由来でも Ext が追加した候補がベア P の候補をシャドーします。したがって `a#P.foo()` はベア P、`a#Ext#P.foo()` と `a#P#Ext.foo()` は Ext 内の P を選びます。これは `#` の並び順でなく、適用された extension と選択した trait provenance の組で決まります。
- これは「全メソッド呼び出しが一意に解決する」という Plew の原則を、**この曖昧化に限って**緩めたものです（同名提供メソッドを持つ複数準拠は併存でき、曖昧なベア呼びだけがエラー＝use site で「曖昧はエラー」）。曖昧化は常に `#` で明示するので解決は可視。
- レシーバ起点なので `a#P.` のオートコンプリートには instance メソッドが出て、`P.` は関連関数・factory のまま汚れません（`P.foo(self: a)` 形を採らない理由）。
- **`#X` は純粋に加算的＝何も差し引かない。** `a#X` は `a` の型に「X ビュー」というタグを 1 つ積むだけで、X が提供するメソッドを可視にし（同名のベアメソッドがあれば X 側がシャドー）、それ以外の `a` の表面（inherent メソッド・**他の準拠が与えるメソッド**・他ビュー）は**そのまま見えます**。だから `a#X.foo()` の `foo` が X の提供物でなくても、`foo` が `a` のどこかにあれば解決します（例：`s#A.p()` で `p` が A でなく `S as P` 由来でも呼べる／`s#P.s()` で inherent `s` も呼べる）。ビューは「見える源を**足す**・曖昧を**選ぶ**」ものであって、引き算ではありません。**extension ビュー**を重ねると（`a#A#B`、または `val v = a#A; v#B`）extension タグは集合として積み上がり、各ビューが同じ規則で参加します（同名を 2 ビューが提供したら曖昧）。trait ビューはこの集合に直交する**単一の選択子**であり、異なる trait タグをさらに重ねて曖昧性を解く機構ではありません。

> **読み手の注意**：`a#X` の X が拡張かトレイトかは字面では決まりません（操作が同一なので実害は小さい・種別はツールで判別）。

### 拡張ビューと トレイトビューの非対称（混同しやすい）

`#X` の解決規則は拡張・トレイトで同型（前項）ですが、**何のために存在するか**は正反対です。ここを取り違えやすいので明示します。

- **拡張ビュー `#A`＝「足す」側（元々衝突させない仕組み）。** 拡張のメソッドは opt-in で、ビューを開くまでベア表面に**載りません**。だから既存のメソッドと**既定では衝突しません**。`#A` はその opt-in されたメソッド群を**追加で可視化**するためのもので、何も絞りません。`#A` を書ける条件は「拡張 A がその型に対する `impl` を持つこと」です。
- **トレイトビュー `#P`＝「絞る」側（衝突しているものを選ぶ仕組み）。** 型が `P` に準拠すると、`P` の witness と公開提供メソッドは**すでにベア表面に載っています**。だから `#P` は**何も足しません**。`#P` が要るのは、同名メソッドや[トレイト間準拠](#トレイト間準拠の衝突は-でパスを選ぶ)が**複数経路で衝突**して、ベア呼びが曖昧なときに**経路を選んで絞る**ためだけです。`#P` を書ける条件は「型がすでに `P` に**準拠していること**」で、**`#P` は準拠を発生させません**（準拠していない型に `#P` を付けることはできない＝コンパイルエラー）。

一行でいえば：**拡張タグは「衝突させない（足す）」仕組み、トレイトタグは「衝突を絞る（選ぶ）」仕組み**。どちらも加算的な解決（前項の通り base を隠さない）という点では同じですが、拡張は新しい源を持ち込み、トレイトは既存の源の中から選ぶ、という役割の違いがあります。

したがって `a#X` の妥当性条件も非対称です ── **トレイト `#P` は型が `P` に準拠していること**、**拡張 `#A` は A がその型への `impl` を持つこと**。いずれも満たさない `#X`（準拠していないトレイト・適用されない拡張・未知の名前）は、ビューが黙ってベア表面に素通りして意味を失う（hidden meaning）ため**エラー**です。

## 継承・ネストは持たない

拡張は**継承もネストもできません**（`extension B: A` のような「A を引き継ぐ B」も、拡張の入れ子も無し）。理由は拡張の核である「**呼び出し位置の `#Ext` だけで定義場所が一意に分かる**」を保つため。継承・ネストは「`value#B.foo()` の `foo` が B 由来か取り込んだ A 由来か」を不透明にし、これを壊します。

代わりに：

- **別の拡張のメンバを再利用**したいときは、`self#A.foo()` と**明示修飾**して呼ぶ（暗黙の取り込みはしない）。
- **複数の拡張を同時に効かせたい**ときは、呼び出し位置で `value#A#B` と**スタック**する（同名衝突はエラー）。

> **複数の拡張を 1 つの名前にまとめる別名（`alias Web = Json + Http` のような bundling）は持ちません（既定 no）。** 純粋なエルゴ糖衣（`value#Json#Http` → `value#Web`）で新しい意味は無い一方、`value#Web` は**どの拡張が効くか使用位置から見えなくなり**、Plew の「出どころが常に分かる（明示 > 暗黙）」をわずかに侵します。よって「便利だが将来やる」ではなく **「既定で持たない・具体的な苦痛が出たときだけ非破壊で再検討」** とします。万一入れるなら、エディタ／エラーが `#Web` を `#Json#Http` に**1 対 1 で逆展開**できること（出どころの追跡可能性を保てること）を条件とします。

## 外部型への実装は拡張で（orphan rule）

無名 impl（`impl Type …`）を書けるのは **その型を自分のモジュールで定義している場合だけ**です（コヒーレンスのため。Rust の「型 or トレイト所有」より厳格な**型所有**版。配置できるモジュールの詳細は [モジュール](../04-execution/15-modules.md) の「無名 impl の配置」参照）。トレイト自身の提供メソッド `impl Trait { … }` も同じく**トレイトを所有するモジュール内でのみ**書けます（トレイト所有版・→ [トレイト](08-traits.md)）。**外部型（他モジュール定義）への実装は、トレイトの所有を問わず、すべて名前付き拡張で行います** ── 外部トレイトはもちろん、**自分のトレイトを外部型に実装する場合も拡張が必要**です（無名だと呼び出しで結局 `#` 修飾が要り「実質拡張」になるため、経路を `#Ext` 一本に統一）。

```plew
extension Ext {
    impl ImportedStruct as ImportedTrait {
        // ImportedTrait の各要求を via で束ねる / 本体定義する
    }
}
```

Rust がこれを一律禁止するのは大域コヒーレンス（唯一の impl 保証）のためですが、Plew が許せるのは解決方法が違うからです：

- 拡張の impl は**無名ではない**ため暗黙ディスパッチには乗らず、呼び出し位置で `#Ext` を明示した時だけ発動する。
- `#Ext` は**型の一部**（`ImportedStruct#Ext`）なので、ジェネリックにも決定論的に流れる。`fn f[T](x: T) where T: ImportedTrait` は `f(x: v#Ext)` のように、拡張適用済みの型を渡して呼ぶ。
- **素の `ImportedStruct` は `ImportedTrait` を満たさない**。使用箇所で `#Ext` を適用して `ImportedStruct#Ext` にする必要がある。
- 同じ実装を別々の拡張が与えても `ImportedStruct#Ext1` と `ImportedStruct#Ext2` は**別の型**となり、`Array[ImportedStruct#Ext1]` などはコンテナごと型レベルで区別される（非コヒーレンスを「禁止」ではなく「型による区別」で防ぐ）。

### 制限

- トレイトの要求がメソッド・関連値・factory・`via` で満たせる場合は完全に実装できるが、**フィールド要求は外部型に新設できない**ため、外部 struct が該当フィールドを既に持ち `via` で束ねられる時だけ満たせる。
- 空の `impl … as … {}` が成立するのは要求ゼロのマーカートレイトのみ。要求があれば本体（`via`／定義）が必要（[トレイト準拠と via](08-traits.md) 参照）。

## 拡張はトレイトも対象にできる（第三者の提供メソッド・トレイト間準拠）

`impl` は**トレイトを主語**にできます。用途は 2 つ：**トレイトに提供メソッドを足す**（`impl Trait`）と、**トレイト間準拠**（`impl B as A`＝あるトレイトの全準拠型を別トレイトへ準拠させる）。

**ベアに書けるのは、増える surface の主語を所有するモジュールだけです**。`pub impl Trait` の提供メソッドはその trait の owner が書き、準拠型の外部ベア表面に自動で生えます（修飾なし `impl Trait` は所有モジュール内限定の helper）。`impl B as A` は主語トレイト `B` の owner だけがベアに書け、`B` の全準拠型が `A` の準拠をベアに得ます。`A` owner であるだけではベアに書けません。**第三者**（主語所有者でない側）が同じことをするのは `extension` 経由で、使用箇所で `#Ext` を明示して opt-in します。拡張に書くのはその追加分です。トレイト間準拠の衝突解決は[後述](#トレイト間準拠の衝突は-でパスを選ぶ)。

```plew
extension ExtraIterOps {             // 第三者が Iterator に足す追加メソッド（opt-in）
    impl Iterator {                  // self は Self: Iterator
        fn chunks(size: U64) -> … { /* self.next() の上に組む */ }
    }
}

extension ToStr {
    impl Format as ToString {        // トレイト間準拠：self は Self: Format
        fn toString() -> String { return self.format(format: "") }
    }
}
```

- **`impl Trait { … }`（拡張内の追加提供メソッド）**：`self` は `Self: Trait`。要求や同拡張内の他メソッドを呼べる。本体は一度だけ型検査される。所有者の `pub impl Trait` と違い**自動では生えず**、`x#ExtraIterOps` で opt-in する。
- **`impl B as A { … }`（拡張内のトレイト間準拠）**：`B` の全準拠型を `A` へ準拠させる。`self` は `Self: B`。`A` の要求を `B` の語彙で実装する。所有者がベアに書いたものと違い拡張版は**自動では生えず**、`#Ext` 適用済みの B 型だけが `A` を満たすので、`fn g[T](x: T) where T: A` には `g(x#Ext)` で渡す（[外部型への実装](#外部型への実装は拡張でorphan-rule)の `f(x: v#Ext)` と同じ流れ）。
- **頭なし Self の blanket は禁止**：`impl[T] T { … }` や `impl[T] T as A where T: B { … }`（Self が型構築子を持たないベア型変数）は書けません。型所有が錨を下ろせず、型のベアメソッド一覧をローカルに列挙できなくなるためです。「B の全準拠型を A へ」は `B` owner なら**ベアな `impl B as A`**（主語＝トレイト）、`A` owner を含む第三者なら拡張内の `impl B as A` で表現します（`impl[T] T as A where T: B` と同義だが、頭なし型変数の代わりに主語トレイト `B` が錨になるので列挙可能性が保てる）。

### トレイト間準拠の衝突は `#` でパスを選ぶ

トレイト間準拠（`impl B as A`）は提供メソッドと同じく、**ベアに併存させ、曖昧なときだけ `#` で源を選ぶ**（提供メソッドの `a#P.foo()` と対称）。型が**複数の経路で同じトレイトに準拠**できてしまう場合は、その型は**単体ではそのトレイトに準拠しないものとして型解決**し、`#` で経路（主語トレイト）を指定して初めて準拠と認めます。

```plew
trait P {}  trait Q {}  trait R {}
struct S { … }

impl P as R { … }   // P 準拠型は R になる
impl Q as R { … }   // Q 準拠型も R になる
impl S as P { … }
impl S as Q { … }   // S は P 経由でも Q 経由でも R に届く＝衝突

fn f[T](a: T) where T: R { … }

f(a: s)     // ✗ コンパイルエラー：s が R に準拠する経路が P・Q の 2 通りで曖昧
f(a: s#P)   // ✓ P 経由の R 準拠を選択（`impl P as R` の witness を使う）
f(a: s#Q)   // ✓ Q 経由
```

経路が 1 つだけなら（`impl P as R` と `impl S as P` のみ）`s` はベアで `R` に準拠し、`#` は要りません（提供メソッドが 1 つならベア呼びできるのと同じ）。**`#` は「メソッド源セレクタ」を「準拠経路セレクタ」にも広げたもの**で、エッジケースの衝突時にだけ要る点は提供メソッドと共通です（解決規則はやや複雑になるが、複雑さを衝突時のみに閉じ込めるのが狙い）。

**準拠は推移する（連鎖）** ── `impl C as B` と `impl B as A` があれば、`C` の全準拠型は `B` を経て `A` にも準拠します（`S: C → B → A`）。直線的な連鎖（各ホップが一意）ではベアにそのまま載り、直接呼び `s.a()` も `where _: A` 境界も同じく推移準拠を見ます。衝突したときの絞り込みは上と同じく `#` で行い、**`#` で指定できるのは `impl` を持つ（経路上の）主語トレイトだけ**で、衝突が消えるところまで（経路上の任意の段で）絞れば解決します。2 つの異なる経路は必ずどこかの主語トレイトで分岐するので、一意化する `#` は常に存在します。

衝突が**連鎖の中間**で起きるときは、共有されたメソッド本体の呼び出しも通常の曖昧性規則に従います。`impl C as B`・`impl D as B`・`impl B as A` があり、`A` の witness `a` が本体で `self.b()`（`B` のメソッド）を呼ぶとします。`b` は C・D の 2 経路で実装されるため、**`self.b()` 自体が曖昧でエラー**です。外側の `s#C.a()`／`s#D.a()` は `a` を呼ぶ入口を選ぶだけで、共有本体の意味を暗黙に書き換えません。

必要なら `a` の実装者が本体で `self#C.b()`／`self#D.b()` のように明示します。ただしその選択子は `impl B as A` の抽象的な `self: B` に実際に適用でき、かつ本体を一度だけ型検査できる場合に限ります。呼び出し元の view・具象準拠経路・単相化順序を本体の解決へ持ち込むことはありません。これは extension メソッド本体で呼び出し側の `#` が見えない規則と同じで、メソッド本体の意味を宣言位置だけから読めるようにします。

### トレイト間準拠の循環は構造エラー

トレイト間準拠の循環は、値の実体・generic の concrete instance・extension を適用する receiver を待たずに reject する**構造エラー**です。`impl A as B` と `impl B as A` のような bare な循環は、宣言グラフそのものが不正です。これは Rust / Swift / Java / C# が trait / protocol / interface の直接・間接循環を拒否する方針と同じです。

```plew
trait A {}
trait B {}

impl A as B {}
impl B as A {}   // エラー: A <-> B の循環
```

extension は receiver に対する適用を opt-in にする名前付き bundle だが、bundle 内の trait-to-trait edge が循環してよいことは意味しない。cycle 検査における bundle は、同じ解決済み extension identity を持つ全 declaration part を統合したものである。コンパイラはロードされたプログラムの全 bare edge と全 bundle を登録してから、各 bundle についてその edge と bare edge の和を宣言検査する。したがって module・ファイル・宣言順によって cycle の可否は変わらない。この検査は関数本体・receiver・concrete instance を検査する前に行う。

そこに cycle があれば、一部の receiver では cyclic edge が到達不能でも、その bundle の**宣言整合性**が壊れているため bundle 全体を reject する。これは「どの receiver にも安全な利用がない」という主張ではない。receiver/bound は後述する bundle の**適用可能性**だけを決め、cycle を解消しない。

```plew
trait A {}
trait B {}

impl A as B {}

extension E {
    impl B as A {} // エラー: bare A -> B と E の B -> A が循環
}
```

同じ bundle の内部だけで閉じる cycle も同様に定義時エラーである。receiver がその trait に準拠しているか、プログラムが bundle を実際に使うかは関係ない。

```plew
trait B {}
trait C {}

extension E {
    impl B as C {}
    impl C as B {} // エラー: E 自身が循環
}
```

別々の extension を**組み合わせたときだけ** cycle になる場合、各 bundle は単独では健全なので宣言検査では reject しない。extension 集合は `#`/`#!` により明示的に変更されるが、cycle 検査の source-set は一つの view 式を最後まで正規化して得る extension 集合である。trait source selector は edge を加えないため含まない。選択した bundle 群と bare edge の和に cycle があれば、receiver の型・bound・実体に依存せず、その view 式を reject する。したがって `T#E#F#!F` は最終集合が `{E}` なら `{E}` として一度だけ検査し、中間の `{E,F}` は観測しない。この規則は値 view・型注釈・関数シグネチャ・generic 引数など、extension 集合を形成または変更する全ての箇所に適用する。

```plew
trait A {}
trait B {}

extension E {
    impl A as B {}
}
extension F {
    impl B as A {}
}

fn use[T](value: T#E) where T: A {}       // OK
fn invalid[T](value: T#E#F) where T: A {} // エラー: source-set E + F が A <-> B を作る
```

型引数を持つ trait edge も同じく、宣言された generic pattern を単一化して**同じ trait instance へ戻る有限 cycle があり得るか**を検査する。各 edge の traversal では binder を fresh にし、substitution はその path にだけ属する。trait owner、位置ごとの型引数、関連型 binding を同一性の対象にし、単一化には occurs check を用いる。`where` の充足可能性や現存する conformer は、この構造検査を抑制しない。同じ宣言 edge を別の instance で再び通ることも許され、閉路が同じ instance に戻るなら reject する。

```plew
impl[T] A[T] as B[T] {}
impl B[I64] as A[String] {}
impl B[String] as A[I64] {} // エラー: A[I64] -> B[I64] -> A[String] -> B[String] -> A[I64]
```

実際の concrete instance が現れるまで診断を延期しない。一方、`A[T] -> B[Array[T]] -> A[Array[T]]` のように始点へ戻るには `T = Array[T]` が必要で occurs check に失敗し、同じ instance へ戻らず型だけが成長する関係は cycle とは別問題である。この規則では reject しない。

これは Rust の trait solver のように、具体的な利用時に再帰制限や overflow として落ちる挙動にはしない。Plew は extension の**適用可能性**だけを receiver/bound に基づいて判定し、cycle・重複・解決不能な衝突のような**宣言整合性**は eager に reject する。たとえば `extension E { impl B { … } }` に対する `value#E` は、value が B に準拠する（generic なら `where T: B` がある）ときだけ有効だが、E の trait graph が健全かどうかは value を待たずに決まる。

### 拡張は名前付きバンドル（à la carte 適用）

1 つの拡張は**任意の `impl` を束ねた名前付きバンドル**にすぎず、名前から中身は決まりません。`value#P` は「P の中で value の型が**資格を持つ部分だけ**を活性化する」à la carte 適用です。

```plew
extension P {
    impl B {}        // B 準拠型に効く部分
    impl C {}        // C 準拠型に効く部分
}
```

- `T#P` という型は**何の準拠も主張しません**。`#P` はあくまで「拡張 P のビューを開く」だけで、その中の `impl Iterator { … }` が `T` に**適用される**ことは別問題です。拡張のメソッドを境界型変数で使うには、`it#P` で拡張ビューを開き、かつ `where T: Iterator` で `impl Iterator` 部が適用されることを保証する必要があります（直交する 2 条件で、両方必要）。

```plew
extension ExtraIterOps {                               // Iterator の既定ではない追加拡張
    impl Iterator { fn chunks(size: U64) -> … { … } }
}
fn process[T](it: T#ExtraIterOps) where T: Iterator {  // where は T#ExtraIterOps から含意されない＝省略不可
    it.chunks(size: 2)                                 // 本体はベア
}
```

## 拡張の既定化は持たない

Plew は `defaultExtension` のように、型本体で拡張を既定化する構文を持ちません。拡張は常に使用箇所の `#Ext` で明示します。型の作者が拡張由来の振る舞いを型のベア表面に採用したい場合は、型を所有するモジュールの `pub impl Type` / `pub impl Type as Trait` に明示的な forwarding メソッドや準拠を書きます。

この制限により、拡張メソッド・拡張内トレイト準拠・witness provenance は常に `#Ext` の有無だけで読めます。トレイト自身の公開提供メソッドはこの仕組みではなく、準拠（`impl Type as Trait`）によって自動的にベア表面へ載ります（→ [トレイト](08-traits.md)）。
