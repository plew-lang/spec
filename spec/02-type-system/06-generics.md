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

**型パラメータの `[...]` には名前と能力マーカーを並べます**（`[T]`・`[T, U]`・`[sendable T]`・`[allowunique T]`）。**トレイト制約**はインラインに書かず、すべて [`where` 句](#where-句)に集約します（`fn`・`struct`・`enum`・`trait`・`impl` で共通）。役割分担は「**`[...]`＝名前＋能力マーカー（下記）／`where`＝トレイト制約**」です。

> ジェネリック境界 `where T: P`（同種・静的・全要求が使える）と、トレイトを値の型にする[存在型](08-traits.md#トレイトを値の型として使う存在型-any) `any P`（異種混在・動的・呼べるメンバに制限）は別物です。「1 種類でよい」ならジェネリック、「異種を混ぜたい」なら `any P` を使います。

## 型引数の能力マーカー（unique / allowunique / sendable）

型引数のコピー能力と送信可能性は別の軸です。`[...]`で宣言し、trait制約は`where`に書きます。

| 宣言 | 許す型 | 本体でのコピー |
|---|---|---|
| `[T]` | コピー可能型のみ | 可能 |
| `[unique T]` | unique型のみ | 不可 |
| `[allowunique T]` | コピー可能型とunique型 | 仮定できない |

送信可能性を要求する場合は`[sendable T]`・`[unique sendable T]`・`[allowunique sendable T]`と書きます。順序はコピー能力、sendable、名前です。無指定はnonsendableも許します。`sendable`はuniqueを受け入れる指定ではなく、`allowunique`も送信可能性を保証しません。能力はfn・型・trait・implの型引数で共通です。

traitのSelfと関連型にも独立したコピー能力を指定できます。適用範囲、継承、関連型束縛、存在型は[traitの能力](08-traits.md)に従います。制約によって型引数の宣言範囲が片側へ暗黙に縮む場合は、宣言側を`T`または`unique T`へ狭めるよう診断します。共通範囲が空でもコンパイルエラーです。`sendable`という別軸の明示的な絞り込みは、このコピー能力の縮小と区別します。

### 所有モードとコピー可能な具体化

具体的なコピー可能型や通常の`[T]`に対して、意味のない`borrow`・`move`を引数宣言・明示操作・selfモードへ書くことはエラーです。可変アクセスの`inout`はコピー可能型にも意味があります。

一方、`unique`になり得る`allowunique`本体では、共通の所有規則を記述するため`borrow`・`move`を許します。コピー可能な型へ具体化した場合、検査済みの借用／消費契約を通常の値渡しへ適応します。その具体化を理由に宣言本体を再診断しません。uniqueな具体化では元のborrow／moveを維持します。呼び出し側で既にコピー可能と分かっている値に明示moveを書くことは、この適応とは別であり拒否します。

```plew
fn inspect[allowunique T](value: borrow T) { /* 共通能力の範囲で検査 */ }
fn consume[allowunique T](value: move T) { /* 同上 */ }
```

本体はコピーを仮定せず、通常の終了時破棄、明示move、借用／inoutの区間、アクセス衝突を検査します。コピー可能な型で偶然成立する操作をuniqueな型へ持ち込みません。型引数・self・関数型内の所有モードにも同じ適応を行います。

所有local・消費引数は、合法なmoveで所有権を渡した後、未所有のままスコープ／関数を終了できます。一方、呼び出し元へ返すinoutの書き戻し先と、生存する共有セルは初期化済みの値を維持しなければなりません。move後に空のまま返すことや共有先からのmove-outを、所有localと同じ扱いにはしません。置換には[replace](../01-basics/03-values.md#汎用dropとreplace)を使います。

trait要求と選択する実装の所有モードは、上記のコピー可能型への適応後に一致しなければなりません。borrowをmoveで代用するなど、モード差を任意のadapterで補うことはしません。引数・selfの所有モードは関数型の同一性にも含め、異なるモード間の暗黙変換は認めません。

### autounique

型引数がuniqueになり得る名目型は、次の三つを区別します。

| 宣言 | 型自身のコピー能力 |
|---|---|
| `struct` / `enum` | 常にコピー可能。uniqueを値で所有できない |
| `unique struct` / `unique enum` | 常にunique |
| `autounique struct` / `autounique enum` | 所有するフィールド・payloadの能力に従う |

```plew
autounique enum Optional[allowunique T] {
    Some(value: T)
    None
}
```

`autounique`は明示宣言です。すべてのvariant・所有フィールドを対象とし、一つでもuniqueを所有すれば外側もuniqueになります。現在のvariant、配列の長さ、実行時の排他的な保持状態などで型のコピー可否を変えません。`Ref[T]`などコピー可能なハンドルはTがuniqueでもコピー可能であり、phantom型引数もunique性を伝播させません。Promiseは物理的なRC表現とは別に結果Tを所有する契約を持ち、[autouniqueと消費await](../04-execution/14-concurrency.md)に従います。

宣言が許す型引数の全範囲で常にコピー可能、または常にuniqueとなる型への`autounique`指定は宣言エラーです。それぞれ通常の型宣言、または`unique`宣言を要求します。判定は宣言条件と所有構造に基づき、現在の利用例・生成済みの具体化の列挙によって変えません。適切なautounique宣言をuniqueな型引数で具体化することは合法です。

```plew
autounique struct Shared[allowunique T] {
    val value: Ref[T]
} // Error: 常にコピー可能

autounique struct OnlyUnique[unique T] {
    val value: T
} // Error: 常にunique
```

通常の型を主語とするimplでは、`impl[allowunique T] unique Box[T]`のようなSelf能力修飾は書けません。Selfの能力は型宣言・所有構造とimplの型引数条件から導きます。traitを主語とする提供implの能力修飾は[trait](08-traits.md)で定めます。条件付きdeinitは[deinit](../01-basics/03-values.md#deinit)に従い、deinitの有無から型のunique性を導きません。

### 能力条件付き候補と重複判定

診断では、具体的な重複を証明した場合と、限定した判定範囲では排他性を証明できない場合を区別します。どちらも拒否しますが、未証明を重複の証明として報告しません。

コピー可能型用・unique型用に、同じ名前の関数やimpl内のメソッド・factoryを排他的に定義できます。同じ呼び出しに複数の候補が適用される定義を、後から優先順位で選ぶことはしません。Selfと関連型を含む能力条件、型の等式、型宣言から保存したautounique導出条件の単純な組合せで、重複がないと証明できる場合だけ受理します。証明不能なら拒否し、任意のtrait制約を解く一般の論理solverを要求しません。

能力導出・能力判定に必要な型の正規化・能力条件の判定は、コンパイラ実装が定める固定の解析上限内で行います。**無限に続くことの証明や、そのための特別な検査は要求しません。** 有限でも上限を超える要求は、解析上限に達した旨を示して拒否できます。上限到達を、型が矛盾している証明・候補が重複する証明・コピー可能／uniqueという判定の代用にはしません。通常の再帰型や相互再帰は、同じ問題を繰り返し展開せずに扱えます。上限値は言語の数値定数として保証せず、初期提供では利用者が変更する構文・APIを設けません。

```plew
fn handle[T](value: T) {}
fn handle[unique T](value: move T) {}
```

allowuniqueなgenericから、このような排他的候補群を呼べます。ただし宣言の全範囲を覆い、各具体化で一意に決まり、ラベル・引数／結果型・所有モード・既定値による呼び出し形などが適応後の共通契約として成立することが必要です。一方の候補しか存在しない場合や、具体化して初めて任意のoverload探索をやり直さなければ決まらない場合は拒否します。

コンパイラはgeneric検査時に共通契約と候補・選択条件・適合根拠を保存し、具体化時はその条件を適用します。後段で名前・型名を再検索して別の候補を選びません。一つのtrait要求を`via`でこの候補群へ結び付けることも、[準拠の規則](08-traits.md#トレイト準拠と-via)に従って認めます。一つの型・trait instanceに対する準拠宣言そのものを、コピー能力だけで複数に分割する機能は提供しません。

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

型パラメータへの制約を表現します。各述語は **`型: 制約`** の形です（`T: Eq + Format`、関連型射影への制約 `T#Iterator.Item: Format` も書けます）。関連型射影は [トレイト](08-traits.md#関連型associated-type) の通り、正規形 `T#Trait[Args].Item` を使います。trait が位置型引数を持つなら `Args` は必須で、引数なしなら省けます。`T.Item` は完全な trait instance が一意な場合だけの短縮です。

```plew
fn func[T](param: T) where T: Eq + Format {
    // 処理
}
```

- **特定の具体型に対する実装は型の位置に直接書きます** — `impl MyStruct[I32] as SomeTrait { }`（`where T = I32` のような型等価述語は持ちません）。
- 関連型を特定の型に束ねるのは `where` の述語ではなく、**トレイト名の `[...]` 内の関連型束縛**です（`T: Iterator[Item = I32]`。→ [トレイト](08-traits.md)）。

## ジェネリック本体の型検査（宣言時・abstract）

ジェネリック関数・メソッド・提供メソッドの本体は、**具体instanceではなく、宣言のboundに基づいて抽象的に型検査されます**。`check`・`build` は未使用本体も検査し、`run` は必要になった宣言本体の抽象検査を実行前に行います。「宣言時」は検査の型環境を指し、すべてのコマンドで起動前の全体検査を要求する意味ではありません（[コマンド別の検査契約](../04-execution/15-modules.md#runcheckbuild-の検査と実行)）。具体型で操作が偶然成立することを宣言の正当性の根拠にはしません。

型パラメータ `T` は **不透明な型**として扱われ、`T` の値に対してできる操作は **`where T: Trait` で宣言した制約が提供するものに限られます**。

- `T` の値へのメソッド呼び・演算子・比較は、その能力を **bound が正当化していなければエラー**です（`a + b` には `where T: Add`、`a == b` には `where T: Eq`、`a < b` には `where T: Ord` が要る）。bound のトレイトが継承する上位トレイト（`trait Sub: Super`）の能力も使えます。
- 型パラメータ `T` は **それ自身とだけ同じ型**で、具体型や別の型パラメータ `U` とは別の型です。`fn f[T](a: T) -> I64 { return a }` は `a`（型 `T`）を `I64` 位置で返すのでエラー（`T` が `I64` とは限らない）。`val y: I64 = a` も同様。
- インスタンス化は意味を変えません（単相化は発現＝コード生成の都合で、観測される型規則は宣言時の抽象検査が定める）。具体型への置換時に新たな制約違反があれば、それは **呼び出し位置の `where` 適合検査**で別途エラーになります。

> 拠り所：意味は唱えた通り（曖昧・無根拠な操作は宣言のboundに基づいて弾く）。`check`・`build` では、一度もインスタンス化されないgeneric本体の型エラーも検出します。`run` での検査延期は、この型規則を弱めません。
