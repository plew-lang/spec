# メソッドと impl

型に紐づく関数（メソッド・関連関数）と生成（factory）は `impl` ブロックにまとめます。`impl Type { … }`（無名 impl）はその型固有の振る舞いを、`impl Type as Trait { … }` はトレイト準拠を与えます（[トレイト](08-traits.md)）。

> **無名 impl を書けるのは、その型を自分のモジュールで定義している場合だけ**です（コヒーレンス＝由来の一意性のため）。外部型（他モジュール定義）への実装は、トレイトの所有を問わず名前付き拡張 `#Ext` で行います（→ [拡張](09-extensions.md)、配置の詳細は [モジュール](../04-execution/15-modules.md) の「無名 impl の配置」）。

```plew
pub impl MyType {                          // pub impl → ブロック内のメンバは全て公開
    // インスタンスメソッド（self を読み借用）
    fn instanceMethod(arg: Type) -> ReturnType {
        // メソッド本体
    }

    // 可変メソッド（self を可変借用＝inout self）
    inout fn mutableMethod(arg: Type) {
        // self のフィールドを書き換えられる
    }

    // 消費メソッド（self を move＝呼ぶと self は使えなくなる）
    // ※ move fn は unique 型でのみ。コピー可能型はコピーで済むため move は冗長＝エラー（→ spec/03）
    move fn intoOther() -> Other {
        // self を消費して別の値を返す
    }

    // 関連関数（静的メソッド相当）
    assoc fn associatedFunction() -> ReturnType {
        // 関連関数本体
    }

    // 関連変数（静的変数相当）
    assoc val associatedValue: Type = defaultValue
}

impl MyType {                              // 修飾なし impl → ブロック内は全て非公開
    fn helper() { /* この型の無名 impl からのみ見える */ }
}
```

