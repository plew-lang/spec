# Plew プログラミング言語仕様書

**Plew — 唱えた通りに発現する。仕組みは問わない。**

Plew は、複雑な状態を持つクライアントと、それを支える API サーバを、握っている *感覚* はそのままに軽く書くためのフルスタックのアプリケーション言語です（モノレポでクライアント／サーバが同じモデル層を共有）。設計の拠り所は二軸です。**意味は唱えた通り** ── 何をして・何を呼び・どの型で・どこから来たか、は常に明示で、解決のときコンパイラは複数候補から忖度せず、一意に定まらなければエラーにします（補完はする・忖度はしない・曖昧はエラー）。**仕組みは問わない** ── その意味を *どう* 実現するか（脱糖・CoW・ARC・ヒープ確保・スケジューリング）は丸ごと引き受け、観測される挙動が唱えた意味から逸れない限り、裏では好きなだけ動いてよい。Rust が意味もコストも人間に握らせる（OS・組み込みで強い）のに対し、Plew はコスト軸だけを降ろします。型安全性・非同期プログラミング・パターンマッチング・独自の拡張システムを重視し、シングルプロセス・シングルスレッド＋イベントループ（JavaScript ライク）を基盤とします。値は値意味論（CoW）、メモリは ARC（自動参照カウント）で管理し、必要なときだけ opt-in の所有権（`unique`）とスレッド移送保証（`sendable`/`nonsendable`）を使います。spawn・チャネルを介して複数スレッドから共有され得る allocation だけは whole-program 解析により最初から atomic カウントとなり、境界でのディープコピーや動的昇格は行いません。

本仕様書はトピック別に分割し、論理依存順に **4 部**へまとめて `spec/` 配下に置いています。

## 目次

### 第Ⅰ部 言語の基礎

1. [概要・基本構文](spec/01-basics/01-overview.md) — 概要、命名規則、キーワード、コメント
2. [基本型](spec/01-basics/02-basic-types.md) — プリミティブ型、数値リテラル、文字列（不変・UTF-8・変数展開）、複合型（配列／辞書／ラベル付きタプル／レンジ）
3. [値・変数・所有権](spec/01-basics/03-values.md) — 値意味論（CoW）、val／mut val、アクセスモード（borrow／inout／move）、unique、sendable／nonsendable、Ref／WeakRef、deinit、再宣言（shadowing）、構造化代入
4. [関数](spec/01-basics/04-functions.md) — 関数宣言、引数ラベル、無名関数（クロージャ）、sendable fn

### 第Ⅱ部 型システム

5. [構造体と列挙型](spec/02-type-system/05-structs-enums.md) — 構造体／列挙型の宣言、フィールド統一原則、Optional／Result、インスタンス生成（JSX／factory）、メンバの可視性
6. [ジェネリクス](spec/02-type-system/06-generics.md) — 型パラメータ、impl[T]、where 句
7. [メソッドと impl](spec/02-type-system/07-methods-impl.md) — impl、メソッド、関連関数／関連値、メソッドのオーバーロード、無名 impl のコヒーレンス
8. [トレイト](spec/02-type-system/08-traits.md) — 定義（要求＋提供メソッドはベア `impl Trait`）、関連型、継承（supertrait）、準拠と via、値型としての利用（存在型 `any`）、標準トレイト
9. [拡張システム](spec/02-type-system/09-extensions.md) — extension、`#` による拡張適用、orphan rule、ビューの型扱い（暗黙キャストなし）
10. [newtype（名目型）](spec/02-type-system/10-newtype.md) — 実装の継承と Self 置換、as による再タグ

### 第Ⅲ部 式・制御・エラー

11. [制御構造](spec/03-expressions/11-control-flow.md) — ブロックと give、条件分岐、条件チェーン（束縛つき条件）、パターンマッチング、ループ、ガード、panic
12. [型変換と演算子](spec/03-expressions/12-operators.md) — From トレイト、演算子システム、等価／順序（Eq／Ord）、論理結合子（&&／||）、オプショナルチェーン（Chain）、nil 合体（Coalesce）
13. [エラーハンドリング](spec/03-expressions/13-error-handling.md) — try 式

### 第Ⅳ部 実行モデルとツール

14. [非同期処理とメモリ管理](spec/04-execution/14-concurrency.md) — async/await、spawn（spawn fn）、静的な atomic RC 選択、境界規則（借用・Ref・トップレベル状態は越えない）、チャネル、実質 race-free
15. [モジュールシステム](spec/04-execution/15-modules.md) — トップレベルとモジュールスコープ（ユーザーは ambient 定義を作れない・トップレベル変数・言語アイテムは import 不要）、import、export、part、パッケージ、extern
16. [メタプログラミング](spec/04-execution/16-metaprogramming.md) — `@[...]` 組み込みディレクティブ、ユーザー定義メタプログラミング（方針転換中）
17. [パッケージ](spec/04-execution/17-packages.md) — マニフェスト（`Plew.toml`）、依存（git/path・桁数バージョン）、依存解決（最新互換・複数版共存・phantom 禁止）、ロックファイル、ソース配布＋消費側ビルド、ネイティブ依存（C/Rust/pkg-config 祝福）

### 付録

18. [サンプルコード](spec/05-appendix/18-examples.md)

---

この仕様書は Plew プログラミング言語の現在の設計を反映しています。
