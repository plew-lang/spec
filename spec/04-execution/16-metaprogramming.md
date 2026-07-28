# メタプログラミング

> **状態: 方針確定・基盤実装済（M1＋共有パーサ統合済）。** 当初の「閉じたメタプログラミング」（言語提供の `@[...]` のみ）から、**ユーザー定義可能なメタプログラミング**へ転換し、その骨格（入出力モデル・出力先・取り込み方法・実行モデル）を確定した。本章はその確定方針の正典。**入力はかつて `TokenStream` だったが、AST 構文ライブラリを共有する設計（下記）に伴い `AST`（構文木＝`TopItemAst`）に確定**した。**実装済**＝`plewc --gen` モード＋共有 `@Plew/Syntax`（レクサ＋値ツリー AST＋宣言/本体/トップレベルパーサ `parseProgramAst`/`parseItem`）＋ローダ自動 part＋`plew gen` ランナーで、**マクロが注釈対象項の実構造（型名・フィールド名/型/可視性・enum バリアント/ペイロード・fn シグネチャ＋本体・impl のメンバ・trait の要求）を読んで生成し、`@[Name(label: expr)]` のディレクティブ引数も `self` から読める**まで貫通する（`@[Name]`→`<Foo>.gen.pw`→通常ビルドで取り込み）。**コンパイラ本体もこの共有パーサで構文解析する（真の 1 AST＝重複パーサなし）**＝マクロが見るのはコンパイラが見た構文そのもの。注釈対象は struct/enum/fn に加え **impl/trait** も可。`quote` 等 authoring 層・生成コマンド設定など**細部はなお未決**で末尾に残す。実装の段取り・実行系の具体は [agents/metaprogramming-architecture.md](../../../agents/metaprogramming-architecture.md)。

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

> 命名規則: ディレクティブ名は `macro Foo as Derive` / `macro Foo as ParameterizedDerive` が付いた**既存宣言**の名前で、組み込み・ユーザー定義を問わず **PascalCase** です（`Eq` / `Hash` / `Encode` / `Decode` / `All`）。`Derive` / `ParameterizedDerive` は通常 trait ではなく host-only な `macrointerface` です。一方、ディレクティブが**生成する**メンバ（例: `Enum.all()`）は通常の関数・変数なので camelCase です。

## ユーザー定義メタプログラミング

ユーザーが独自のコード生成を定義できます。骨格は以下の通り確定。

### マクロ = 既存宣言への macro role・入力は `AST`／出力は `String`

**derive 可能性 = 既存宣言に macro role が登録されていること。** `@[X]` が有効なのは、**名前解決された既存の `X` に `macro X as Derive` が見えるとき**だけ。`@[X(args)]` / `@[X()]` が有効なのは、同じく **`macro X as ParameterizedDerive` が見えるとき**だけ。`macro X` は `X` を新しく定義しない。`X` が既存宣言として解決できなければエラー。

`macro X as ...` は通常の `impl X as Trait` ではない。Plew の `impl B as A` は「`B` に準拠する型は `A` にも準拠する」という trait-to-trait conformance を意味するため、`impl Hash as Derive` と書くと「Hash を実装した型すべてが Derive でもある」という別の意味になってしまう。derive provider は runtime conformance ではなく host-exec/tooling surface なので、専用の `macro` 宣言で既存宣言へ role を付ける。

`macro X as ...` は `X` の所有モジュール内でだけ宣言できる（v1）。第三者が外部宣言へ勝手に同名の derive provider を後付けすることはできない。これは「`@[X]` の意味は、名前解決された `X` の所有者だけが決める」という provenance 規則であり、コアライブラリやコンパイラの依存グラフをどう切るかとは別軸の仕様判断である。

macro role の可視性は `impl` と同型に扱う。外部モジュールから `@[X]` / `@[X(args)]` を使えるのは、(1) 既存宣言 `X` 自体がその位置から見える、かつ (2) `X` の所有モジュールが `pub macro X as ...` を公開している場合だけ。`macro X as ...`（`pub` なし）は所有モジュール内だけで有効。`pub macro` であっても `X` 自体が `export` されていなければ、外部から `X` を名指せないので外部 derive としては使えない。

