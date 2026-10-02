# 文字列補間・書式化の設計メモ

## 状態とレビュー範囲

実装前の設計メモ。合意済みの方針と、末尾の未確定事項を区別してレビューする。現行Stdは文字列を返す `Format.format(format: String)` のみで、以下のAPIと補間は未実装。レビュー後に仕様本文・実装へ反映する。

- 表示は出力先へ書く `Format` 一つに統一する。
- 出力先は `inout`、失敗は出力先の関連型 `Output.Error` で伝播する。
- 書式は型付きOptionsで渡し、省略時は失敗しない引数なしfactoryで構築する。
- 定数評価は一般の最適化であり、観測挙動を変えない。

## 補間の概要

文字列内で `{ 式 }` による変数展開が可能です。補間が要求する準拠は **`Format`** です。波括弧の中には値式を書けます（ブロック式・`match` 式を除く）。書式指定は文字列ではなく、対象型が定める **型付きOptionsの値**です。

```plew
val message = "Hello, {name}! You are {age} years old."
val formatted = "Vector = {vector: options}"
```

## Formatと出力先の契約

表示用traitは **`Format` 一つ**に統一し、出力先へ書く方式にします。文字列を返す方式との二重化、自動準拠、二重実装の衝突規則は設けません。以下は実装前の設計メモであり、現行Stdの実装を示すものではありません。

```plew
pub trait TextOutput {
    type Error

    inout fn write(text~: String) -> Result[(), Error]
}

pub trait Format {
    type Options: FormatOptions

    fn format[Output](
        output: inout Output,
        options: Options
    ) -> Result[(), Output.Error]
        where Output: TextOutput
}
```

- 出力先は呼び出し元の状態を更新するため `inout` で受けます。メソッドをgenericにし、一つのFormat実装で異なる出力先を扱います。
- 失敗は出力先の関連型 `Error` で表し、そのまま伝播します。固定の `FormatError` へ変換して詳細を失う設計にはしません。`Output.Error` は制約から出どころが一意なので `Output#TextOutput.Error` と明示する必要はありません。
- `format` 自身の失敗は、出力先から受け取ったエラーの伝播に限定します。子要素のformatを介した伝播も含みます。表示処理独自のエラーは生成せず、Optionsの構築・妥当性検査の失敗は通常のAPIと式で扱います。
- 表示実装は同じ出力先へ断片や子要素を順に書けます。`write(String)` は一時文字列のヒープ確保を要求しません。リテラルは定数領域を参照でき、値引数もデータの複製を意味しません。既存文字列の共有・短い文字列の格納・安全な確保除去などは内部実装の課題です。
- 数値を一度Stringへ変換してから書けば、その生成・コピーのコストは残り得ます。標準型の生成処理も含めて検証します。`writeScalar` や借用スライスを必須APIとして追加することは決めていません。
- `output.write("({self.x}, {self.y})")` は通常の意味では補間全体のStringを構築してから書きます。途中の副作用・panic・出力失敗との順序を変える逐次出力への変換はできません。一行で直接出力する補助APIは別途検討します。

## writeの共通契約と実装側の責務

`TextOutput` はgenericな表示処理が依存する最小限の契約だけを定めます。

- `Ok` は、渡されたテキスト全体について、その出力先が定義する書き込み処理が完了したことを示します。呼び出し側に未処理の残りの再送を要求しません。
- `Err` は書き込みの失敗を示し、表示処理は出力先のエラーをそのまま伝播できます。
- 完了がバッファへの格納・ファイルへの書き込み・配送などの何を意味するか、失敗前の部分出力が残るか、flushや永続化を別途必要とするかは、各出力先の実装者が定義・文書化します。traitはこれらを一律に保証しません。
- `Result[(), Error]` は処理量を返しません。部分処理量を返す低水準APIを使う場合、必要な反復は出力先側で行うか、処理量を返す別APIとして提供します。

## 型付きOptionsとデフォルト構築

`FormatOptions` は失敗しない引数なしfactoryを要求します。書式文字列を解析するfactoryではありません。

```plew
trait FormatOptions {
    factory()
}
```

