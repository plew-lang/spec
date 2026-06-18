# メタプログラミング

> **状態: 方針確定・基盤実装済（M1＋共有パーサ統合済）。** 当初の「閉じたメタプログラミング」（言語提供の `@[...]` のみ）から、**ユーザー定義可能なメタプログラミング**へ転換し、その骨格（入出力モデル・出力先・取り込み方法・実行モデル）を確定した。本章はその確定方針の正典。**入力はかつて `TokenStream` だったが、AST 構文ライブラリを共有する設計（下記）に伴い `AST`（構文木＝`TopItemAst`）に確定**した。**実装済**＝`plewc --gen` モード＋共有 `@Plew/Syntax`（レクサ＋値ツリー AST＋宣言/本体/トップレベルパーサ `parseProgramAst`/`parseItem`）＋ローダ自動 part＋`plew gen` ランナーで、**マクロが注釈対象項の実構造（型名・フィールド名/型/可視性・enum バリアント/ペイロード・fn シグネチャ＋本体・impl のメンバ・trait の要求）を読んで生成し、`@[Name(label: expr)]` のディレクティブ引数も `self` から読める**まで貫通する（`@[Name]`→`<Foo>.gen.pw`→通常ビルドで取り込み）。**コンパイラ本体もこの共有パーサで構文解析する（真の 1 AST＝重複パーサなし）**＝マクロが見るのはコンパイラが見た構文そのもの。注釈対象は struct/enum/fn に加え **impl/trait** も可。`quote` 等 authoring 層・生成コマンド設定など**細部はなお未決**で末尾に残す。実装の段取り・実行系の具体は [claude/metaprogramming-architecture.md](../../claude/metaprogramming-architecture.md)。

## 組み込みディレクティブ `@[...]`

型宣言に付与し、定型実装を合成させる属性です（Rust の `derive` 相当）。

```plew
@[Eq, Hash]
struct Point {
    val x: I32
    val y: I32
}
```

組み込みディレクティブ（暫定リスト。詳細は今後決定）:

- `Eq`, `Hash` — 対応する `Eq` / `Hash` トレイトの実装を合成（`Clone` は無い＝値意味論で複製不要）。
- `Encode`, `Decode` — シリアライズ用の実装を合成。
- enum 専用 `All` — 全バリアントを列挙する `Enum.all()` を合成。
- enum 用（集約エラー、名称未定） — 各 variant が単一ペイロードでソースエラー型を保持する enum に `From[各ソース]`（variant ラップ）を合成し、`try` のエラー変換の boilerplate を消す（Rust の `thiserror` 相当）。`try` 側の機構は単一の `From` のまま（→ [エラーハンドリング](../03-expressions/13-error-handling.md)）。