マクロは生成メソッド（要求＝`derive(input: TopItemAst) -> String`）を提供し、**注釈対象項の構文木を受け取り生成 Plew ソースを `String` で返す**。`Derive` 系契約は通常 trait ではなく `macrointerface` として宣言され、`AST` 型とともに**構文/マクロ用ライブラリ**（当面 `@Plew/Syntax`・将来は外部共有パッケージ＝後述）が提供します。**入力型 = `TopItemAst`**（注釈対象のトップレベル項の値ツリー＝`Decl`〔struct/enum/fn〕・`Impl`・`Trait`・… のタグ付き union）。マクロは `match` で対象の種別に分岐する＝対象が何かを**明示**して扱う（struct 専用 derive は `Decl` だけ扱い他を弾く）。これは「コンパイラとマクロが唯一の同一 AST を読む（1 AST 原則）」の帰結で、マクロ専用の縮小 AST を作らない。

**`macrointerface` は host-only なマクロ実装契約で、通常 trait ではない。** 見た目は trait に近い要求集合だが、runtime conformance ではなく、生成コマンドが既知の契約として読む compile/host 側 interface である。したがって `where T: Derive`、`any Derive`、`impl B as Derive`、extension、trait-to-trait conformance には使えない。`macro Foo as I` の `I` は `macrointerface` として解決され、通常 trait を指定するとエラー。v1 で認める `macrointerface` は compiler/generator が package identity 付きで知る閉集合だけで、既知パッケージ以外での新規 `macrointerface` 宣言はエラーにする。

```plew
export macrointerface Derive {
    assoc fn derive(input: TopItemAst) -> String

    assoc fn deriveFromSource(source~: String, start: U64, end: U64) -> String {
        return derive(input: parseItem(source: source, start: start, end: end))
    }
}

export macrointerface ParameterizedDerive {
    fn derive(input: TopItemAst) -> String

    fn deriveFromSource(source~: String, start: U64, end: U64) -> String {
        return self.derive(input: parseItem(source: source, start: start, end: end))
    }
}
```

**derive macrointerface は「設定（引数）の有無」で 2 つに分かれる。** derive の唯一の本質的な軸は設定を持つかどうかで、それが invocation 構文に直結する：

- **設定なし → `Derive`**（要求 `assoc fn derive(input: TopItemAst) -> String`・`self` 無し）。**bare `@[X]`** で呼ぶ。Eq/Hash/All 等、derive の大半。`Self` を含まない `assoc fn` なので、インスタンスを作らずに `X.derive(input)` を一意に呼べる。
- **設定あり → `ParameterizedDerive`**（要求 `fn derive(input: TopItemAst) -> String`・`self` = 設定構造体）。**`@[X(args)]`** で呼ぶ（`@[X()]` のように**括弧を強制**）。設定は構造体の型付きフィールドで、`self` から読む（`@[Builder(prefix: "with")]` 等）。

設定あり derive の `X` は通常の構造体であり、`@[X(args)]` は通常の factory 呼び出しと同じ可視性・型検査を受ける。外部パッケージから使わせる設定 schema は、`export struct X` と公開 factory として runtime surface にも現れる。これは host-exec 専用 schema を別言語・別名前空間に逃がさず、Plew の通常の型・可視性・factory 規則で説明するための意図的なコストである。

**`()` 規約が構文に機構を持たせる。** **bare `@[X]` は `Derive`（assoc）を、括弧つき `@[X(args)]`／`@[X()]` は `ParameterizedDerive`（インスタンス構築）を指す。** 引数を一切取らない構造体 derive も括弧を付けて `@[A()]` と書く（`()`＝「インスタンスを構築する」の合図）。bare `@[A]` と書きたければ `A` は `Derive`（assoc）の macro role を持つ。これにより、`@[X]` を見ただけでどちらの機構かが一意に決まる。

```plew
// 設定なし derive：既存の Hash トレイトに macro role を付ける
export trait Hash { /* ... */ }
pub macro Hash as Derive {
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
export struct Builder { pub val prefix: String }
pub impl Builder { factory }
pub macro Builder as ParameterizedDerive {
    fn derive(input: TopItemAst) -> String { /* self.prefix を使って生成 */ return "" }
}
@[Builder(prefix: "with")]       // = <Builder prefix="with" /> を構築して .derive(input)
struct Config { }
```