- `{value}` はOptionsの引数なしfactoryでデフォルト値を生成します。
- `{value: options}` は通常の式 `options` を評価し、その値を渡します。期待型は対象型の関連型 `Options` です。
- [型省略構築](../spec/02-type-system/05-structs-enums.md#型を文脈から省く)も使えます。例えば精度を受け取るfactoryを持つOptionsなら `{value: <precision=2 />}`、デフォルト構築の明示は `{value: </>}` です。
- Optionsの構築は通常のfactory呼び出しです。省略時も呼び出しごとに評価し、副作用を保持します。値の妥当性検査や失敗し得る構築は通常のAPI・エラー処理で扱い、補間専用の文字列解析・失敗伝播を設けません。
- 外部設定などの文字列をOptionsへ変換するAPIは別途定義できます。補間は文字列パーサーの提供を要求しません。

## Stringのfactoryによる文字列化

明示的な文字列化はStringの無名factoryへ統一します。Format準拠型への共通 `toString` メソッドは提供しません。表示能力はFormat、Stringの生成はStringのfactoryが担当します。

```plew
val text = <String value=vector />
val detailed = <String value=vector options=<precision=2 /> />
```

- 入力ラベルは `value` とし、`From` の変換用ラベル `from` と区別します。
- Options省略時は、入力型の関連型 `Options` の引数なしfactoryを呼びます。明示Optionsには同じ関連型を期待型として与えます。
- factoryは失敗しない文字列構築先を用意して `value.format` を呼び、`Result[(), Infallible].value()`で成功を取り出し、完成したStringを返します。
- 補間は各要素に同じ文字列構築先を渡せます。各要素をこのString factoryで個別に文字列化してから連結することを要求しません。最終結果の格納領域は必要です。

宣言の概念形は以下です。factory自身の型引数・関連型のデフォルト構築を含む正確な宣言可否は確認が必要ですが、利用側APIと責務の方針は採用済みです。

```plew
pub impl String {
    factory[T](
        value: T,
        options: T.Options = </>
    ) where T: Format {
        // 文字列構築先を用意し、formatで書き込んで完成したStringを返す
    }
}
```

## 定数評価との関係

Optionsの構築は一般の最適化の対象であり、フォーマット専用のコンパイル時実行ではありません。安全に静的評価できる呼び出しは結果を埋め込み、実行時の評価を省いてよいものの、これは保証ではなく内部挙動として変わり得ます。`const` 宣言や、補間から利用するための定数評価契約は要求しません。

最適化は観測挙動を保持します。`print`・外部状態へのアクセス・暗黙の破棄を含めて安全性を証明できなければ通常の実行へ戻します。成功値だけでなく `Err` も事前計算可能な値ですが、最適化で判明した実行時エラーをコンパイルエラーへ昇格させず、エラー処理の実行条件・順序を保持します。異常終了する評価は原則として定数評価を断念します。

## 失敗しない出力先

作成不能な空enum `Infallible` を導入し、失敗しない文字列構築先の関連型を `type Error = Infallible` とする方針を採用する。一般のNever型や任意型への暗黙変換の導入とは分ける。空enumの構築不能性と、空matchの網羅性・後続への到達不能性を実装する必要がある。

`Result[T, Infallible]` の成功値は、専用の `value() -> T` で取り出す。Okから値を返し、Errのpayloadに対する空matchで存在しない経路を除去する。panicによるforce-unwrapにはしない。

特定のエラー型に対するimplは、既存のgeneric仕様に従って `impl[T] Result[T, Infallible]` と書く。`where E = Infallible` のような型等価述語は導入しない（[Where句](../spec/02-type-system/06-generics.md#where-句)）。

```plew
pub impl[T] Result[T, Infallible] {
    fn value() -> T {
        return match self {
            Result.Ok(value: val value) => value
            Result.Err(error: val error) => match error {}
        }
    }
}
```

一般の `Result[T, E]` には以下の対称なAPIを提供する。

- `fn ok() -> Optional[T]`：Okなら成功値をSomeで返し、ErrならNoneを返す。エラー情報を捨てる。
- `fn err() -> Optional[E]`：Errならエラー値をSomeで返し、OkならNoneを返す。成功値を捨てる。

`ok()`／`err()`は情報を落とす変換、`value()`は失敗不能なResultの成功値取り出しとして名前を分ける。エラー型によって同名メソッドの戻り型をOptionalから裸のTへ切り替えない。通常のResultには `value()` を提供せず、エラー型が失敗し得る型へ変われば呼び出しはコンパイルエラーとなる。

現在のgeneric型引数はコピー可能型に限定されるため、これらは通常の `fn` とする。元のResultを消費・変更せず、値意味論に従って結果を返す。unique型引数の対応は将来の `allowUnique` と合わせて検討し、今回の必須要件にはしない。`ok()`／`err()`は一般のResult APIであり、書式化のためにエラーを黙って捨てる用途には使わない。

## 未確定事項

以下は採用済みの関係とは区別し、実装前に契約を確定します。

- String factoryの型引数・関連型のデフォルト構築を含む宣言の対応確認、および関連型・factory要件を含むtrait宣言全体の整合確認。

## 後回しにする事項

補間を一行で直接出力する簡潔なAPIは初期実装の必須要件にせず、後回しにします。当面は同じ出力先へ `write` と子要素の `format` を順に呼ぶことで中間Stringを避けられます。通常の `write(String)` に補間結果を渡す場合は、Stringを構築してから渡す意味を維持します。

## 補間の構文

- 空のOptions指定 `{value:}` は許可しません。デフォルト指定は `{value}`、明示的なデフォルト構築は `{value: </>}` と書きます。
- **Options指定の境界**：補間対象を通常のPlew式として解析し、その外側の `:` をOptions式の開始として扱います。Optionsも通常の式として解析します。呼び出し引数・構築式・括弧・文字列リテラルなどの内部はそれぞれの構文として読み、単純に次のコロンや閉じ波括弧を検索して境界にしません。例えば `{calculate(scale: 2): <precision=2 />}` の `scale:` は引数ラベル、呼び出し後の `:` はOptionsの区切りです。
- **補間内に書けるのは値式だけ**で、ブロック式・`match` 式は書けません。複雑な計算は一度 `val` に束縛してから埋め込みます。
- 波括弧を**リテラルとして**出すには `{{` / `}}` と二重にします（`{{` → `{`、`}}` → `}`）。`\{` のようなバックスラッシュ系エスケープは波括弧には用意しません。
- 文字列リテラルは**生の改行をそのまま含められます**（複数行可）。一方で**行末の `\` は続く改行と次行の先頭空白を取り除き**、改行を入れずにソース上で折り返せます。

## 補間の評価順と寿命

補間は左から右へ、一つずつ表示を完了させます。補間対象とOptionsはそれぞれ一度だけ評価します。

```plew
"A{makeValue(): makeOptions()}B{nextValue()}C"
```

この例の順序は以下です。

1. 文字列構築先に `"A"` を書く。
2. `makeValue()` を評価する。
3. `makeOptions()` を評価する。
4. 得られた値の `format` を、得られたOptionsと共通の構築先で呼ぶ。
5. `"B"` を書く。
6. `nextValue()` を評価する。
7. その型のデフォルトOptionsを引数なしfactoryで構築する。
8. 得られた値の `format` を呼ぶ。
9. `"C"` を書き、完成したStringを返す。

全補間対象・Optionsを先に評価してから表示する方式にはしません。通常の式の `try` による早期リターンやpanicが起きれば、それ以降は実行しません。

一時値の寿命・破棄は既存のPlewの規則に従い、補間固有の短縮規則を設けません。一つの要素の表示完了は、その値を直後に必ず破棄することを意味しません。最適化は副作用・破棄・異常終了を含む観測挙動を保持する範囲で行います。

## 関連する構築仕様

[factoryと型省略構築](../spec/02-type-system/05-structs-enums.md#factory)の変更と組み合わせる。

| 明示した構築 | 期待型が一意な場合の省略 |
|---|---|
| `<Struct value=1 />` | `<value=1 />` |
| `<Struct />` | `</>` |
| `<Struct.named />` | `<.named />` |
| `<Enum.X />` | `<.X />` |

型は注釈・戻り型・引数位置から確定し、属性やメンバ名から検索しない。確定後にラベル・具体型でfactoryを選択する。無名factoryも通常の関数と同じ型オーバーロード規則を使う。詳細はリンク先を正本とし、型省略構築・一般のfactory選択の実装確認は残っている。

## レビューの確認点

- `TextOutput`の失敗とOptionsの通常の構築失敗が混同されていないか。
- 失敗しない出力先から、通常の補間・String factoryまで一貫した型付けができるか。
- 出力先の更新・寿命・副作用順序を通常のPlewの規則で説明できるか。
- 一時的なString値とヒープ確保を区別し、中間確保・コピーを省く実装経路があるか。
- 構築型の推論、コロンの構文境界、空指定の扱いが曖昧でないか。
