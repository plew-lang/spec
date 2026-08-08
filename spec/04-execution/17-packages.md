# パッケージ

パッケージは **TOML マニフェスト `Plew.toml`** を持つディレクトリです。`src/` 以下にソースを置き、依存・ネイティブ依存を宣言します（**公開面はマニフェストでなくルート `_.pw` の `export`** で決まります）。中央レジストリは持たず、依存は **git リポジトリ／ローカルパス**で解決します（Go modules / SwiftPM と同じ分散思想）。モジュール・`import`・`export` の意味論は [モジュールシステム](15-modules.md)、本章はその上の**パッケージ単位の配布・依存・ビルド**を定めます。

## マニフェスト（`Plew.toml`）

最低限は **3 項目**だけです：

```toml
name = "Http"             # パッケージ名（PascalCase・"/" 可・@Http 参照のデフォルト束縛名）
plew = "0.3"              # 対象とする Plew バージョン（後方互換は当面保証しない）

[dependencies]            # 依存（下記）
```

- **version / license / repository / description は持ちません**。それぞれ唯一の真実源を参照します ── version は **git tag**、license は `LICENSE` ファイル、repository は git remote、description は `README`。マニフェストに重複させない（単一真実源・DRY）。
- `name` は**グローバル一意 ID ではなく、消費側が `@Http` で参照するデフォルトの束縛名**です。**`/` を含められます**（`Acme/Http`・`Std/Http`）が、これは**名前の字面**で、パッケージ内へ潜る経路ではありません（外部 API はフラット）── 関連パッケージを `Acme/…` で揃える組織的命名や std の分割に使います（**第一成分 `Std` は予約**）。パッケージの同一性の基礎は依存指定の git URL（衝突は消費側のローカル名で解決 → [モジュール章](15-modules.md#依存とローカル名)）。
- **`plew` は「対象とする Plew バージョン」**（要求最低版ではありません）。Plew は当面（0.x）**後方互換を保証しない**ため、`plew = "0.3"` は「このソースは Plew 0.3 でコンパイルされる前提」を意味し、**コンパイラがその版を受理できなければ loud fail**（「この依存は Plew 0.3 用・非互換」と早期に告げ、意味不明なコンパイルエラーの山を避ける）。ソース配布＋消費側ビルドゆえ全依存が現コンパイラで通る必要があり、`plew` はその**非互換の早期検出**を担います。将来「ここから後方互換を維持する」宣言（1.0 的なコミット・edition 的な互換機構）は additive です。
- **lib の公開面はマニフェストで宣言しません**（ルート `_.pw` の `export` だけ＝フラット・`import @Name`／`with`・パッケージ内へは潜れない → [モジュール章「公開面と外部到達」](15-modules.md#公開面と外部到達)）。**lib は 1 つ＋bin は複数**＝公開する実行ファイルは [`bin`](#bin公開する実行ファイル)（任意）で列挙します。
- **依存は 3 種**（いずれも任意）＝`[dependencies]`（コードから `import` するライブラリ依存。公開 API に現れる依存もここに置くが、public/private の区分は持たない・tree shake で含有判定）／`[tools]`（import せず run するだけの外部実行ファイル → [tools](#tools開発時に-run-する外部実行ファイル)）／ネイティブ依存 `[[native.<kind>]]`（下記）。**`dev-dependencies` は持ちません**（理由は下記「依存の含有＝tree shake」）。
- **`lockfile`（任意・既定 `"./Plew.lock"`）**でロックファイルの位置を指せます。複数パッケージで依存版を統一したいとき各 `Plew.toml` が同じ lock を指す（下記「ロックファイル」）。

## 依存（`dependencies`）

依存は **git リポジトリ URL の配列**です。最低限は URL のみ：

```toml
dependencies = [
    "https://github.com/foo/http.git",                              # これだけで可（ルートを name で束縛）
    { git = "https://github.com/bar/json.git", version = "3" },     # 制約する時だけ table
    { git = "https://github.com/x/db.git",    commit = "abc123ef" },# commit 固定
    { git = "https://github.com/x/edge.git",  branch = "main" },    # branch 追跡
    { path = "../local-lib" },                                      # ローカルパス
    { git = "https://github.com/y/z.git", members = [               # 単一パッケージの rename
        { path = "/", as = "MyZ" }                                  # ルート（/）を MyZ で束縛
    ] },
    { git = "https://github.com/x/mono.git", version = "3", members = [  # ワークスペース（複数メンバ）
        "/Http",                          # repo の /Http に依存（Http/Plew.toml の name で束縛）
        { path = "/Json", as = "MyJson" } # repo の /Json に依存（束縛名を上書き）
    ] },
]
```

- **束縛名は依存先マニフェストの `name` から自動**で決まります（`@Http` 等）。これにより**消費側は URL だけ書けば束縛名もバージョンも要りません**。**rename は member の `as` だけ**（下記）。
- 指定は `git` に対して `version` / `tag` / `commit` / `branch` の**いずれか 1 つ**、または `path` 単独です。複数併記はエラー（曖昧はエラー）。
- **`.git` は省略しません**（git リポジトリ URL を明示）。**短縮形は持ちません** ── レジストリ前提の裸バージョン（`"3" だけ`）も、ホスト短縮（`foo/http`）も無し。URL/path が常に可視で出どころが分かること（provenance）を優先します。
- **`members`（メンバ選択＋rename）**：`members` で**使うメンバを repo ルート起点の `/` パスで選びます**（`/Http`・`/` は repo ルートのパッケージ ── 自前ルート絶対 `import /Models/User` と同じ「ルートからの絶対」記法）。要素は「パス文字列 | `{ path, as }`」（トップレベルが「URL | テーブル」なのを 1 段下で踏襲）。**束縛名は各メンバの `name` から自動・rename は `as`（rename はここだけ）**。version/tag/commit は外側 git に**1 回だけ**書け、メンバ側には書けません ── **1 ワークスペース 1 バージョン**を構文で担保（→ [ワークスペース](#ワークスペース複数パッケージのリポジトリ)）。列挙したメンバだけが import 可（phantom 禁止）。**単一パッケージは `members` 省略（＝ルート `/` を name で束縛）の退化形**で、rename したいときだけ `members = [{ path = "/", as = … }]` と書きます。

### バージョン指定（桁数モデル）

git tag を semver として解釈します。**書いた桁が固定・書かない桁は最新**という桁数モデルです（`^`/`~`/`=` 演算子は使いません）：

| 指定 | 解決される範囲 | 意味 |
|---|---|---|
| なし | 制約なし | メジャーも自動更新（最新追従） |
| `"3"` | `>=3.0.0, <4.0.0` | メジャー固定・マイナー以下を更新 |
| `"3.2"` | `>=3.2.0, <3.3.0` | マイナー固定・パッチを更新 |
| `"3.2.1"` | `=3.2.1` | 完全固定 |

- tag に **`v` は付けません**（`1.2.0`・semver 仕様に忠実。git tag 慣習の `v1.2.0` 対応は当面なし・将来 additive 可）。
- **0.x を特別扱いしません**：`"0"` は 0.x 全体追従（破壊的変更の取り込みは自己責任）、`"0.3"` はマイナー固定（`>=0.3.0, <0.4.0`）。
- **下限と範囲を独立に指定する形（`>=3.2.5, <3.3.0` 等）は持ちません**（桁数モデルのみ・必要になれば将来 additive）。

## 依存解決

- **制約内の最新を選びます**（newest-compatible）。最新を取りに行くので、再現性は下記のロックファイルで固定します。
- **複数バージョン共存**：同じメジャー範囲の複数要求は**最新 1 つに統合**します。**メジャー違いの要求は両方を共存ビルド**します（A が `B v1`・C が `B v2` を要求しても両立＝依存地獄の回避）。ただし：
  - **`@Std`・言語アイテム・コアトレイトは単一版必須**です（`Array`/`Eq` 等が複数版あると型システムが壊れる）。
  - **異なるバージョンの同名型を API 境界で晒すと別型として扱われ、コンパイルエラー**になります（hidden にしない）。共存が安全なのは、型・トレイトを境界で晒さないライブラリです。
- **間接依存は import できません（phantom dependency の禁止）**：パッケージ内のコードが `import @Name` できるのは、**自身の `dependencies` に書いた直接依存と `@Std` のみ**です。推移依存はビルドには使われますが import できません（依存先が `export @Dep with { ... }` で明示的に再公開した名前を、その依存先の公開面として使う場合を除く）。公開 API のシグネチャに推移依存由来の型が現れるだけでは、その型を現在パッケージの公開名として再公開したことにはならず、消費側に `@Dep` の import 権も与えません。消費側がその型を自分のソースで名指したい場合は、`@Dep` を直接依存として宣言します。`dependencies` に無い `@Name` の import はコンパイルエラーです。

### 依存の含有＝tree shake（`dev-dependencies` を持たない）

**`dev-dependencies` という区分は持ちません。** ライブラリ依存はすべて `[dependencies]` に置き、**何が成果物に入るかは whole-program tree shake（到達可能性）が決めます**。テストからしか到達しない依存は prod ビルドから自動で落ち、テスト依存を「dev」とラベルする必要はありません。

- **「dev か runtime か」は宣言でなく到達可能性から創発**します（正直）。`dev-dependencies` は「import できるが使ってよい文脈はテスト関数の中だけ」という**暗黙の文脈ルール**を生むので採りません ── どこで使ってもよく、到達した所にだけ含まれる。
- **`derive` は `[dependencies]`**：`@[Derive(...)]` で**シンボルを import して参照する**ので依存。「`plew gen` で run される」は宣言区分と直交し、生成 `.gen.pw` が提供元 runtime を参照すれば成果物に入り、自己完結なら tree shake で落ちる。区分は **import するか（→ `[dependencies]`）／import せず run するだけか（→ [`[tools]`](#tools開発時に-run-する外部実行ファイル)）** で割れます。
- **消費側の解決は到達駆動**：consumer は自分の entry から辿った import だけを解決します。あなたのテストからしか参照されない依存は consumer の到達範囲外ゆえ解決されません（Cargo が manifest 駆動ゆえ `dev-dependencies` で枝刈りした問題を、whole-program from source の到達駆動で構造的に消す）。

## ロックファイル（`Plew.lock`）

解決結果を固定して再現ビルドを保証します。形式は **TOML**（`Plew.toml` と同形・Cargo/uv と同じ現代的選択・git merge しやすい）で、**解決した各パッケージを `[[package]]` で 1 ブロック**ずつ並べます：

```toml
version = 1                           # スキーマ版（将来の形式変更用）

[[package]]
git = "https://github.com/foo/http.git"
version = "3.2.1"                     # 解決した tag
commit = "abc123ef…"                 # 確定 commit SHA（fetch ターゲット兼整合性）
dependencies = ["Json", "Url"]        # 解決済みの直接依存（辺）

[[package]]                           # 多版共存は同名で複数エントリ
git = "https://github.com/bar/json.git"
version = "2.0.0"
commit = "…"

[[referrer]]                          # 共有 lock のとき：この lock を指す Plew.toml（自己修復用・下記ワークスペース）
path = "../app/Plew.toml"
```

- **整合性は commit SHA で足り、別の content hash は持ちません**。git の commit は木に暗号学的にコミットしている（fetch した commit を git が検証）ので、Cargo の git 依存と同じく **commit だけが整合性の保証**。path 依存は自分が握るローカルゆえ整合性ハッシュ不要。
- **`commit` は fetch ターゲット兼同一性**。`version`（tag）は人間可読の参照・`commit` が真の固定。
- **依存の辺（`dependencies`）を記録**します（provenance＝「なぜこの版が居るか」が追え、再解決の検証に使える）。
- **決定的ソート**（git URL → version）で merge 競合を最小化（Cargo が[同じ理由](https://github.com/rust-lang/cargo/pull/7070)で 1 パッケージ 1 ブロック・ソート形に）。
- **ライブラリ・アプリともコミット推奨**です。ただし **lock が効くのはトップレベル（ビルド対象）のみ** ── 依存ライブラリ同梱の lock は**無視**され、利用側は依存の**制約（マニフェスト）だけ**を見て自分のグラフを再解決し、自分の lock を書きます。
- バージョンの更新は **`update` コマンドで明示的に**行います（ビルドが暗黙に上げることはしない）。
- ネイティブの **Rust 依存（cargo ビルド＝ビルド時にホストでコード実行）は lock に記録し初回承認**を求めます（→ [ネイティブ依存](#ネイティブ依存c--c--rust--system-ライブラリ)）。

## ワークスペース（複数パッケージのリポジトリ）

ワークスペースは**複数のメンバパッケージ（各サブディレクトリの `Plew.toml`）を含む単一 git リポジトリ**です。**repo 側に `members` を列挙する root manifest（Cargo 流の手書き第二真実源）も「ワークスペース」専用の別概念も持ちません** ── メンバはサブディレクトリに `Plew.toml` で在るだけで、消費側が依存の `members` で必要なものを選びます（→ [依存](#依存dependencies)・消費側の選択は repo 側の真実源とは別物）。

- **1 ワークスペース 1 バージョン**：version は git tag で、タグはリポジトリ全体に 1 つ打たれます。ゆえにメンバを跨いだ独立バージョンは**構造的に存在しえません**（lockstep＝1 つを上げれば全メンバが同じタグで上がる）。「リポジトリが複数バージョンを持つ」状態を最初から表現不可能にしています（不正を構造で排除）。消費側も version を外側 git に 1 回しか書けないので、メンバ間で食い違うバージョン指定も書けません。
- **メンバ選択は `/` パス・束縛は `name`**：`members` は**repo ルート起点の `/` パス**でメンバを指し（`/Http`・`/` は repo ルートのパッケージ＝自前ルート絶対 `import /Models/User` と同じ記法）、束縛名は各メンバの `name`（`as` で上書き）。トップレベルが「git URL（場所）を指して `name` で束縛」なのと同型で、メンバは「`/` パス（場所）を指して `name` で束縛」。**rename（`as`）はこの member 内だけ**＝単一パッケージの改名も `members = [{ path = "/", as = … }]`（`as` の置き場が 1 つに収束・単一は「ルート member 1 つ」の退化形）。解決はその 1 リポジトリ内に限定（暗黙のスコープ公開ではない）で、列挙したメンバだけが import 可。

### 共有ロックで版を統一

複数パッケージを一緒に開発し**依存版を統一**したいときは、各 `Plew.toml` の `lockfile` を**同じ lock に向ける**だけです：

```toml
# app/Plew.toml ・ lib-a/Plew.toml ・ lib-b/Plew.toml がそれぞれ
lockfile = "../Plew.lock"     # 既定は "./Plew.lock"（自分の隣）
```

- **「1 個」も「n 個」も `lockfile` パスの違いだけ**＝仕組みが 1 つ。複数のビルドルート（各メンバを単独ビルド/テスト）が同じ lock を共有するので、**「単独テストで通った版＝app に組んでも同じ版」**が保たれます。
- **lock は完全な導出物**：共有 lock を指すパッケージ群を統一解決し、lock は**使われた参照元（各メンバの `Plew.toml` パス）を記録**します。解決時にその参照元を辿り、**今も this lock を指すものだけ**をメンバとして再解決し、指さなくなった/消えたものは記録から落とします（**真実源は各 `Plew.toml`**・lock は自己修復するキャッシュ）。
- **統一は累積的**：lock が知るメンバは「一度でも解決されて記録されたもの」です。新メンバは**初回ビルドで記録されるまで**統一に入りません（その後は収束）。「手動列挙の事前性」より「自動記録の自己修復」を取る設計です。

## bin（公開する実行ファイル）

パッケージは **1 つの lib（`_.pw` の export 面）＋ 複数の bin**（実行可能エントリ）を持てます（Rust の lib+bin モデル）。公開する bin は `bin` で列挙します ── **lib 面は `_.pw` で表す（列挙しない）が、bin は複数あり消費者が発見する必要があるので列挙する**（発見性のための明示・lib の単一根とは事情が違う）。

```toml
bin = [
    "/Cli",                                     # bin "Cli"（src/Cli.pw・main 必須）
    "/Server",                                  # bin "Server"
    { path = "/Tools/Migrate", as = "migrate" } # rename した bin "migrate"
]
```

- 要素は「**`/` ルート起点のエントリモジュールパス** | `{ path, as }`」（`dependencies`/`members` と同じ string|table・`[[bin]]` ブロック記法でも書ける）。**名前 = パス末尾**（`Cli`・`Server`）が既定、`as` で rename（path=file の正直さは既定で保つ）。
- 列挙したファイルは **`main` 必須**。`main` を持つが `bin` 未列挙のファイルは公開 bin ではありません（**偶発的な `main` が公開 bin になる穴を塞ぐ**）。`_.pw` が `main` を持てば**パッケージ名の既定 bin**（`plew run @Pkg`・無セレクタ）。
- **呼び出しは `plew run @Pkg:Name`**（bin セレクタ `:`・`/` はパッケージ名の字面ゆえ不可）。**全 bin が install 対象**で特権はなく（`plew install @Pkg` で全 bin・`--bin` で選択＝`cargo install` 流）、自分／ワークスペースは `plew run :Name` → [モジュール章「ビルド・実行」](15-modules.md#ビルド実行)。

## tools（開発時に run する外部実行ファイル）

`[tools]` は **import せず run するだけの外部実行ファイル**（migration CLI・linter・自前でない codegen 等）。`[dependencies]`（import するライブラリ）とは「**import するか／run するだけか**」で分かれます。

```toml
[tools]
"https://github.com/x/migrate.git" = "3"      # plew run @migrate で実行
```

- 表記は `[dependencies]` と同じ（git URL・members・`commit` 等）。**この repo の開発でだけ使い・consumer に伝播しない・runtime グラフ外**。
- **プロジェクトローカルに宣言**するが、**ビルド実体は content-addressed キャッシュで全プロジェクト共有**＝Node/Go 流のローカルツールを、Rust の「ローカル化＝再コンパイル重複」コスト無しで持てる（[配布とビルド](#配布とビルドソース配布消費側ビルドキャッシュ)）。
- グローバルに入れたい公開ツールは `[tools]` でなく `plew install`（`cargo install` 流）。

## 配布とビルド（ソース配布＋消費側ビルド＋キャッシュ）

- 配布物は **Plew ソース**です（precompiled lib のリンクは一次手段にしない）。
- ビルドは**消費側の `plewc` が fetch 済みソースから whole-program** で行います。
- 速度は**ビルドキャッシュ**で回収します（hidden cost は可・hidden meaning は不可）。キーは **`compiler version × target × flags`** に、**不変な fetched 依存は lock の commit SHA・編集中の局所コード/`path` 依存は content/mtime** を掛けたハイブリッド（commit がある依存は content hash を別途計算しない＝[ロックファイル](#ロックファイルplewlock)と同じ姿勢）。
- **置き場は Cargo 流**＝**fetch ソースはグローバル一元 cache**（commit キーで content-addressed・read-only・同 commit を二度落とさない）、**コンパイル成果物は per-project**（whole-program 粒度。クロスプロジェクト共有はシンボルの content 安定化を要する将来課題）。`[tools]` の実体だけはグローバル content-addressed 共有（[tools](#tools開発時に-run-する外部実行ファイル)）。
- 実行可能ビルドのエントリ選択は → [モジュール章「ビルド・実行」](15-modules.md#ビルド実行)。

## ネイティブ依存（C / C++ / Rust / system ライブラリ）

ネイティブ依存は **`Plew.toml` の `[[native.<kind>]]` 宣言**で表します。Plew 独自の **`Build.pw` / `build.zig` 的な任意 build hook は持ちません**。Plew 本体が宣言を読み、祝福した既知ツール（clang/cargo/pkg-config）を決まった作法で実行します。ただし `[[native.rust]]` は Cargo エコシステムの `build.rs` を推移的に含み得るため、cargo build は **host-exec** として lock / trust / sandbox / artifact provenance の対象になります。

```toml
[[native.c]]
sources = ["native/sqlite3.c"]
include = ["native/"]
defines = { SQLITE_THREADSAFE = "0" }
std = "c11"

[[native.cpp]]
sources = ["src/engine.cpp"]
std = "c++17"

[[native.rust]]
path = "rust/mycrypto"                    # crate ディレクトリ（staticlib + extern "C"）・これだけ

[[native.system]]
pkg-config = "llvm"                       # system ライブラリ（identity は値 "llvm"）
link = "static"                           # 外部 lib を静/動どちらで繋ぐか（system のみ）
```

- **kind はテーブルヘッダ**＝`[[native.c]]` / `[[native.cpp]]` / `[[native.rust]]` / `[[native.system]]`。**ヘッダ自体が判別子**なので各ブロックは**固定スキーマ 1 つ**＝判別子フィールド不要・スキーマ補完が確実に効き、**kind 外のフィールドはそもそも書けない**（illegal state を構造で排除）。
- **`link`（`"static" | "dynamic"`）は `[[native.system]]` 専用**＝外部システムライブラリを静的/動的どちらで繋ぐかの選択。バンドルした C/C++・Rust crate は「自分のプログラムに組み込む＝常に static」で選択肢がなく `link` を持ちません（crate の静/動は `Cargo.toml` の `crate-type` が決め、Plew は出た成果物を読んでリンク）。
- **ターゲット条件付け（per-target の `unsupported`/差し替え）は現状持ちません**。「native の一部がターゲット非対応のときプログラムがどうフォールバックするか」は**パッケージの target サポート行列＝トップレベルの関心**で、per-dep のブール値では解けないため、入れるなら正しい階層で（additive）。Rust は `Cargo.toml` の `[target.'cfg()']`/`#[cfg]` で crate 内に条件分岐を持てます。
- **C/C++・Rust・system(pkg-config) を祝福**（名指しサポート）。これ以外（cmake の巨大プロジェクト・Go 等）は v1 では **Rust ラッパクレート**か system library 経由で取り込みます。直接のプリビルドバンドル宣言は未対応です（下記「巨大ライブラリ」）。
- **Plew 独自の任意コマンド実行を持たない**ことで、Plew が直接起動する入口は clang/cargo/pkg-config に限定され、サンドボックスのポリシーが書けます。Cargo が起動する `build.rs` や子プロセスは Rust host-exec closure として別途 provenance に含めます（任意コマンドのサンドボックスは原理的に困難＝SwiftPM も自動ビルドから締め出している）。

### C / C++（`[[native.c]]` / `[[native.cpp]]`）

C と C++（コンパイラ駆動と stdlib リンクだけが違う）。フィールドは**ビルドが意味を所有・検証できるものだけ**を構造化して受けます（生のコンパイラフラグは渡せません ── 下記）：

| フィールド | → | 内容 |
|---|---|---|
| `sources` | `.c`/`.cpp` を各々 `-c` でコンパイル | 翻訳単位（`.h` は**入れない**） |
| `include` | `-I` | ヘッダ検索ディレクトリ（パッケージ内に限定検証・任意） |
| `defines` | `-D` | key=value（値は文字列か `true`） |
| `std` | `-std=` | 既知の列挙値（`c11`・`c++17` 等） |

- **複数ソースは各々 object へコンパイルしまとめてリンク**（C の標準モデル＝翻訳単位ごとに `.o`）。`include` はそのブロックの全ソースに共通で `-I`。
- **`.h` は `sources` に列挙しません**（ヘッダは翻訳単位でない・`.c` が `include` の `-I` 経由で見つける）。**ヘッダ変更時の再ビルドは depfile（`clang -MMD`）で実際に include されたヘッダを自動採取しキャッシュキーに畳む**（ユーザーは宣言しない・機構が拾う＝hidden cost 可・meaning は透明）。
- **生のコンパイラフラグ（`flags = [...]`）は持ちません**＝`-fplugin`/`-B`/`-Xclang -load`/`-wrapper`/`@response-file` 等が**ビルド時の任意コード実行（ACE）**になり、「起動するのは clang だけ→サンドボックス可」の前提を崩すため（マニフェストがそもそもコード実行を表現できないことが守りの一段目）。**ツールパス上書き（`compiler`/`archiver`/`ranlib`）も同理由で不可**。
- **codegen/target は Plew が所有**＝`-O`（最適化）・`-g`（debug）・`-fPIC`・`-target`（クロスコンパイル）・MSVC ランタイム等は消費側のビルドモード/ターゲットから決定（依存だけ別設定は whole-program を壊す）。**第三者 C の警告は既定オフ**（あなたが書いた C ではない）。
- C++ の stdlib（libc++ vs libstdc++）は**ターゲット既定を Plew が持ち、上書きは将来 additive**。

### Rust（`[[native.rust]]`）

`path` で**パッケージ内の crate ディレクトリ**（`Cargo.toml` のある場所）を指し、Plew がそこで cargo を駆動して成果物をリンクします。

- **crate は `crate-type = ["staticlib"]`（or `cdylib`）で `extern "C"` を晒す**＝Plew は `extern(c)` でそのシンボルを束縛します（C と同じ・cargo はリンクするシンボルを増やすだけ）。**Rust ABI は直接呼べない**ため、crates.io crate を使う場合も**間に C ABI を晒す薄い shim crate が不可避**で、`path` はその shim を指します。
- **crates.io 依存・feature は crate の `Cargo.toml`/`Cargo.lock` が所有**（単一真実源 ── version を tag に寄せるのと同じ DRY で、`Plew.toml` に `features` を二重化しません）。
- **Plew が駆動・所有**：runtime native artifact は消費側 build target の `--target`（クロスコンパイル）と profile（release/debug）で作ります。一方、derive provider / `[tools]` / build-time helper として実行される Rust crate は**開発者ホスト triple**で build し、target artifact とは別 cache / fingerprint / trust entry にします。同じ package が runtime 依存かつ host-exec 依存であれば、host 用と target 用の 2 instantiation が存在します。**fetch（`cargo fetch --locked`・ネット・content hash 検証）と build（`cargo build --locked --offline`・隔離）を分離**し、成果物 `.a` の場所は cargo の `--message-format=json` 出力から取得します（機構＝hidden cost）。
- crate 側 `build.rs`（推移 crates.io 依存を含む）の信頼は Rust エコシステムに委ねます。Plew は cargo プロセスとその子プロセスを OS レベルで sandbox し、ネットワーク・書き込み先・入力 root を policy に閉じ、wall clock / random / undeclared env / host CPU probing など deterministic profile 外の観測を閉じるところまで担います。消費側に **Rust ツールチェーンが要り**、**`Plew.lock` に「cargo ビルド＝ビルド時にホストでコード実行」という解決事実と runtime native artifact provenance**を記録し、実行許可は user-local trust DB で初回承認を求めます。`build.rs` と cargo が起動する非 cargo/rustc executable（`cc`、`pkg-config`、`protoc`、python 等）とその入力は、runtime native build でも host-exec closure の一部です。実行を許すには manifest / resolved closure / sandbox policy で列挙され、tool identity/hash、入力 root、system library / dynamic loader closure、出力 artifact hash が lock/build provenance と trust key に入らなければなりません。closure 外 executable の起動、PATH 探索、undeclared filesystem/network input は runtime native build でも reject します。

### system ライブラリ（`[[native.system]]`）

- **`pkg-config = "name"`**（system ライブラリ）。`link = "static" | "dynamic"` は任意。system lib は通常 runtime link でも host-exec 入力でも、解決された pkg-config package name/version、cflags/libs、library path、linked artifact fingerprint、runtime dynamic library identity（dynamic link 時）、pkg-config binary/toolchain fingerprint を lock / build cache key / resolved closure に含めます。
- dynamic link では、直接 dylib だけでなく推移的 loader closure（`DT_NEEDED` / `LC_LOAD_DYLIB`、`RPATH`/`RUNPATH`、loader search path、dyld/ld.so 関連 env の固定値、各 dylib identity/hash）も provenance に含めます。host-exec に使う dynamic lib は exact loader closure を pin できなければ reject し、runtime link でも loader closure が変われば別 provenance / cache key です。pkg-config が返す flags は構造化して安全な subset に parse し、`-Xclang -load`、`-fplugin`、`-B`、`-wrapper`、linker plugin、`@response-file`、tool path override、任意コマンド起動に繋がる flag は reject します。
- 複数の system lib は `[[native.system]]` を並べます（単一値ゆえやや冗長ですが、全 kind を `[[native.<kind>]]` で揃える一貫性・補完の確実性を優先）。

### 巨大ライブラリ（ONNX 等）

declarative な C/C++ ブロックは **vendored な小〜中規模 C**（amalgamation 一枚・単機能 lib）が対象です。CMake 製の巨大 C++ をソースからビルドするのは非現実的（数千ファイル・プラットフォーム別の define 分岐・生成コード・自前の third-party 依存＝ビルドロジックの手再現＝ビルドツール地獄）。v1 の選択肢は、① **既存の Rust ラッパクレート**（ONNX なら `ort`・cargo の `build.rs` が prebuilt 取得/リンクを解決済＝`[[native.rust]]` で自分の shim crate が `ort` に依存し `extern "C"` を晒す・Rust アプリと同じ手間）② **system `pkg-config`**（環境に入っていれば）です。**プリビルドバンドルを直接宣言する `[[native.prebuilt]]` は v1 では持たず reject** します。対応するなら target/ABI matrix、static/dynamic、artifact hash/signature、link path、runtime dylib provenance を lock に入れる別 schema が必要ですが、それは将来の追加です。Rust ラッパも system lib も無い「ソース自力ビルドのみ」の巨大 lib は Plew 単体では詰む── ビルドシステムを内蔵しない（CMake/Bazel 相当を抱えない）ことの帰結。

言語側の `extern(c)` 構文（型マッピング・`CPtr`・`repr(c)`）は → [モジュール章「外部コード統合」](15-modules.md#外部コード統合externc-ffi)。本章はその**リンク／ビルドの宣言**を担います。

## 越境メタプログラミング

依存が公開する `Derive` を消費側の型に当てる場合も `.gen.pw` コミットモデル（生成は当てる型を持つ**消費側**で起き `.gen.pw` を消費側にコミット・`@Plew/Syntax` は各 derive の依存版で隔離実行・derive はホスト実行）を使います。複数バージョン共存と String 境界で両立します。→ [メタプログラミング](16-metaprogramming.md)。