> ディレクティブは**コードを合成する derive 専用**です。既定 factory（memberwise）の合成・公開は derive ではなく**コア言語の構築意味論＋素の構文**で扱います（フィールド既定値と `pub impl { factory }` 公開 → [既定 factory（memberwise）とフィールドの既定値](../02-type-system/05-structs-enums.md#既定-factorymemberwiseとフィールドの既定値)）。可視性制御のような言語機能を `@[...]` の祝福ディレクティブにしないことで、ディレクティブ名前空間にはユーザー定義可能な derive だけが残ります。

> 命名規則: ディレクティブ名は後述の derive インターフェース（`Derive` / `ParameterizedDerive`）を実装する**トレイトまたは構造体**で、組み込み・ユーザー定義を問わず **PascalCase** です（`Eq` / `Hash` / `Encode` / `Decode` / `All`）。一方、ディレクティブが**生成する**メンバ（例: `Enum.all()`）は通常の関数・変数なので camelCase です。

## ユーザー定義メタプログラミング

ユーザーが独自のコード生成を定義できます。骨格は以下の通り確定。

### マクロ = derive インターフェースの実装・入力は `AST`／出力は `String`

**derive 可能性 = derive インターフェースを実装していること。** `@[X]` が有効なのは、**`X` が derive インターフェース（下記 `Derive` / `ParameterizedDerive`）を実装しているとき**だけ。`X` は **トレイトでも構造体でもよい**——`@[Hash]` の `Hash` は**トレイトそのもの**（`impl Hash as Derive` で「derive 可能」を後付け）、`@[All]` の `All` はトレイトを持たない**構造体**（`impl All as Derive`）。いずれも生成メソッド（要求＝`derive(input: TopItemAst) -> String`）を提供し、**注釈対象項の構文木を受け取り生成 Plew ソースを `String` で返す**。`Derive` 系トレイトと `AST` 型は**構文ライブラリ**（当面 `@Plew/Syntax`・将来は外部共有パッケージ＝後述）が提供します。**入力型 = `TopItemAst`**（注釈対象のトップレベル項の値ツリー＝`Decl`〔struct/enum/fn〕・`Impl`・`Trait`・… のタグ付き union）。マクロは `match` で対象の種別に分岐する＝対象が何かを**明示**して扱う（struct 専用 derive は `Decl` だけ扱い他を弾く）。これは「コンパイラとマクロが唯一の同一 AST を読む（1 AST 原則）」の帰結で、マクロ専用の縮小 AST を作らない。

**derive インターフェースは「設定（引数）の有無」で 2 つに分かれる。** derive の唯一の本質的な軸は設定を持つかどうかで、それが invocation 構文に直結する：

- **設定なし → `Derive`**（要求 `assoc fn derive(input: TopItemAst) -> String`・`self` 無し）。**bare `@[X]`** で呼ぶ。Eq/Ord/Hash/All 等、derive の大半。`Self` を含まない `assoc fn` なので、**トレイト**（`impl Hash as Derive`）でも**フィールド無し構造体**（`impl All as Derive`）でも実装でき、インスタンスを作らずに `X.derive(input)` を一意に呼べる。
- **設定あり → `ParameterizedDerive`**（要求 `fn derive(input: TopItemAst) -> String`・`self` = 設定構造体）。**`@[X(args)]`** で呼ぶ（`@[X()]` のように**括弧を強制**）。設定は構造体の型付きフィールドで、`self` から読む（`@[Builder(prefix: "with")]` 等）。

**`()` 規約が構文に機構を持たせる。** **bare `@[X]` は `Derive`（assoc）を、括弧つき `@[X(args)]`／`@[X()]` は `ParameterizedDerive`（インスタンス構築）を指す。** 引数を一切取らない構造体 derive も括弧を付けて `@[A()]` と書く（`()`＝「インスタンスを構築する」の合図）。bare `@[A]` と書きたければ `A` は `Derive`（assoc）を実装する。これにより、`@[X]` を見ただけでどちらの機構かが一意に決まる。

```plew
// 設定なし derive：トレイトが自身の deriver を持つ（`struct Hash` を作らない＝衝突しない）
pub impl Hash as Derive {
    assoc fn derive(input: TopItemAst) -> String {
        match input {
            TopItemAst.Decl(val d) => { /* d.name / d.fields … から `pub impl T as Hash {…}` を生成 */ return "" }
            _ => { return "" }
        }
    }
}
@[Hash]                          // bare = Hash.derive(input) を呼ぶ（インスタンス不要）
struct Point { }

// 設定あり derive：構造体のフィールドが設定スキーマ・`@[X(args)]` で構築
struct Builder { pub val prefix: String }
pub impl Builder as ParameterizedDerive {
    fn derive(input: TopItemAst) -> String { /* self.prefix を使って生成 */ return "" }
}
@[Builder(prefix: "with")]       // = Builder { prefix: "with" } を構築して .derive(input)
struct Config { }
```

注釈は struct / enum / fn だけでなく **`impl` ブロック・`trait`** にも付けられる（`@[Name] impl T { … }`・`@[Name] trait U { … }`）＝そのとき `input` は `TopItemAst.Impl` / `TopItemAst.Trait`。**`extern` ブロック内の不透明 lang-item / FFI 型**（`extern(plewIntrinsic) { @[Name] struct I8 }`・`extern(c) { @[Name] type … }`）にも付けられ、`input` は本体なし `struct` の `TopItemAst.Decl`（`d.name` ＝その型名）。これでコア床のプリミティブ型に derive で実装を生やせる（例＝`@[IntTryFrom(sources: […])] struct I8` が整数 narrowing `TryFrom` witness 群を生成）。derive マクロ自身は `@Plew/Syntax`（→`@Std/Core`）に依存するので、その出力をコア床に取り込む循環は **生成物（`X.gen.pw`）をコミットする**ことで断つ（出荷物は生成された `impl` のみで構文ライブラリに依存しない）。なお derive がコア床の循環を踏むのは**生成物がツールチェーン自身のコンパイルに必要なとき**だけで（算術 witness はコアが内部使用するので循環＝独立したスタンドアロン生成器が要る）、`TryFrom` narrowing のようにコア／構文ライブラリが内部使用しないものは `@[...]` derive で生成できる。

**なぜこの 2 分割か（trait と derive の同名衝突の解消）。** Plew は **型とトレイトが同一名前空間**なので、Rust 流に「`trait Hash` ＋ derive 用 `struct Hash`」を共存させられない（Rust はマクロ名前空間と型名前空間が別なので可能だが、Plew には無い）。そこで **derive を必ず構造体にする前提を外し、トレイト derive はトレイト自身が `assoc fn derive` を持つ**ことで `struct Hash` を不要にした＝`Hash` は 1 エンティティ（トレイト）のまま衝突しない。`All`/`Builder` のような**トレイトを持たない** derive だけが構造体（対応するトレイトが無いので衝突もしない）。**却下案**：(A) Rust 流の別名前空間＝Plew に無く、Core 内でモジュールを分けて名前空間で逃がすのは不自然。(C) trait と derive を別名（`@[DeriveEq]` 等）＝同名前空間なら筋は通るが、共通ケース（設定なし derive が大半）に恒久的な命名負担が乗り「`Eq` の derive は何という名前か」が毎回問われ混乱する。**採用**＝設定なしを無印 `Derive`（assoc）に置くことで共通ケースに修飾子が要らず、命名問題そのものが消える。→ [claude/design-decisions.md](../../claude/design-decisions.md)「trait と derive の同名」。

**入力 `AST`／出力 `String` の非対称は、エラーをどこに出すかと一致している。**

- **入力＝`AST`（span 付きノード）**：マクロが入力を解析して拒否する（「この型は Hash 不可」等）とき、**ユーザーの元ソース位置を指す**必要がある。AST ノードは span を保持し、その span は**原本座標**（`<Foo>.pw` の位置）なので、入力エラーはユーザー元位置を指せる。
- **出力＝`String`**：生成コードのエラーは**生成ファイル `<Foo>.gen.pw`（実ファイル）の普通のコンパイルエラー**で出るので、出力側に span は要らない（再コンパイルが位置を持つ）。`quote` のような埋め込み構文を作らず、生成は**普通の Plew 文字列・既存の補間 `"{…}"`** で組む（「Plew の中の Plew でないもの」を作らない・ハイライトもただの Plew）。

**ディレクティブ引数 = `ParameterizedDerive` マクロ struct のフィールド。** `@[Name(a: 32)]` は `Name { a: 32 }` を構築して `.derive(input)` を呼ぶ糖衣（`ParameterizedDerive`・instance）。マクロ struct のフィールドがそのまま設定スキーマで、`derive` は `self`（構築済み設定）からそれを読む。既存の labeled args / 構築構文を再利用し、新概念を作らない。**設定なし derive（`@[Eq]`/`@[Hash]` 等）は bare＝`Derive`（assoc）で構築を伴わない**（上の `()` 規約）。

**ディレクティブ引数はマクロ struct の[ファクトリ可視性](../02-type-system/05-structs-enums.md#既定-factorymemberwiseとフィールドの既定値)に従う。** `@[Name(a: 32)]` は `Name` をハーネス（生成コマンド側＝マクロ定義モジュールの外）が構築するので、これは**外部からの `<Name a=32 />` 構築と同じ規則**に服す：マクロ struct は**公開ファクトリ（`pub impl Name { factory }`）を露出し、引数に渡すフィールドが `pub`** でなければならない（既定 factory は非公開なので、引数を取るマクロは明示的に公開する）。したがって**ディレクティブ引数を取れるかどうかはファクトリ可視性で決まる**＝引数を一切取らない `@[Name]` は構築が `Name {}`（フィールド設定なし）なので公開ファクトリ不要だが、`@[Name(a: …)]` のように引数を取るには公開ファクトリが要る。引数フィールドが非公開なら生成時にエラー。

**なぜ `TokenStream` でなく `AST` 入力か（当初は TokenStream だった）。** 当初 `TokenStream` 入力を選んだ理由は (a) **安定境界**（トークンは AST より文法版に強い）と (b) **汎用性**（型宣言に限らず任意のトークン列を渡せる）。だが下記「共有 AST パッケージ」を最終形に据えると **(a) は消える**——AST も真実の源が 1 つで安定するので、トークンを緩衝材にする必要がない。しかも **derive は結局 AST へ parse して使う**ので、トークン段の安定性の恩恵を受けない（生トークンのまま処理するマクロだけが恩恵を受けるが、それは別 flavor）。残る (b) 汎用性は、Plew が **宣言への derive 専用**（関数形 `macro!(…)` / DSL マクロを持たない）なので出番が稀。Rust でも生 `TokenStream` 必須なのは `json!`/`html!`/`sql!`/`quote!` のような**関数形/DSL マクロ**で、**derive はほぼ常に `syn` で AST へ parse**する＝Plew の derive には AST で十分。よって入力は `AST`、生トークンは**将来の escape hatch**（構文ライブラリは内部に TokenStream/lexer/parser を持つので、関数形マクロを後で入れるなら `derive(input: TokenStream)` を別提供メソッドとして additive に足せる）。

**String → AST 変換と受け渡しは derive インターフェースの提供メソッドが担う（中間層は不要）。** `Derive`（assoc）／`ParameterizedDerive`（instance）はそれぞれ**要求メソッド `derive(input: AST) -> String`**（ユーザー実装）に加え、**提供メソッド `deriveFromSource(source, span) -> String`**（構文ライブラリ提供＝「String を lex+parse して AST にし、`derive(input:)` へ委譲」）を持つ。ランナー（生成コマンド）はこの提供メソッドを呼ぶハーネスを合成・実行するだけ＝**ランナーは String↔String の版非依存な機械**で、AST 型に一切触れない（AST 型・parser はマクロが固定依存する構文ライブラリ版のもの）。詳細は architecture doc。

**リッチな AST はコンパイラ外のライブラリに分離する（最終形）。** Rust が `rustc` と `syn` で**2 つのパーサ**を持ち同期に苦しむのを避けるため、Plew は self-host の利を活かし **lexer+parser+AST を 1 つのライブラリに切り出し、コンパイラもマクロもそれに依存**する構成を目指す（真実の源が 1 つ＝AST のバージョン違いのツラミを最小化）。ただし**現状パッケージ管理機構が無い**ため切り出しは後回しで、**当面は `@Plew/Syntax`（in-tree）が AST 型・lexer/parser・`Derive` 提供メソッドを持つ一時的な置き換え**（パッケージ管理導入後に外部共有パッケージへ昇格）。

### 出力モデル：別ファイル・原本不変・add-only（透明性の要）

1. **ビルドとは別コマンドで実行し、結果を別ファイルの Plew ソースとして出力**（Go の `go generate` 相当）。コンパイル時にインライン展開しない。生成物は普通の Plew ソースとして読め・追跡でき・コミットする（ビルドはマクロを走らせず生成済みファイルを読むだけ＝再現性）。

2. **生成先は `<Foo>.pw` に対し `<Foo>.gen.pw`**（ソースファイル単位）。

3. **原本（`<Foo>.pw`）は決して編集しない。** マクロは既存コードを**書き換えず追加するだけ（add-only）**。「上書き／削除するマクロ」は提供しない。

> この出力モデルが「**唱えた通りに発現・明示 > 暗黙**」を守る要。マクロが入力 AST を自由に読めても、**出力は読める別ファイルへの追記のみ**で原本は不変なので、「書いた通りが残る」透明性が壊れない（入力の自由は最大・出力は追跡可能な追記のみ）。

### 取り込み：`@[...]` の存在でローダが自動 part

生成ファイルを本体に綴じ込むのに、手書きの `part` も原本編集も**要らない**。

> `<Foo>.pw` に `@[...]` ディレクティブがあれば、ローダは **`<Foo>.gen.pw` を同じモジュールの `part` として自動で取り込む**。

- **原本不変**（ツールが編集しない）・**ボイラープレート無し**・**ダングリング無し**（derive を消せばディレクティブも消え取り込みも消える）。
- 取り込みは暗黙すぎない：可視の `@[...]` が「ここに生成コードが綴じ込まれる」マーカーになり、出どころ（このディレクティブ → 隣の `.gen.pw`）が明確。
- ビルド時に `<Foo>.gen.pw` が無い/古ければ**明示エラー**（「生成コマンドを走らせよ」）。hygiene・名前解決は「生成ファイルは普通のソース＝そのモジュールスコープで解決」で素直に成立。

### 組み込みの扱い：当面コンパイラ特権・将来 dogfood

組み込み（`Eq`/`Ord`/`Hash` …）も理想は同じ `Derive` 実装。だが bootstrap・性能・基盤未整備のため、**当面はコンパイラ特権の合成のまま**据え置き、API が固まってからユーザーマクロと同じ仕組みへ移行（dogfood）する。現行の `@[Eq]`/`@[Ord]` 合成はその特権実装。

## 確定した骨格（まとめ）

- **derive 可能性＝derive インターフェースの実装**（`X` はトレイトでも struct でもよい）。**設定なし→`Derive`（`assoc fn derive(input: AST) -> String`・bare `@[X]`）／設定あり→`ParameterizedDerive`（`fn derive`・self=設定 struct・`@[X(args)]`）**。`()` 規約で構文が機構を表す（bare=assoc／括弧=構築）。+**提供 `deriveFromSource`**（構文ライブラリ＝String→AST 変換＋委譲）。
- trait と derive の同名衝突（Plew は型/トレイト同一名前空間）は、**トレイト derive をトレイト自身の `assoc fn derive` にする**（`struct Hash` 不要）ことで解消。`All`/`Builder` 等トレイトを持たない derive のみ struct。
- 入力＝**AST**（span は原本座標・入力エラーはユーザー元位置）／出力＝**String**（生成ソース・出力エラーは `.gen.pw` 再コンパイル）。
- ディレクティブ引数＝`ParameterizedDerive` マクロ struct のフィールド（`@[Name(a:32)]`＝`Name{a:32}.derive(...)`）。
- 出力＝**別ファイル `<Foo>.gen.pw`・原本不変・add-only**、`@[...]` でローダ自動 part。
- ランナー（生成コマンド）＝**String↔String の版非依存な機械**（ハーネス合成→compile→run→stdout 回収）。AST 型・parser はマクロが固定依存する構文ライブラリ版のもの。
- 構文ライブラリ＝当面 `@Plew/Syntax`（in-tree）・最終形は外部共有パッケージ（コンパイラもマクロも依存）。
- 組み込み Eq/Ord/Hash は当面コンパイラ特権・将来 dogfood。

## 未決事項

- **AST 構文木の具体形の細部**（入力型は **`TopItemAst`** に確定済＝`@Plew/Syntax` の値ツリー。`Decl`〔`name`/`fields(name,type,vis)`/`variants`/fn シグネチャ＋本体〕・`Impl`〔`members`〕・`Trait`〔`reqs`〕…・全ノード原本座標 span。残るは各ノードのフィールド追加や型式 `TypeAst` の表現の磨き込み＝additive）。
- **`Derive` のトレイト名の最終確定**（**設定なし＝無印 `Derive`〔assoc〕・設定あり＝`ParameterizedDerive`〔instance〕** の方向は確定／正確な綴りは dogfood 着手時）。dogfood の前提として、**`Self` を含まないトレイト提供 `assoc fn` を、その trait 名で実体型なしに一意呼び出しする機構**が要る（`Hash.derive(input)`＝`impl Hash as Derive` が提供する唯一の deriver なのでディスパッチ不要・当面はメタ機構限定で実装可）。組み込み Eq/Ord/Hash が特権合成のうちは未着手で問題ない。
- 生成コマンドの名前・設定方法・source の渡し方（stdin/ファイル/リテラル＝実装時に最も楽な形で確定）。
- **authoring 層（将来・additive）**：`String` 直書きはシンタックスハイライトが効かない。テンプレートファイル方式（ほぼ Plew の雛形を読み込んで穴埋め）や opt-in の `quote` を**コアを汚さず後付け**で足す。コア（AST in / String out）は不変。
- **生トークン escape hatch**（関数形/DSL マクロを将来入れるなら `derive(input: TokenStream)` を additive に）。
- マクロ自身のビルドと配布（パッケージ管理導入後・同一リポジトリ内か別パッケージか）。
- 構文ライブラリの外部パッケージ切り出し（パッケージ管理導入後）。
- [拡張システム](../02-type-system/09-extensions.md)（`#Extension`）との関係。