- **メンバの可視性は `impl` ブロック単位**（[メンバの可視性](05-structs-enums.md#メンバの可視性)）。`pub impl Type { … }` はブロック内のメソッド・関連関数・`factory` をすべて公開し、修飾なし `impl Type { … }` はすべて非公開（その型の無名 impl からのみ見える）にします。**メソッド個別の `pub`（`pub fn`）は書けません** ── 公開と非公開を混在させたいときは `pub impl` と `impl` の **2 ブロックに分けます**（フィールドは読み書きの非対称＝`pub(get)` があるので例外的にフィールド単位 → [メンバの可視性](05-structs-enums.md#メンバの可視性)）。
- **`assoc val` の初期化はトップレベル `val` と同じ規則**：起動時 eager・依存順（force-on-read）・循環は起動時 panic・トップレベル await 可（→ [モジュール § トップレベル初期化と実行順序](../04-execution/15-modules.md#トップレベル初期化と実行順序)）。ジェネリック型の `assoc val` が型引数ごとに別実体になるか（単相化）は実装詳細として別途。

- **self のモードは `fn`（読み借用）／`inout fn`（可変借用）／`move fn`（消費）** ── [アクセスモード](../01-basics/03-values.md#アクセスモードborrow--inout--move)と同じ語。`inout fn`/`move fn` は対象が可変束縛／唯一所有のときだけ呼べる。**`Ref` 越しは `fn`/`inout fn` のみ**（`move fn` は共有を消費するため不可 → [Ref](../01-basics/03-values.md#ref--weakref共有可変)）。
- **`async` メソッドは self を借用できない**（`inout self` が境界を跨げない）。非消費なら copy-self（`unique` でない型のみ）、消費なら `async move fn`、await を跨いで self を変更するなら **Ref 裏打ち** → [非同期処理とメモリ管理](../04-execution/14-concurrency.md)。

`impl` には `factory`（インスタンス生成）も書けます。生成は型の生成として [構造体と列挙型](05-structs-enums.md) の「インスタンス生成」にまとめています。トレイト準拠の `impl Type as Trait` と `via` は [トレイト](08-traits.md) を参照。

## メソッドのオーバーロード

同じ名前のメソッドを、引数で区別して複数定義できます（オーバーロード）。解決は**ラベルと引数の具体型**だけで行い、トレイト探索や import に依存しません。

**セレクタ** = メソッド名 ＋ 順序付きラベル集合。Plew は呼び出しにラベルを強制するので、**ラベル集合が違えば別メソッド**（型を見ずに区別）。

同一セレクタ内に複数の定義を置く条件：

- **型引数の形が一致**：型引数の個数と、それを使う引数位置が全オーバーロードで同じ。違反は、たとえ他で解決できても一律エラー。
- **具体位置でのみ区別**：全員で具体型な引数位置のうち、最低 1 つで型が異なる。
- **制約は区別に使わない**：`where T: TraitA` の違いだけでオーバーロードを分けられない（区別は具体型のみ。制約は選ばれたオーバーロードの型引数に効く）。

```plew
impl A {
    fn f[T](a: I16, b: T) {}
    fn f[T](a: I32, b: T) where T: TraitA {}   // 具体位置 a で区別、b は全員 T
    fn f[T](a: I64, b: T) where T: TraitB {}
    fn f[T, U](b: T, c: U) {}                  // ラベルが (b, c) → 別セレクタ＝別メソッド
}

// 禁止：同セレクタ f(n:) で n が generic vs 具体
// fn f[T](n: T) {}
// fn f(n: I32) {}
```

- すべてのオーバーロードはその型のモジュール内（無名 impl）に集まるので、**衝突検出はモジュール単体のビルドで完結**する（Rust のようなグローバルコヒーレンスは不要）。拡張（`#Ext`）のメソッドはベア解決に参加しないので別扱い。ただし型のベア集合には、型自身のメソッドに加えて**準拠トレイトの公開提供メソッド**（`pub impl Trait` → [トレイト](08-traits.md)）が加わる。同セレクタ・同シグネチャが**同一の出どころ**で重複すれば定義地点でエラー、**別トレイト由来で鉢合わせ**たら併存は許し*曖昧なベア呼び*だけがエラー（`a#P.foo()` で源を選ぶ → [トレイト](08-traits.md#提供メソッドの衝突曖昧はエラー)）。型自身の inherent メソッドとトレイト由来のメソッドが同セレクタで重なる場合、ベアの `a.foo()` は型自身のメソッドを既定として解決し、トレイト由来のものは `a#P.foo()` で選ぶ。
- 解決：ラベルでセレクタを確定 → 具体位置の型で 1 つ選ぶ → 型引数位置は推論＋制約チェック。重なり得るジェネリック impl は保守的に拒否し、specialization（最特化選択）は行わない。
- リテラル引数は文脈（このオーバーロード集合）で型が一意に定まらなければエラー（→ [数値リテラル](../01-basics/02-basic-types.md)）。
- **拡張違いは別の具体型として区別**：`#Ext` は型の一部なので `f(a: A)` と `f(a: A#P)` は別オーバーロードとして共存できる（[暗黙の拡張キャストは無く](09-extensions.md#拡張ビューの変更は明示暗黙キャストなし) exact 一致で解決・呼び出し側が `#` の有無で選ぶ）。

### メソッド・関連関数は値に取り出せない（クロージャで包む）

`val f = obj.method` や `val f = Type.associatedFunction` のように**メソッド・関連関数をそのまま関数値へ代入することはできません**。これは[名前付き関数宣言](../01-basics/04-functions.md#関数宣言)と同じく「呼び出せる宣言」であって値ではありません。関数値が欲しいときは**明示クロージャで包みます**。

```plew
val f = fn(value: I32) { obj.method(value: value) }   // OK
// val f = obj.method                                  // ✗
val g = fn() -> I32 { return Type.associatedFunction() }  // OK
// val g = Type.associatedFunction                         // ✗
```

理由は 2 つ。**(1) どのオーバーロードか**：メソッドはセレクタ（名前＋ラベル）と引数型でオーバーロードされ得るので `obj.method` 単体では一意でない。クロージャの中の呼び出しが具体型・ラベルでオーバーロードを一意に選ぶ。**(2) `self` のアクセスモード**：メソッドは `self` を `borrow`/`inout`/`move` で取るが、これらを値として持ち回ると borrow は寿命を越えられず inout/move/unique-self は脱出できない。クロージャで包めば、self の取り回しは[クロージャの参照キャプチャ規則](../01-basics/04-functions.md#環境のキャプチャ)が**そのまま**効き、通常の `fn` はsendable保証を持たず、別スレッドへ送るなら [`sendable fn`](../01-basics/04-functions.md#sendable-クロージャ) として明示・検査するので専用規則が要らない。ラベルもクロージャリテラルの引数名として保たれる。[ラベル違いの適応を明示クロージャで行う](../01-basics/04-functions.md#ラベルは型の一部厳密な同一性暗黙変換なし)のと同じ方針（暗黙の魔法を足さない）。セレクタ明示の糖衣（`obj.method(value:)`）は将来の非破壊追加候補。
