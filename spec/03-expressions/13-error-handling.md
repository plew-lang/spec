# エラーハンドリング

## try 式 - Rust の?演算子相当

`try`は特別なエラーハンドリングではなく、Result 型の早期リターンのシンタックスシュガーです。

**`try` は `Result` 専用で、`Optional` には効きません**（Rust の `?` が `Option` でも効くのとは異なる）。`Optional` を返す関数で `try optional` とは書けません。`Optional` の早期離脱・連結は `unwrapOr(fallback:)`・`?.`（オプショナルチェーン）・`match`/`guard` を使ってください。`Optional` を `Result` に持ち上げたい（欠落をエラーとして伝播したい）ときは、`match opt { … None => <Result.Err … /> }` 等で明示的に `Result` 化してから `try` します。`try` の経路をエラー型 1 本（`Result` の `From` 変換）に絞ることで、「欠落（`None`）」と「失敗（`Err`）」を構文レベルで混同しません。

```plew
fn parseAndProcess(input: String) -> Result[I32, String] {
    val number: I32 = try parseInt(text: input)  // エラー時は早期リターン
    val processed = number * 2
    return <Result.Ok value=processed />
}

// エラー型の変換（From トレイト実装時）
fn complexOperation() -> Result[Data, GeneralError] {
    val result1: Result[I32, ParseError] = tryParse()
    val value = try result1  // ParseError → GeneralError に自動変換
    // 処理続行
}
```

`try` は、`expr` のエラー型 `F` が関数の戻りエラー型 `E` と違う場合、Err 時に `<E from=f />`（`From` の無名 factory）を挿入して変換します（Rust の `?`＝`From::from` と同じ）。ターゲット `E` は関数シグネチャから既知なので、`GeneralError` が複数のソースエラーから変換可能（`From[ParseError]`・`From[IoError]` …）でも、ソース型でオーバーロード解決されます（→ [型変換と From トレイト](12-operators.md)）。`E == F` のときは変換不要でそのまま伝播します。

```plew
impl GeneralError as From[ParseError] {
    factory(from: ParseError) { /* ... */ }
}
impl GeneralError as From[IoError] {
    factory(from: IoError) { /* ... */ }
}
```

集約エラー enum（各ソースエラーを variant で保持する型）では、こうした variant ラップの `From` 実装を**メタプログラミングで自動生成**でき、手書きは不要です（Rust の `thiserror` 相当）。`try` 側の機構は単一の `From` のまま、boilerplate だけを生成で消します（→ [メタプログラミング](../04-execution/16-metaprogramming.md)）。

## try の優先順位（前置・後置チェーン全体に掛かる）

`try` は**前置演算子**で、直後の後置チェーン全体に掛かります。`try parse(input).validate()` は `try (parse(input).validate())` と解釈され、`parse(input).validate()` 全体（の `Result`）を評価してから早期リターン判定します。Rust の後置 `?`（`f()?.g()` ＝ `(f()?).g()`）とは逆なので、**Ok 値を取り出してから続きを呼ぶ**には括弧を使います：`(try parse(input)).validate()`。`try` は二項演算子より強く（`try f() + 1` ＝ `(try f()) + 1`）、後置より弱い位置にあります（→ [優先順位と結合性](12-operators.md#優先順位と結合性)）。

## force-unwrap は持たない

`Optional` / `Result` から中身を強制的に取り出す**後置演算子（`!` のような force-unwrap）は提供しません**。取り出しは `Optional` / `Result` の `unwrap` メソッドで行いますが、空・エラー時に実行時エラーとなるため**基本的に非推奨**です。通常は `match` / `guard` / `?.` / `unwrapOr(fallback:)` / `try` で分岐・伝播してください。

```plew
val v = maybeValue.unwrap()  // 空なら実行時エラー。基本は使わない
```
