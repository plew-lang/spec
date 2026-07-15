# 値・変数・所有権

## 値意味論（value semantics）

Plew の値の受け渡し ── 代入・引数渡し・構造体への埋め込み ── は**論理的なコピー**です。`val b = a` の後、`b` と `a` は独立で、片方を変更しても他方に影響しません。**共有された可変状態は [`Ref`](#ref--weakref共有可変) を通したときだけ**生まれます。

> **CoW（copy-on-write）**：論理コピーですが、コンパイラ／ランタイムは実コピーを**遅延**します。共有中は同じ内部バッファを指し、いずれかを**変更した瞬間**にだけ複製します。だから読み取り共有は無料で、`Array`／`String`／`Dictionary` のような大きい値も安価に渡せます（「全代入が即コピー＝遅い」は誤解）。観測意味論は常に「独立」で、CoW は観測できない最適化です。

ランタイムは **ARC（自動参照カウント）** でメモリを管理します。決定的に破棄されるので [`deinit`](#deinit) による資源解放が使えます。通常の単一スレッド値は軽量な非 atomic カウントを使い、`spawn`・チャネルを介して複数スレッドに共有され得る値だけは、whole-program 解析により allocation 時点から atomic カウントを使います。例外として、不変トップレベル/`assoc val` のストレージと、それが不変値として固定的に所有する backing はプロセス寿命の immortal allocation なので寿命管理の retain/release 自体が不要です（`Ref` の中へ後から格納される値など、可変な内部状態の allocation は含みません）。これらの違いはコンパイラ内部の実装方式で、型名や構文には現れません（→ [非同期処理とメモリ管理](../04-execution/14-concurrency.md#参照カウント方式の静的選択)）。循環参照だけは参照カウントで回収できないため [`WeakRef`](#ref--weakref共有可変) で断ち切ります。

## 再帰する値型（finite・間接化は隠れたコスト）

値型は**自分自身を含む**ように定義できます。連結リストや木がそのまま書けます。

```plew
enum List {
    Cons(head: I64, tail: List)   // ペイロードに自分自身を含む
    Nil
}
struct Node {
    pub val v: I64
    pub val next: Optional[Node]  // Optional 経由で自分自身を含む
}
```

これらは**有限サイズ**として定義できます。素朴に展開すれば無限サイズになりますが、コンパイラが循環を閉じるフィールドを**裏でヒープ間接化**して有限化します（Rust の `Box`・Swift の `indirect` のような**明示キーワードは書かせません**）。間接化は**隠れたコスト**であって意味ではありません（拠り所：唱えた通りに発現・仕組みは問わない）。観測上はあくまで値で、**値意味論はこの間接化を越えても保たれます**──`val b = a` のコピーは独立で、片方の（再帰）フィールドを書き換えても他方は影響を受けません。

- **再帰には終端（base case）が要ります**。終端は enum が与えます（`Nil`・`Optional.None` のような空バリアント）。`struct A { val a: A }` のように enum を経由しない構造体だけの循環は、終端が無く**値を構築できません**（有限型ですが住人がいません）。
- `Array[T]` を経由する再帰（`struct Tree { val kids: Array[Tree] }`）は、`Array` がもともと要素をヒープ（CoW バッファ）に持つので**そのまま有限**です（追加の間接化は不要）。

> 実装上の現状の限界：再帰フィールドのヒープセルは当面 **ARC 解放を未配線（リーク）**です。観測挙動（値意味論・出力）は正しく、解放は隠れたコストの後付け（additive）です。詳細は [claude/provisional.md](../../claude/provisional.md)。

## 変数宣言（val / mut val）

```plew
val x: I32 = 0               // 不変束縛（既定・初期化必須）
mut val y: String = "init"   // 可変束縛（再代入・可変メソッド可）
```

- **既定は不変 `val`**。可変にするには `mut val` と一語多く書く（「手を抜くと不変」）。`mut` は**記憶域の可変性**を表す語で、後述の[借用](#アクセスモードborrow--inout--move)とは別軸です。
- **宣言時に必ず初期化**します（未初期化宣言は持ちません）。分岐で初期値を決めたいときは式ブロック：`val x: I32 = if flag { give 1 } else { give 2 }`。
- **`val` 束縛の中身は変更できません（合成規則・Swift と同じ）**。フィールドを書き換える（`c.n += 1`・場所越しの代入）・`inout fn` を呼ぶには、**束縛が `mut val` で、かつそのフィールド自身が `mut val`** の両方が要ります。値型は束縛単位で凍結されるので、`val c = <Counter n=0 />` の `c.n += 1` は**フィールドが `mut val` でも**エラーです（束縛 `c` が `val`）。逆に `mut val c` でもフィールドが `val n` なら `c.n` は変更できません（`c` 全体の再代入は可）。Ref はこの合成の例外で、`val` 束縛越しでも referent を変更できます（[Ref / WeakRef](#ref--weakref共有可変) 参照）。

## アクセスモード（borrow / inout / move）

関数引数・メソッドの `self`・呼び出しは、値を**どう扱うか**を 3 つのモードで表します。**呼び出し側にもモードが現れる**ので、各変数の運命が呼び出し位置だけで読めます。

| モード | 意味 | 引数宣言 | self | 呼び出し側 |
| --- | --- | --- | --- | --- |
| **borrow** | 読み借用（変更しない） | `x: borrow T` | `fn` | `f(x: borrow a)` |
| **inout** | 可変借用（書き戻し・排他） | `x: inout T` | `inout fn` | `f(x: inout a)` |
| **move** | 消費（所有を移す） | `x: move T` | `move fn` | `f(x: move a)` |

- **`unique` でない型（コピー可能型）は既定で by-value**：`x: T`（モード語なし）＝コピー渡し・呼び出しも `f(x: a)`。コピー独立なので「読むだけ」相当です。
- **コピー可能型に `borrow` / `move` を書くのはコンパイルエラー**です。`borrow` は観測挙動が by-value と完全に同一（読むだけ・元は無傷・コピーは CoW で実質ゼロコスト、しかもコンパイラがコピー省略を自動で行う）で**何も意味を足さない冗長な指定**であり、`move` はコピーで済むのに「以後使えない」制約だけを課す（`f(x: move a)` 後に `a` が使えなくなる）footgun だからです。コピー可能型で意味を持つモードは **`inout` だけ**。これらが要るのは `unique` 型（コピー不可）のときです。
- **`inout` はコピー可能型でも `unique` 型でも書けます**：書き戻し（呼び出し側の値が後で変わる）という観測可能な意味を持ち、コピー可能性とは独立だからです。
- **対称性**：コピー可能型は「コピーできるので `borrow`/`move` は不要＝エラー」、`unique` 型は「コピーできないので by-value（モード無し）が不可＝モード必須」。`inout` だけが両方にまたがります。これで各変数の運命が呼び出し位置で一意に読めます。
- 関連：v1 のジェネリクスはコピー可能型限定なので、generic `[T]` の引数は by-value（と `inout`）で書き、`borrow`/`move` は書きません（将来 `allowUnique` を入れたとき初めて「unique かもしれない `T` には `borrow`/`move` 可」になります）。メソッドの self も同様で、コピー可能型は `fn`（self 読み・既定）／`inout fn` のみ、`move fn`（self 消費）はエラーです。
- **`inout` は旧 `&mut` の CoW 版**（Swift の `inout` と同じ位置づけ）。「変更する」行為ではなく「変更可能に借りる関係」を表すので、`modify` のような行為語ではなくこの語を使います。
- 呼び出しの読み分け：`f(x: a)`＝x は無傷／`f(x: inout a)`＝x はこの後変わり得る／`f(x: move a)`＝x は以後使えない。

```plew
inout fn deposit(amount: I32) { self.balance = self.balance + amount }  // self を可変借用
fn balance() -> I32 { return self.balance }                            // self を読み借用
```

## unique（コピー不可型）

```plew
unique struct File {
    val fd: I32
    deinit { sysClose(fd: self.fd) }
}
```

`unique` を先頭に付けた型は**コピー不可・move 専用**です。唯一所有が要る型 ── 資源（ファイル・ソケット）、ハンドルなど ── に使います。**struct と enum の両方に付けられます**（Swift の noncopyable enum と同型）：

```plew
unique enum Handle {
    Open(file: File)     // unique なペイロードを合法に保持できる
    Closed
}
```


> **用語**：本書は `unique` を基準に書き、**「`unique` でない」＝コピー可能**（他言語でいう *Copyable*）と表します。*Copyable* はキーワードではありません。スレッド境界を安全に越えられる性質は [`sendable`](#sendable--nonsendableスレッド間の移送可能性)、越えられない性質は `nonsendable` と表します。

- **束縛は move のみ**：`val f2 = move f`（以後 `f` は使えない）。`val f2 = f`（コピー）は**エラー**です。
- **アクセスはモード明示必須**：コピー可能と違い bare 不可。`borrow`/`inout`/`move` のいずれかを書く。
- **`unique`（または unique を推移的に含む型）をフィールドに持つ型は `unique` 明示必須**。書かなければコンパイルエラー（自動伝染させず、フィールド追加時に決定を強制する＝[全フィールド明示](11-control-flow.md)と同じ精神）。**enum も同様**：unique（または unique を推移的に含む型）をバリアントのペイロードに持つ enum は `unique enum` 明示必須。
- **ジェネリクスには直接渡せない**（型引数はコピー可能な型に限定）。共有・格納したいときは [`Ref`](#ref--weakref共有可変) で包む（`Optional[Ref[File]]` 等 → [ジェネリクス](../02-type-system/06-generics.md)）。

### deinit

v1 で明示 `deinit` 本体を書けるのは `unique struct` だけです。コピー可能な型は不可 ── コピーのたびに多重実行され二重解放になるためです。`unique enum` はコピー不可・move 専用の値として使えますが、enum 型自身の明示 `deinit` 本体は v1 では書けません（Swift の `~Copyable enum` でも deinitializer は未サポート）。enum を破棄すると active payload は通常通り破棄されます。

```plew
impl File {
    deinit { sysClose(fd: self.fd) }   // 最後の所有者が消えるとき一度だけ
}
```

- 引数・戻り値なし、失敗不可。最後の所有者が所有者として到達不能になるとき（または最後の [`Ref`](#ref--weakref共有可変) が解放されるとき）に**ちょうど一度**走る。
- **決定的契約は正常路のみ**：`deinit` が決定的に走るのは正常な終了路（スコープ離脱・`return`・`try`/`Result` 伝播）に限る。**契約外の脱出は 2 つ＝`panic`（abort・[panic と発散](../03-expressions/11-control-flow.md#panic-と発散)）と「リークした循環」**（`Ref` で意図せず作った循環）で、どちらも `deinit` を走らせない。後者は将来サイクルコレクタがメモリを回収する際も deinit を走らせず loud に報告する（→ [非同期 § メモリ管理](../04-execution/14-concurrency.md#メモリ管理arc)）。`deinit` の有無で循環の扱いを変えない（「`deinit` を足したらリークし始める」を防ぐ）。
- 値 struct と enum には書けない（v1 では `unique struct` 宣言が前提）。
- **一時値の破棄時点＝それを生んだ文（完全式）の終端**：名前の無い値（関数結果・構築・要素読みをそのまま引数やレシーバに使った値）は、その**文の評価が終わった時点**で破棄される（Swift の一時値と同時点）。ブロック末まで生き延びない ── 資源（`unique`＋`deinit`）を一時値として使えばその文の終わりで解放され、文を跨いで使いたければ名前を付ける（以後は名前付き値の破棄規則に従う）。条件付きに評価された部分式（短絡 `&&`/`||` の右辺・`?.` の腕）の一時値は、**その経路が実行されたときだけ**生成・破棄され、**その部分式の評価終端（合流前）で破棄される**（生成されなければ破棄も無い）。
- **制御文の条件式はそれ自身が完全式**：`if`/`while` の条件式の一時値は**条件の評価が終わった時点（分岐の前）**に破棄される（Swift/C++ の full-expression と同時点）。`while` では条件は反復ごとに評価されるので、その一時値も**反復ごとに**生成・破棄される。条件の一時値が then/ループ本体を跨いで生きることはない ── 本体でも使いたければ名前を付ける。`match` **文**の scrutinee は条件ではなく**束縛の出どころ**なので、match 文全体の終端まで生きる。**値位置の `match`（式として大きい文に埋め込まれた match）** の scrutinee は、条件付き部分式（`?.` の腕）と同じ規則で **その match 式自身の評価終端（値の確定直後・囲む文の続きより前）** に破棄される ── 腕の値は独立コピーとして合流するので、scrutinee を文末まで延命する理由が無い。
- **名前付き `unique` 値の破棄時点＝accessibility cliff**：名前付き `unique` place は、「その place が名指せて所有している」最後の点まで生き、所有者として到達不能になる最初の CFG edge で破棄される。通常のスコープ離脱はこの一種だが、条件付き `move` の合流や shadowing でも起きる。例：`if c { consume(file: move f) }; print("done")` では、`c == false` の経路で残った `f` も `print` より前の合流 edge で `deinit` され、合流後のコードは `f` を所有していない前提で一様に読む。
- **drop edge は借用衝突点**：accessibility cliff / scope exit / shadowing など、外側の制御フローが place を破棄する edge は、その place への破棄アクセスである。そこを越えて生きる `borrow` / `inout` があればコンパイルエラーにする。借用は drop 点を後ろへ延ばさない。
- **同一文内の複数の一時値は生成の逆順で破棄**（Swift/C++ と同じスタック規律）：後に作られた値が先に消える＝後の値が前の値に依存し得る向きと整合。
- **破棄の順序（決定的）**：まず**型自身の `deinit` 本体**が走り（このとき全フィールドはまだ有効で本体から使える）、続いて**各フィールドの `deinit` が宣言順（上から下）**に走る。フィールドが入れ子の unique 型なら再帰的に同じ規則。`deinit` 本体は全フィールドが有効な値を前提に読むため、v1 では [`unique struct` の field partial move](../03-expressions/11-control-flow.md#パターンマッチ) を reject する。「上から下」で一貫させる ── Plew はフィールド既定値が[他フィールドを参照できない](04-functions.md#デフォルト引数)ため**フィールド間の構築依存が存在せず**、C++ の「構築の逆順で破棄」にする動機が無いので、最も単純な宣言順を採る。外部的な解放順序の制約が要る稀なケースは構造（ネスト）で表現する。
- **同じ drop edge 上の named locals は宣言の逆順**（C++/Rust/Swift と同じスタック規律）：後の束縛は先行する束縛を参照して構築でき（`val b = f(a)`）、依存される側が後に消えるのが安全な向き。通常のスコープ離脱では、そのスコープ内でまだ所有されている locals がこの順に破棄される。フィールドの「宣言順」と非対称なのは意図的 ── フィールドには構築依存が存在しない（前項）が、locals にはあるため。
- **`return` では文の一時値が frame の locals より先に破棄される**：一時値は最も内側の寿命（生成の逆順規律の帰結＝最後に生まれたものが最初に消える）。順序は「return 値の計算・move → その文の一時値 → named locals（宣言の逆順）」。

## sendable / nonsendable（スレッド間の移送可能性）

`sendable` は、値を別スレッドへ安全にコピーまたは move できる性質です。`nonsendable` はその否定で、[`Ref` / `WeakRef`](#ref--weakref共有可変) のようにスレッド境界を越えられない根本型を宣言します。ユーザー定義型も、フィールドだけからは分からないスレッド束縛を表すために明示できます。

```plew
nonsendable struct Ref[T] { /* 組み込み表現 */ }
nonsendable struct WeakRef[T] { /* 組み込み表現 */ }

nonsendable struct UiContext {
    val rawHandle: U64
}
```

`nonsendable` は**構造的に自動伝播**します。`nonsendable` なフィールドを直接または推移的に含む struct、nonsendable な payload を持つ enum は、外側に `nonsendable` と書かなくても nonsendable です。

```plew
struct Session {
    val conn: Ref[Connection]
}
// Session は Ref[Connection] を含むため自動的に nonsendable
```

これは [`unique`](#uniqueコピー不可型) と意図的に非対称です。`unique` はコピー可否と呼び出し規則を日常的に変える肯定的な所有権モードなので、unique フィールドを持つ外側にも `unique` の明示を要求します。一方、nonsendable の伝播は spawn 可能性を狭めるだけで、通常の単一スレッドコードの意味を変えません。日常コードへ不要な宣言を広げないため、外側の明示は要求しません。

通常の struct/enum は、すべてのフィールド/payload 型が sendable なら自動的に sendable です。generic 型では、**実際にフィールド/payloadとして所有する型引数**から構造的に導出します。例えば `Box[I64]` は sendable、`Box[Ref[I64]]` や `Optional[Ref[I64]]` は nonsendable です。表現に現れない phantom 型引数だけでは外側の sendability は変わりません。[存在型](../02-type-system/08-traits.md#存在型の-sendability)は無印 `any P` が nonsendable、保証を保持する `any sendable P` が sendable な構成型です。明示的な `nonsendable struct` は、フィールドがすべて sendable でも sendability の導出を禁止します。

nonsendable 値は実スレッド境界（`spawn`・スレッド間チャネル）を越えられません。単一スレッド（`async` を含む）では sendable 値と全く同じに扱えます。関数値の明示的な sendability は [無名関数（クロージャ）](04-functions.md#sendable-クロージャ)を、境界規則の詳細は [非同期処理とメモリ管理](../04-execution/14-concurrency.md) を参照。

**sendability と参照カウント方式は別軸**です。`sendable` は値の内容を別スレッドから扱ってもデータ競合しないことを表し、atomic な参照カウントは共有された値の寿命管理だけを安全にします。したがって `Ref` を atomic カウントで保持しても、その referent の共有可変性は消えず nonsendable のままです。逆に、深く sendable な不変値や CoW 値は、共有され得る allocation の参照カウントだけを atomic にすれば、実データをコピーせずスレッド間で安全に共有できます。

## Ref / WeakRef（共有可変）

値意味論では共有された可変状態を作れません。**`Ref[T]` だけ**がその唯一の手段で、同一性を持つヒープ上の箱を複数の保有者で共有します。

```plew
val r = <Ref value=conn />   // conn を箱へ move（共有可変ハンドル）
r->state = Connected         // -> で中身にアクセス（書き込みは全保有者に見える）
val r2 = r                   // ハンドルをコピー＝同じ箱を共有（参照カウント++）
```

- `Ref[T]` は**祝福プリミティブ**（`Buffer` と同様、純 Plew では書けない。`String` はその安全床の上の純 Plew 値型＝`struct String { buffer: Buffer[U8] }` なので祝福不要）。コピーで共有＋retain し、最後の解放で中身の `deinit` を走らせる。
- **`.` は Ref ハンドル自体への操作、`->` は中身（pointee）への操作**（C と同じ）。共有変異が `->` で構文的に明示され、値の `.` と区別されます。これが値意味論の中で「共有が起きる唯一の場所」を見えるようにしています。
- **`val` な Ref 越しでも中身は変更できる**：Ref 束縛の `val`/`mut val` は **Ref 変数の再代入（`r = other`）** を gate するだけで、referent の変更（`r->x = v`・`inout fn` 呼び出し）は gate しません（Ref は共有可変が本分・Swift の `let class` と同じ）。値型の `val`（凍結）との非対称は、値 vs 参照の差を `Ref`＋`->` で可視化したものです。
- **`move fn`（消費メソッド）は Ref 越しに呼べない**：共有された referent を消費すると他の `Ref` が無効化される（use-after-consume）ため、`Ref` 越しは `fn`/`inout fn` のみ。共有資源の後始末は **`deinit`（最後の `Ref` 解放時）** に委ねます。明示消費や失敗し得る `close() -> Result` が要るなら、共有せず裸の [`unique`](#uniqueコピー不可型) 値（唯一所有）で `move fn` を呼びます（共有時は「誰が close エラーを受けるか」が原理的に不定なので呼べないのが正しい）。〔additive：唯一保持なら中身を取り出す `tryUnwrap() -> Optional[T]` を後付けし得る。〕
- **循環**は参照カウントで回収されないので、断ち切りに **`WeakRef[T]`**（非所有・指す先が消え得る）を使います。循環が生じ得るのは `Ref` グラフだけで、その自動回収（Ref グラフ限定のサイクルコレクタ）は将来 additive に足せる方針です（検出時は全ビルドで loud 報告＋メモリ回収・ただし循環メンバの `deinit` は走らせない＝下記 [deinit](#deinit) の契約外脱出。→ [非同期処理とメモリ管理 § メモリ管理（ARC）](../04-execution/14-concurrency.md#メモリ管理arc)）。それまでは `WeakRef` ＋リーク報告で運用します。**`WeakRef` は直接 deref できません（`w->x` は無い）** ── 指す先が生きている保証が無いからです。生存確認とアクセスは **`upgrade() -> Optional[Ref[T]]`** を通します（生きていれば強参照 `Ref`、消えていれば `None`＝可謬性が Optional で表に出る）：

```plew
val w = r.weak()                  // ダウングレード（. ＝ハンドル操作）
match w.upgrade() {               // Optional[Ref[T]]
    Optional.Some(value: val r) => r->x   // 強参照に格上げ済み＝生存保証・直接 deref 可
    Optional.None                  => skip()
}
```

- **再帰的な値型に `Ref` は要りません**：自己参照する `struct`/`enum`（木・連結リスト・AST 等）はコンパイラが自動でヒープ間接を挟みレイアウトを有限化します（値意味論のまま sendable）。`Ref` を書くのは**共有可変グラフ**（コピーで共有・循環あり）を意図したときだけです。詳細は [構造体と列挙型 § 再帰的な値型](../02-type-system/05-structs-enums.md#再帰的な値型)。

  `w->x` を「直接 `Optional` を返す deref」にしないのは、(1) **生存を固定**するため（一度 `upgrade` すれば強参照が指す先を生かし続ける。`w->x`/`w->y` を個別に可謬にすると間に最後の強参照が落ちて途中で死に得る）、(2) **書き込みが綺麗**（`r->x = v` は生存保証された強参照に書く。死んだ先への静かな no-op を避ける）、(3) **`->` の意味を一定に保つ**（常に「生存保証された `Ref` の deref」）ため。Rust の `Weak::upgrade()` と同じ。

## 再宣言（shadowing）

同名の `val`/`mut val` を**再宣言できます**（Rust と同じ・無制限）。再宣言は代入ではなく**新しい束縛**で、型も可変性も変えてよく、それ以降 `name` は**レキシカルに直近の宣言**を指します。外側スコープの名前を内側で覆う（shadowing）こともできます。

```plew
val config = load()                 // Optional[Config]
guard Optional.Some(value: val config) = config { panic "missing" }
// 以降 config は Config（元の Optional は退役）

val raw = read()                    // Bytes
val raw = parse(data: raw)          // ParsedData（同名で変換・型が変わる）
```

- **代入 `x = e` とは別物**：代入は既存の `mut val` 束縛を同じ型で書き換える。再宣言 `val x = e` は新しい束縛を作る（`mut` 不要・型変更可）。
- **RHS は旧束縛が見える環境で評価**します。`val raw = parse(data: raw)` の RHS の `raw` は旧束縛を指します。RHS 評価とその一時値の破棄が終わってから、新束縛を commit します。
- **覆われた `unique` 束縛は shadowing の commit 時に破棄**されます。再宣言により旧束縛は所有者として名指せなくなるため、その点が accessibility cliff です。旧 `unique` を借用した値が shadowing を越えて使われる場合はコンパイルエラーです。借用があるから旧値の drop を遅らせるのではなく、借用を旧値の寿命内に収めます。
- **覆われたコピー可能束縛の寿命は変わらない**：コピー可能な旧束縛は通常のローカル束縛と同じくスコープ離脱まで生存し、破棄はスコープ離脱時に束縛の逆順（新しい束縛が先）。名前で参照できなくなるだけで、寿命は普通のローカル束縛と同じです。
- `guard`/`match`/`if` のパターン束縛が同名で値を絞り込めるのは、この一般規則の帰結であって特別扱いではない（「絞り込みのときだけ同名可」という線引きは設けない）。
- 再宣言で覆って一度も読まれない前の束縛は、未使用束縛として診断（lint）で拾える（Rust 同様）。

## 代入と構造化代入

代入 `x = e` は既存の `mut val` 束縛を書き換えます。分解では各要素を **`val name`（新しい束縛）** か **bare `name`（既存へ代入・要 `mut val`）** で書き分け、混在もできます（`val`＝新規・bare＝既存は[再宣言](#再宣言shadowing)と同じ区別）。

```plew
// 基本代入（既存の mut val 束縛）
variable = expression
variable += expression          // -=, *=, /= も同様

// ラベル付きタプルの分解（フィールド名で対応）
(val x, val y) = point          // x, y を新規宣言
(x, val y) = point              // x は既存へ代入（要 mut val）・y は新規

// 構造体の分解（先頭に型名 → ブロックと曖昧にならない）
SomeStruct { val field1, val field2 } = someStruct       // punning（同名束縛）
SomeStruct { field1: val a, field2: val b } = someStruct // 別名

// 入れ子（不要フィールドは _ で破棄。全フィールドの明示が要る）
(name: val n, info: Person { name: _, val age }) = record
```

- 分解は**フィールド名で対応**します（位置ではない）。`(val x, val y)` は record の `x`/`y` を束縛し、書く順序は問いません。
- 文頭の `(` はラベル付きタプルの分解、型名始まりは構造体の分解、`{` 始まりは[ブロック](../03-expressions/11-control-flow.md)で、互いに曖昧になりません。
- **全フィールドを明示**します（束縛するか `_` で破棄）。未記載フィールドの暗黙無視も、残りを捨てる `..` もありません＝フィールド追加時に既存の分解がエラーになり取りこぼしを防ぎます。`_` は値を捨てる破棄パターンで `val` 不要、パターン位置ならどこでも書けます（詳細は → [パターンマッチング](../03-expressions/11-control-flow.md)）。

## 場所（place）越しの変更

代入の左辺・`inout` のレシーバ／引数になれるのは**場所（place）**＝`mut val` 束縛を根に、フィールドアクセスと添字を連ねたパスです（`x`・`x.field`・`arr[i]`・`a.b[i].c` 等）。`val` 束縛・関数の戻り値・一時値は場所ではなく、変更先にできません。

### get-modify-set 脱糖

ネストした場所への変更 ── 代入 `place = e`、複合代入 `place OP= e`、`inout` メソッド呼び出し `place.inoutMethod(...)`、`inout` 引数 `f(x: inout place)` ── は **read-modify-write** に脱糖します。場所の部分式（添字のキー・レシーバ）は**1 回だけ評価**し、添字の段は [`Index`](../03-expressions/12-operators.md#添字アクセス)（読み）と [`IndexSet`](../03-expressions/12-operators.md#添字代入indexset)（書き）、フィールドの段は load/store で、**内側から外へ書き戻し**ます。

```plew
mut val arr: Array[Counter] = [...]
arr[i].increment()                    // inout fn increment()
// ≈
val k = i
mut val tmp = arr.index(key: k)       // 読み（Index）
tmp.increment()                       // ローカルへの inout
arr.indexSet(key: k, value: tmp)     // 書き戻し（IndexSet）
```

`arr[i].field = x`・`arr[i].field += x`・ネスト `a.b[i].c = x` も同様に、添字段は `Index`/`IndexSet`、フィールド段は load/store で内側から組みます。これは [複合代入](../03-expressions/12-operators.md#複合代入演算子) の「場所は 1 回評価・`Index`→演算→`IndexSet`」を、一般の場所パスと `inout` レシーバ／引数へ広げたものです。

- **`IndexSet` を持たない添字（読み取り専用コレクション）越しには変更できません**（`Index` だけの型は読みのみ）。
- **値意味論ゆえメモリ安全**：触るのは切り離したコピーで、書き戻しは*その時点の*コンテナに対して行うので、変更中に内部で再確保が起きても dangling しません（C++ のイテレータ無効化が原理的に起きない）。

### in-place は観測不能な最適化

get-modify-set が**意味モデル**です。コンパイラは、(1) バッファが一意所有、(2) 変更窓で再入による無効化が起きない、(3) 場所が重ならない、を**静的に証明できる**ときに限り、コピーを介さず実メモリを直接書き換えてよい（観測挙動は不変・`Index`/`IndexSet` の副作用は保存）。証明できなければコピーへ退避します。**実行時のアクセス計装（Swift の動的 exclusivity 相当）は持ちません** ── 値のコピーは dangling しないので、計装が守るべきハザードがそもそも生じないためです。将来ボトルネックになれば「証明できない場合まで in-place 化する」方式（実行時トラップ付き）を additive に足せます（重ならない全プログラムの観測挙動は不変なので非破壊）。

### 重なる inout は禁止（排他）

`inout` は[排他](#アクセスモードborrow--inout--move)です。1 つの呼び出しの複数の `inout` 位置（レシーバ／引数）が**重なる場所**を指すことは**プログラムエラー**で、**必ず検出されます**——重なったまま実行されることはなく、未定義動作は存在しません（値世界の重なりはコンパイルエラー、共有世界の重なりは実行時 panic）。検査の粒度は **Swift の Law of Exclusivity と同等**（実測で照合済み）に定め、place の世界で二分します：

**値世界（`Ref` の deref を経由しない place）＝静的検査（コンパイルエラー）。**

- **同じ変数を root とする 2 つの `inout` place は、別フィールドへの分岐で互いに素と分かる場合を除きコンパイルエラー**：
  - 構文的に同一（`f(a: inout x, b: inout x)`）・包含（`a.merge(inout a.field)` の `a` と `a.field`・`xs` と `xs[i]`）→ エラー。
  - **同じ変数への添字を含むペアは、添字が異なって見えてもエラー**（`f(a: inout xs[i], b: inout xs[j])` は i・j の値によらず拒否）。添字の distinct 証明は行いません（Swift と同じ**変数粒度**）。要素対の操作はコレクション側のメソッドで提供します。
  - 同じ struct の**別フィールド**（`s.a` と `s.b`）→ OK（互いに素）。
- **異なる変数を root とする place 同士 → OK**（値意味論ではエイリアスし得ない・検査なし）。

**共有世界（`->`＝`Ref` の deref を跨ぐ place）＝ランタイム検査（panic）＋セル pin。**

- セルの同一性は実行時にしか分からない（`r1` と `r2` は同じセルを指し得る）ので、**呼び出し直前に place の一致をアドレスで検査**し、重なれば **panic**：
  - 同一セルの**同一フィールド**（`r1->n` と `r2->n`）→ panic。**別フィールド**（`r1->n` と `r2->m`）→ OK。
  - **コンテナ粒度**：同一セルのコンテナ値とその要素（`r1->xs` と `r2->xs[k]`）・同一コンテナの要素同士（`r1->xs[i]` と `r2->xs[j]`）は、添字を見ずに panic（Swift の配列プロパティと同粒度）。
  - 同一変数経由で重なりが静的に確定する形（`r->m()` レシーバと `inout r->m`・`r->xs` と `r->xs[k]`）は**コンパイルエラーに前倒し**します。
  - 検査は place を求める際に**評価済みの値だけ**で行い、引数の部分式を再評価しません（副作用を二重に発火させない）。
- **セル pin**：deref を跨ぐ place を `inout` で貸すとき、コンパイラは**呼び出しの間そのセルを保持（retain）**します。呼び出し中にセルへの最後の参照が消えても（`Ref` 変数の付け替え等）セルは呼び出し終了まで生き、**pointee の `deinit`・解放は呼び出しの後**に起きます（Swift が class の base を retain するのと同じ機構）。この保証により **`Ref` 変数とその pointee の place は同時に貸せます**——`f(r: inout r1, x: inout r1->n)` は合法で、`x` は**呼び出し時点で `r1` が指していたセル**の `n` に束縛され（place は 1 回評価）、`r` の付け替えと独立に書き込みはそのセルへ届きます。