注釈は struct / enum / fn だけでなく **`impl` ブロック・`trait`** にも付けられる（`@[Name] impl T { … }`・`@[Name] trait U { … }`）＝そのとき `input` は `TopItemAst.Impl` / `TopItemAst.Trait`。**`extern` ブロック内の不透明 lang-item / FFI 型**（`extern(plewIntrinsic) { @[Name] struct I8 }`・`extern(c) { @[Name] type … }`）にも付けられ、`input` は本体なし `struct` の `TopItemAst.Decl`（`d.name` ＝その型名）。これでコア床のプリミティブ型に derive で実装を生やせる（例＝`@[IntTryFrom(sources: […])] struct I8` が整数 narrowing `TryFrom` witness 群を生成）。

コア床の型やトレイトに `macro X as ...` を付ける場合も、所有モジュール限定は崩さない。`Eq` / `Hash` の derive provider はそれぞれ `Eq` / `Hash` の所有モジュールで宣言されるべきで、依存循環が問題になるなら、解くべきは言語仕様ではなく**ライブラリ層分け**である。具体的には、`Derive` / `ParameterizedDerive` / `TopItemAst` などの host-exec 用 `macrointerface` と構文モデルを、コアライブラリが安全に参照できる薄い層（例：`@Plew/Macro` / `@Plew/SyntaxModel`）へ切るか、ホスト実行専用依存として runtime surface と分離する。`@Std/Core` がコンパイラ本体や重い構文実装へ依存する構成にはしない。生成物（`X.gen.pw`）はコミットされ、通常ビルドでは生成済み `impl` だけが読まれるので、出荷物の runtime 依存は構文ライブラリへ漏れない。

**なぜ `macro Foo as ...` か（trait と derive の同名衝突の解消）。** Plew は **型とトレイトが同一名前空間**なので、Rust 流に「`trait Hash` ＋ derive 用 `struct Hash`」を共存させられない（Rust はマクロ名前空間と型名前空間が別なので可能だが、Plew には無い）。一方で `impl Hash as Derive` は Plew 既存の trait-to-trait conformance と意味衝突する。そこで、Swift の macro 宣言に近い専用構文で、**既存の `Hash` という 1 エンティティに derive role を付ける**。`@[Hash]` は `Hash` トレイトそのものを名指すので、利用側の名前と生成される conformance の名前が一致する。`All`/`Builder` のような**対応トレイトを持たない** derive は、同名の marker/config struct を既存宣言として置き、そこへ macro role を付ける。**却下案**：(A) Rust 流の別名前空間＝Plew に無く、Core 内でモジュールを分けて名前空間で逃がすのは不自然。(C) trait と derive を別名（`@[DeriveEq]` 等）＝同名前空間なら筋は通るが、共通ケース（設定なし derive が大半）に恒久的な命名負担が乗り「`Eq` の derive は何という名前か」が毎回問われ混乱する。**採用**＝`macro Foo as Derive` により、名前は既存 `Foo` のまま、derive 用の `macrointerface` と host-exec 性だけを明示する。→ [agents/design-decisions.md](../../../agents/design-decisions.md)「trait と derive の同名」。

**入力 `AST`／出力 `String` の非対称は、エラーをどこに出すかと一致している。**

- **入力＝`AST`（span 付きノード）**：マクロが入力を解析して拒否する（「この型は Hash 不可」等）とき、**ユーザーの元ソース位置を指す**必要がある。AST ノードは span を保持し、その span は**原本座標**（`<Foo>.pw` の位置）なので、入力エラーはユーザー元位置を指せる。
- **出力＝`String`**：生成コードのエラーは**生成ファイル `<Foo>.gen.pw`（実ファイル）の普通のコンパイルエラー**で出るので、出力側に span は要らない（再コンパイルが位置を持つ）。`quote` のような埋め込み構文を作らず、生成は**普通の Plew 文字列・既存の補間 `"{…}"`** で組む（「Plew の中の Plew でないもの」を作らない・ハイライトもただの Plew）。

