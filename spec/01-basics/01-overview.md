# 概要・基本構文

## 概要

**Plew — 唱えた通りに発現する。仕組みは問わない。**

Plew は、複雑な状態を持つクライアントと、それを支える API サーバを、握っている *感覚* はそのままに軽く書くためのフルスタックのアプリケーション言語です（モノレポでクライアント／サーバが同じモデル層を共有）。設計の拠り所は二軸です。**意味は唱えた通り** ── 何をして・何を呼び・どの型で・どこから来たか、は常に明示で、解決のときコンパイラは複数候補から忖度せず、一意に定まらなければエラーにします（補完はする・忖度はしない・曖昧はエラー）。**仕組みは問わない** ── その意味を *どう* 実現するか（脱糖・CoW・ARC・ヒープ確保・スケジューリング）は丸ごと引き受け、観測される挙動が唱えた意味から逸れない限り、裏では好きなだけ動いてよい。Rust が意味もコストも人間に握らせる（OS・組み込みで強い）のに対し、Plew はコスト軸だけを降ろします。型安全性、非同期プログラミング、パターンマッチング、独自の拡張システムを重視し、シングルプロセス・シングルスレッド＋イベントループ（JavaScript ライク）を基盤とします。値は**値意味論（CoW）**、メモリは **ARC（参照カウント）** で管理し、必要なときだけ opt-in の所有権（`unique`）とスレッド移送保証（`sendable`/`nonsendable`）、共有参照（`Ref`/`WeakRef`）を使います（→ [値・変数・所有権](03-values.md)、[非同期処理とメモリ管理](../04-execution/14-concurrency.md)）。

## 命名規則

- **型名**: PascalCase（例: `MyType`, `String`）
- **変数名・関数名・定数**: camelCase（例: `myVariable`, `calculateSum`, `maxRetry`）。定数（`val`/`export val`/`assoc val`）も同じく camelCase で、SCREAMING_SNAKE_CASE は使いません。

## キーワード

```
export, pub, get, type, val, mut, async, spawn, await, loop, break, continue,
give, return, panic, where, enum, struct, newtype, assoc, extension, impl, via, trait, fn,
factory, borrow, inout, move, unique, sendable, nonsendable, deinit, guard, while, for, in, if, else,
match, as, self, Self, extern, repr, import, part, with, try, true, false
```

`true` / `false` は真偽値リテラルです。`borrow`/`inout`/`move`（アクセスモード）・`unique`（所有権）・`sendable`/`nonsendable`（スレッド移送可能性）・`deinit`・generics の `allowUnique`/`sendable` は [値・変数・所有権](03-values.md) を参照。`mut` は記憶域可変性（`mut val`）専用です。

**文脈依存キーワード**：`optional` / `result` は [fallible factory](../02-type-system/05-structs-enums.md#失敗し得るファクトリfallible-factory) の前置修飾（`optional factory …` / `result[E] factory …`）の位置でのみ予約語として働き、**それ以外では通常の識別子**です。`result` は頻出する変数名なので予約語にせず、`val result = compute()` などは従来どおり書けます。

## コメント

```plew
// 行コメント

/*
 * ブロックコメント
 * 複数行にわたって記述可能
 */
```

ブロックコメントは**ネスト可能**です（`/* … /* … */ … */`）。

## 文の区切り

文は**改行で区切ります**（セミコロンは不要・書きません）。区切りは「物理的な改行」そのものではなく、**改行が文を終えられる位置に来たときだけ**効きます。具体的には、改行の直前のトークンが**文を終えられるもの**（識別子・リテラル・閉じ括弧 `)` `]` `}`・`return`/`break`/`continue`）であれば、その改行で文が終わります。

行末が**演算子・`,`・開き括弧**で終わっているとき、および **`(...)` / `[...]` の内側**では、改行は**無視**され文が継続します（暗黙の行継続）。一方 **`{ ... }` ブロックの内側では改行は有意**で、文の区切りとして働きます。

```plew
val x = 1 +
        2              // 行末が `+` なので継続：x == 3

val r = sum(
    a,
    b                  // (...) の内側は改行自由・末尾カンマも任意
)

print(r)               // 各行がそれぞれ 1 文
```

> このため、複数行にまたがる式は **`(...)` で囲む**か、**演算子を行末に置く**かのいずれかで継続させます。`} else {` のように、ブロックを閉じてすぐ続ける構文は同じ行に書きます。
