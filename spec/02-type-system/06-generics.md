# ジェネリクス

基本的に Rust と同様に不変（invariant）です。型パラメータは `[...]` で宣言します。

```plew
struct Container[T] {
    val value: T
}

fn process[T](container: Container[T]) where T: Format {
    // 処理
}
```

**型パラメータの `[...]` には名前と能力マーカーを並べます**（`[T]`・`[T, U]`・`[sendable T]`・`[allowUnique T]`）。**トレイト制約**はインラインに書かず、すべて [`where` 句](#where-句)に集約します（`fn`・`struct`・`enum`・`trait`・`impl` で共通）。役割分担は「**`[...]`＝名前＋能力マーカー（下記）／`where`＝トレイト制約**」です。

> ジェネリック境界 `where T: P`（同種・静的・全要求が使える）と、トレイトを値の型にする[存在型](08-traits.md#トレイトを値の型として使う存在型-any) `any P`（異種混在・動的・呼べるメンバに制限）は別物です。「1 種類でよい」ならジェネリック、「異種を混ぜたい」なら `any P` を使います。

## 型引数の能力マーカー（allowUnique / sendable）

型引数が**どんな型を取れるか**は `[...]` の能力マーカーで調整します（トレイト制約＝`where` とは別軸）。既定 `[T]` は **common case**＝**`unique` でない（コピー可能）かつ nonsendable 許容**で、無注釈で済みます。逸脱するときだけマークします：

- **`[allowUnique T]`**：`unique` 型も admit する（widen）。`T` をコピーできなくなるので本体は借用／move で扱う。**v1 では未実装の将来機能** ── コア型（`Optional`/`Array`/`Promise`/`JoinHandle` 等）も含め `allowUnique` を入れずコピー可能な型に限定し、`unique` を generic に入れるのは常に [`Ref`](../01-basics/03-values.md#ref--weakref共有可変) 包み（`Optional[Ref[P]]`）。
- **`[sendable T]`**：`T` を sendable 型に限定する（narrow）＝`spawn` する generic やスレッド間チャネルへ送る generic で、移送可能性を明示的に要求する（→ [非同期処理とメモリ管理](../04-execution/14-concurrency.md)）。存在型では [`any sendable P`](08-traits.md#存在型の-sendability) は満たすが、保証を消去した無印 `any P` は満たしません。`spawn` を使わない大半のコードでは不要。

非対称（`allowUnique`＝admit／`sendable`＝restrict）は「**既定＝common case・逸脱だけ注釈**」の帰結です（`unique` は generics で稀なので既定除外、nonsendable は日常的な単一スレッドコードで許容）。**条件付き unique**：`allowUnique` なコンテナが `T` を値で所有すると、`T` が `unique` のときコンテナ自身も `unique` になります（**自動導出・`reunique` 等の宣言は不要**。`unique` 性は型引数から見えるので silent でない）。**条件付き sendability**：型引数をフィールド/payloadとして所有する generic 型は、実際の型引数が sendable なら sendable、nonsendable なら自動的に nonsendable になります。表現に現れない phantom 型引数はこの導出に影響しません。

## impl の型パラメータ

`impl` 自身が導入する型パラメータは、型名の前に **`impl[...]`** で宣言します（`fn f[...]` と同形）。`impl[T]` で束縛した名前を、self 型・トレイト・`where` の中で**使います**。

```plew
impl[T] Container[T] { /* … */ }                        // self が generic（inherent）
impl[T] Container[T] as Drawable where T: Format { }    // 条件付き準拠
impl[T] Vector as Mul[T] where T: Scalar { }             // T はトレイト引数だけに現れる
impl Container[I32] as SomeTrait { }                     // 具体インスタンス化（バインダなし）
impl Celsius { /* … */ }                                 // 非ジェネリックはブラケット不要
```

- `impl[T]`（束縛）と `Container[T]`（使用）を分けるので、`impl Container[I32]`（具体型に対する実装）と曖昧になりません。
- 型パラメータがトレイト引数・`where`・メソッドにしか現れない場合も、`impl[...]` が置き場になります（self 型のブラケットに頼らない）。

## 呼び出し位置での型引数の明示

通常は型引数を**推論**しますが、推論が効かないときは呼び出し位置でも `name[TypeArgs](args)` と明示できます。

```plew
val n = parse[I32](text: input)   // 型引数 I32 を明示
```

ブラケットの中身が**型なら型引数の適用、値なら[添字アクセス](../03-expressions/12-operators.md)**として解釈します（Go と同じ判別）。型は PascalCase・値（変数）は camelCase なので、`f[I32](…)`（型引数）と `arr[i]`（添字）は衝突しません。

## Where 句

型パラメータへの制約を表現します。各述語は **`型: 制約`** の形です（`T: Eq + Format`、関連型射影への制約 `T#Iterator.Item: Format` も書けます）。関連型射影は [トレイト](08-traits.md#関連型associated-type) の通り、正規形 `T#Trait.Item` を使い、`T.Item` は源トレイトが一意な場合だけの短縮です。

```plew
fn func[T](param: T) where T: Eq + Format {
    // 処理
}
```

- **特定の具体型に対する実装は型の位置に直接書きます** — `impl MyStruct[I32] as SomeTrait { }`（`where T = I32` のような型等価述語は持ちません）。
- 関連型を特定の型に束ねるのは `where` の述語ではなく、**トレイト名の `[...]` 内の関連型束縛**です（`T: Iterator[Item = I32]`。→ [トレイト](08-traits.md)）。

## ジェネリック本体の型検査（宣言時・abstract）

ジェネリック関数・メソッド・提供メソッドの本体は、**インスタンス化を待たず、宣言時に一度だけ抽象的に型検査されます**（C++ テンプレートのように「使われたときに初めてエラー」ではありません）。

型パラメータ `T` は **不透明な型**として扱われ、`T` の値に対してできる操作は **`where T: Trait` で宣言した制約が提供するものに限られます**。

- `T` の値へのメソッド呼び・演算子・比較は、その能力を **bound が正当化していなければエラー**です（`a + b` には `where T: Add`、`a == b` には `where T: Eq`、`a < b` には `where T: Ord` が要る）。bound のトレイトが継承する上位トレイト（`trait Sub: Super`）の能力も使えます。
- 型パラメータ `T` は **それ自身とだけ同じ型**で、具体型や別の型パラメータ `U` とは別の型です。`fn f[T](a: T) -> I64 { return a }` は `a`（型 `T`）を `I64` 位置で返すのでエラー（`T` が `I64` とは限らない）。`val y: I64 = a` も同様。
- インスタンス化は意味を変えません（単相化は発現＝コード生成の都合で、観測される型規則は宣言時の抽象検査が定める）。具体型への置換時に新たな制約違反があれば、それは **呼び出し位置の `where` 適合検査**で別途エラーになります。

> 拠り所：意味は唱えた通り（曖昧・無根拠な操作は宣言時に loud に弾く）。dead な（一度もインスタンス化されない）ジェネリックでも本体の型エラーは検出されます。