**ディレクティブ引数 = `ParameterizedDerive` の既存 struct のフィールド。** `@[Name(a: 32)]` は `<Name a=32 />` を構築して `.derive(input)` を呼ぶ糖衣（`ParameterizedDerive`・instance）。設定 struct のフィールドがそのまま設定スキーマで、`derive` は `self`（構築済み設定）からそれを読む。既存の labeled args / 構築構文を再利用し、新概念を作らない。**設定なし derive（`@[Eq]`/`@[Hash]` 等）は bare＝`Derive`（assoc）で構築を伴わない**（上の `()` 規約）。

**ディレクティブ引数は設定 struct の[ファクトリ可視性](../02-type-system/05-structs-enums.md#既定-factorymemberwiseとフィールドの既定値)に従う。** `@[Name(a: 32)]` は `Name` をハーネス（生成コマンド側＝マクロ定義モジュールの外）が構築するので、これは**外部からの `<Name a=32 />` 構築と同じ規則**に服す：設定 struct は**公開ファクトリ（`pub impl Name { factory }`）を露出し、引数に渡すフィールドが `pub`** でなければならない（既定 factory は非公開なので、引数を取るマクロは明示的に公開する）。したがって**ディレクティブ引数を取れるかどうかはファクトリ可視性で決まる**。引数フィールドが非公開なら生成時にエラー。

**なぜ `TokenStream` でなく `AST` 入力か（当初は TokenStream だった）。** 当初 `TokenStream` 入力を選んだ理由は (a) **安定境界**（トークンは AST より文法版に強い）と (b) **汎用性**（型宣言に限らず任意のトークン列を渡せる）。だが下記「共有 AST パッケージ」を最終形に据えると **(a) は消える**——AST も真実の源が 1 つで安定するので、トークンを緩衝材にする必要がない。しかも **derive は結局 AST へ parse して使う**ので、トークン段の安定性の恩恵を受けない（生トークンのまま処理するマクロだけが恩恵を受けるが、それは別 flavor）。残る (b) 汎用性は、Plew が **宣言への derive 専用**（関数形 `macro!(…)` / DSL マクロを持たない）なので出番が稀。Rust でも生 `TokenStream` 必須なのは `json!`/`html!`/`sql!`/`quote!` のような**関数形/DSL マクロ**で、**derive はほぼ常に `syn` で AST へ parse**する＝Plew の derive には AST で十分。よって入力は `AST`、生トークンは**将来の escape hatch**（構文ライブラリは内部に TokenStream/lexer/parser を持つので、関数形マクロを後で入れるなら `derive(input: TokenStream)` を別提供メソッドとして additive に足せる）。

**String → AST 変換と受け渡しは derive `macrointerface` の提供メソッドが担う（中間層は不要）。** `macro X as Derive`（assoc）／`macro X as ParameterizedDerive`（instance）はそれぞれ**要求メソッド `derive(input: AST) -> String`**（ユーザー実装）に加え、**提供メソッド `deriveFromSource(source, span) -> String`**（構文ライブラリ提供＝「String を lex+parse して AST にし、`derive(input:)` へ委譲」）を持つ。ランナー（生成コマンド）はこの提供メソッドを呼ぶハーネスを合成・実行するだけ＝**ランナーは String↔String の版非依存な機械**で、AST 型に一切触れない（AST 型・parser はマクロが固定依存する構文ライブラリ版のもの）。詳細は architecture doc。

**リッチな AST はコンパイラ外のライブラリに分離する（最終形）。** Rust が `rustc` と `syn` で**2 つのパーサ**を持ち同期に苦しむのを避けるため、Plew は self-host の利を活かし **lexer+parser+AST を 1 つのライブラリに切り出し、コンパイラもマクロもそれに依存**する構成を目指す（真実の源が 1 つ＝AST のバージョン違いのツラミを最小化）。ただし**現状パッケージ管理機構が無い**ため切り出しは後回しで、**当面は `@Plew/Syntax`（in-tree）が AST 型・lexer/parser・`Derive`/`ParameterizedDerive` macrointerface の提供メソッドを持つ一時的な置き換え**（パッケージ管理導入後に外部共有パッケージへ昇格）。

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
- 生成ファイルは**通常の `part` として parse/check される**。マクロ固有の「注釈対象への impl だけ」などの追加フィルタは持たない。生成結果が part として許されない項目や、所有権・coherence・可視性・公開 API 閉包性に反する項目を含むなら、通常のコンパイルエラーとして reject される。

### 組み込み derive の扱い：Eq/Hash は正規 derive・Ord は derive 不可

`Eq` / `Hash` はコンパイラ特権の合成ではなく、`Eq` / `Hash` の所有モジュールに置かれた `pub macro Eq as Derive` / `pub macro Hash as Derive` による正規 derive として扱う。`All` など対応トレイトを持たない derive も、同名の marker/config struct に macro role を付ける同じ仕組みで提供する。

`Ord` は derive を持たない。順序は型の意味を左右し、宣言順の辞書式順序を暗黙生成するとフィールドや variant の並べ替えが観測意味を静かに変えるため、順序が必要な型は明示的に `impl T as Ord` を書く。

## 確定した骨格（まとめ）

- **derive 可能性＝既存宣言に `macro X as Derive` / `macro X as ParameterizedDerive` が付いていること**。`Derive` / `ParameterizedDerive` は通常 trait ではなく generator-known な `macrointerface`。macro role は `X` の所有モジュール内でだけ宣言でき、外部から使えるには `X` 自体が可視で、かつ role 宣言が `pub macro` でなければならない。**設定なし→`Derive`（`assoc fn derive(input: AST) -> String`・bare `@[X]`）／設定あり→`ParameterizedDerive`（`fn derive`・self=設定 struct・`@[X(args)]`）**。`()` 規約で構文が機構を表す（bare=assoc／括弧=構築）。**提供 `deriveFromSource`**（構文ライブラリ＝String→AST 変換＋委譲）。
- trait と derive の同名衝突（Plew は型/トレイト同一名前空間）は、**既存の trait/struct に macro role を付ける**（`struct Hash` 不要・`impl Hash as Derive` 不使用）ことで解消。
- 入力＝**AST**（span は原本座標・入力エラーはユーザー元位置）／出力＝**String**（生成ソース・出力エラーは `.gen.pw` 再コンパイル）。
- ディレクティブ引数＝`ParameterizedDerive` 設定 struct のフィールド（`@[Name(a:32)]`＝`<Name a=32 />.derive(...)`）。
- 出力＝**別ファイル `<Foo>.gen.pw`・原本不変・add-only**、`@[...]` でローダ自動 part。
- ランナー（生成コマンド）＝**String↔String の版非依存な機械**（ハーネス合成→compile→run→stdout 回収）。AST 型・parser はマクロが固定依存する構文ライブラリ版のもの。
- 構文ライブラリ＝当面 `@Plew/Syntax`（in-tree）・最終形は外部共有パッケージ（コンパイラもマクロも依存）。
- Eq/Hash/All 等は正規 derive。Ord は derive 不可。

## 未決事項

- **AST 構文木の具体形の細部**（入力型は **`TopItemAst`** に確定済＝`@Plew/Syntax` の値ツリー。`Decl`〔`name`/`fields(name,type,vis)`/`variants`/fn シグネチャ＋本体〕・`Impl`〔`members`〕・`Trait`〔`reqs`〕…・全ノード原本座標 span。残るは各ノードのフィールド追加や型式 `TypeAst` の表現の磨き込み＝additive）。
- 生成コマンドの名前・設定方法・source の渡し方（stdin/ファイル/リテラル＝実装時に最も楽な形で確定）。
- **authoring 層（将来・additive）**：`String` 直書きはシンタックスハイライトが効かない。テンプレートファイル方式（ほぼ Plew の雛形を読み込んで穴埋め）や opt-in の `quote` を**コアを汚さず後付け**で足す。コア（AST in / String out）は不変。
- **生トークン escape hatch**（関数形/DSL マクロを将来入れるなら `derive(input: TokenStream)` を additive に）。
- マクロ自身のビルドと配布（パッケージ管理導入後・同一リポジトリ内か別パッケージか）。
- 構文ライブラリの外部パッケージ切り出し（パッケージ管理導入後）。
- [拡張システム](../02-type-system/09-extensions.md)（`#Extension`）との関係。
