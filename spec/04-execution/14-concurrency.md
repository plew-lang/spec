# 非同期処理とメモリ管理

Plew は JavaScript と同様の**シングルプロセス・シングルスレッド + イベントループ**を基盤とします。重い処理を別スレッドで動かしたいときだけ `spawn` でスレッドを立てます。メモリは **ARC（自動参照カウント）** で管理し、破棄は決定的（[`deinit`](../01-basics/03-values.md#deinit)）、循環は [`WeakRef`](../01-basics/03-values.md#ref--weakref共有可変) で断ち切ります。参照カウントの更新は通常は非 atomic で、複数スレッドから共有され得る allocation だけをコンパイラが atomic にします。

> **用語**：本書は、コピー可否を **`unique`／コピー可能**、スレッド間の移送可否を **`sendable`／`nonsendable`** と表します。`sendable`／`nonsendable` はキーワードであり、型・クロージャ・generic 境界に現れます（→ [値・変数・所有権](../01-basics/03-values.md#sendable--nonsendableスレッド間の移送可能性)）。

## 基本的な非同期処理（async / await）

```plew
async fn fetchData(url: String) -> Promise[Result[Data, Error]] {
    val response = await httpRequest(url: url)
    val data = await response.json()
    return <Result.Ok value=data />   // async fn の戻り値は Promise に自動ラップ
}

async fn main() {
    val data = await fetchData(url: "https://api.example.com/data")
    print("Received: {data}")
}
```

- **戻り型は `Promise[T]` と明示して書き**、本体の `return e`（`e: T`）はコンパイラが `Promise[T]` に包みます（TypeScript と同じ）。`await` はその `Promise[T]` を `T` に開きます。
- 戻り型を省略した `async fn`（`main` など）は、値を返さない `Promise` を返します。
- **`async` は単一スレッド上の協調的中断**で、別スレッドは起きません。所有権・借用は同期コードと同じに効きます（`unique` 値を await を跨いで保持してもよい）。

### async メソッドと self

`async fn` のメソッドは **self を借用できません**（借用は async 境界を越えられない＝`inout fn` の async 版は不可）。よって：

- **非消費の async メソッドは self をコピー**する（Self がコピー可能なときのみ）。＝Self がコピー可能必須で、generic struct なら型引数もコピー可能（[`allowUnique` でない](../02-type-system/06-generics.md)）。
- **消費する async メソッドは `async move fn`**（self を move）。
- **await を跨いで self を可変共有したいオブジェクトは `Ref` 裏打ち**にする ── `ref->asyncMutate()` は `Ref` 越しに self を触り、`Ref` は async 境界を越えられる（単一スレッドでメモリ安全・interleave は JS 同様の論理ハザード）。ステートフルな async オブジェクトは Ref 裏打ち＝JS のオブジェクトと同じ姿。

### unique 結果と `allowUnique`（v1 は不可・将来）

v1 では **`Promise[T]`/`JoinHandle[T]` を含むコアの generic 型はすべてコピー可能な型に限定**（`allowUnique` 未導入）。これは `unique` の制約であって、sendability の制約とは別です。`Promise[T]` は単一スレッド内で使うため nonsendable な `T` も許容しますが、別スレッドから結果を受け取る `JoinHandle` は宣言を **`struct JoinHandle[sendable T]`** として `T` の sendability も要求します。帰結：

- async/spawn は **unique 結果を返せない**（`-> Promise[unique]` / `-> JoinHandle[unique]` 不可）。async で unique を持ち回るなら **`Ref` 包み**（`Ref` は async を越えられる）。
- **unique を generic に入れるのは常に `Ref` 包み**（`Optional[Ref[File]]`・`Array[Ref[File]]`）。`Ref` はコピー可能なので通常のコピー可能なコレクションになり、`match`/peek/反復が普通に効く（move-out 専用 API も `take()` も不要）。
- by-value の unique を generic で扱う（`Optional[unique]`・`Promise[unique]`・`Array[unique]`）には **借用束縛＝ライフタイム**が要り（要素を取り出さず借用で覗く操作のため）、v1 の非 escape 借用では実装できない。**`allowUnique` は将来の additive**（保持系＝Optional/Promise が先・Array/Iterable は escaping borrow 導入後）。それまでは `Ref` 包みで代替。

## メモリ管理（ARC）

- **ARC（自動参照カウント）で管理**：スコープを抜けて最後の所有者／`Ref` が消えると**即座に**解放され、`deinit` が走る（決定的破棄＝ファイル・ソケット等の資源解放に使える）。ただし**`panic` 時は abort で巻き戻さないので `deinit` は走らない**（資源は OS が回収・[制御構造 § panic と発散](../03-expressions/11-control-flow.md#panic-と発散) 参照）。決定的破棄が保証されるのは正常終了パスのみ。
- **循環は `WeakRef` で**：参照カウントは循環を回収しないので、親子の逆リンク等は `WeakRef` で断ち切る。**循環が生じ得るのは `Ref` グラフだけ**（値世界＝`Array`/`String`/`Dictionary`・[自動箱化された再帰値型](../02-type-system/05-structs-enums.md#再帰的な値型)は構造上つねに木／DAG）なので、純粋 ARC は値世界を取りこぼさない。
  > **将来 additive：循環の自動回収。** `Ref` グラフは小さく隔離され（trace 対象は `Ref` ボックス＋`mut val` 参照キャプチャしたクロージャだけ）、しかも **`Ref` は非 atomic かつ nonsendable**（spawn を越えない＝per-thread）・**単一イベントループ**（ターン間が天然のセーフポイント）・フルマネージドなので、ARC の上に **`Ref` グラフ限定のサイクルコレクタ**（Bacon–Rajan の trial deletion・CPython/PHP で実証）を **per-thread・idle 実行**で非破壊に足せる。**ゴミ循環を検出したときの挙動**は、(1) retain path 付きで **loud に報告**（dedup・診断フック経由・**全ビルド共通＝release 含む**＝`assert`/overflow panic と同じく「意味論を変えるビルドを持たない」原則の一貫）、(2) **メモリを回収**、(3) **循環メンバの `deinit` は走らせない**。`deinit` の決定的契約は正常路（スコープ離脱・`return`・`try`/`Result` 伝播）のみで保証し、`panic`（abort）と「リークした循環」はともに契約外の脱出で deinit を飛ばす（[panic と発散](../03-expressions/11-control-flow.md#panic-と発散) と対称）。**deinit の有無で挙動を分けない**ので「構造体に `deinit` を足したら循環がリークし始める」崖は生じず、循環メンバの deinit が予期しない時刻に走る驚きも生じない（走らないだけ）。重大度は値の嘘ではない（資源の無駄＋deinit skip）ので panic でなくログ。手動 `WeakRef` は「正しさのため必須」から「決定性・性能の opt-in」へ格下げされ、`Ref[File]` 等を意図せず循環に閉じ込めた場合はメモリは回収され fd は閉じず loud に報告される（直し方は `WeakRef` で輪を切る→正常路で deinit が走る）。実装順は reporter 先行→回収追加（どちらも非破壊）。

### 参照カウント方式の静的選択

参照カウントの atomic 性はユーザー向けの型修飾ではなく、コンパイラ内部の所有方式です。概念上は同じ `T` に対して、thread-confined な `RC[T]` と、複数スレッドから寿命を共有できる `AtomicRC[T]` を使い分けます。

- `spawn` へ到達しないと証明された allocation は、軽量な**非 atomic カウント**で作る。
- 親スレッドと spawn 側の双方に alias が残り得る allocation は、最初から**atomic カウント**で作る。
- `unique` 値と、その値だけが所有する参照グラフを別スレッドへ完全に `move` する場合は、同時に複数スレッドからカウントを触らないため**非 atomic のまま移送**できる。

判定は明示された `spawn`/チャネル境界を根とする whole-program の may-analysis です。ローカル変数、フィールド、クロージャ、関数の引数/戻り値、generic、trait の witness、`any P` への注入と取り出しを通して共有制約を allocation まで逆伝播します。条件分岐の一経路だけでも共有され得るなら、その allocation は最初から atomic です。

```plew
val data = loadData()

if shouldParallelize {
    spawn { process(data: data) }
}
```

この `data` は実行時に `shouldParallelize == false` となる場合も含め、`loadData()` 内の対応する allocation から atomic カウントで作られます。同じ生成関数が thread-confined な経路と共有経路の両方から使われる場合、コンパイラは必要な呼び出し文脈だけを内部的に specialized code として生成できます。

Plew は依存パッケージを含めてソースから whole-program コンパイルし、モジュール/パッケージ単位の独立したバイナリ成果物を ABI として接続しません。ビルドキャッシュはこの解析結果を依存情報に含めます。共有制約が呼び出しグラフを遡って変化した場合は依存する caller も再解析・再生成されますが、これはビルド上の挙動であり、ソース上の型や観測意味論は変わりません。

**spawn 境界での動的昇格は行いません。** 既存 alias を持つ値を実行時に非 atomic から atomic へ変えるには、到達グラフの走査、循環処理、既存の retain/release による mode 判定が必要になり、性能目的の spawn にデータサイズ依存の起動コストを持ち込みます。allocation 時点で静的に方式を決めることで、spawn 時のグラフコピー/昇格と、各 retain/release の mode 分岐を避けます。
- 値意味論なので、共有された可変状態は `Ref` 経由のみ生まれる（→ [値・変数・所有権](../01-basics/03-values.md)）。

## 境界を越えるもの（move / copy / Ref / 借用）

`async` と `spawn` は性質が違い、値が境界を越えられるかが変わります。

| 渡すもの | async（単一スレッド） | spawn（実スレッド） |
| --- | --- | --- |
| 値（コピー可能） | ○ コピー（CoW＝遅延・バッファ共有） | ○ 論理コピー（CoW バッファは atomic カウントで物理共有可） |
| `unique` 値 | ○ `move` | ○ `move`（所有を移す） |
| `Ref[T]` | ○（共有・メモリ安全） | ✗（共有可変＋非 atomic 参照カウント＝競合） |
| `fn(...)` | ○ | ✗（sendable 保証なし） |
| `sendable fn(...)` | ○ | ○（キャプチャ環境も静的に RC 方式を選択） |
| 借用 `borrow`/`inout` | ✗ | ✗ |

畳むと **2 つの規則**：

- **借用は同期専用 ── どの境界も越えない**。`async`/`spawn` 関数の引数に `borrow`/`inout` は使えません（指す先のライフタイムが境界をまたぐ複雑さ＝Rust の async 借用問題を避けるため）。越えたいときは **`move`（必要なら戻り値で返す round-trip）・コピー・`Ref`** を使う。
  - 帰結：`Promise.all([f(x: move a), f(x: move a)])` は **2 度目の `move a` が use-after-move でエラー**になり、並行な可変アクセスの footgun が単純な move 追跡で弾ける（借用ライフタイム追跡は不要）。
- **`Ref` は async は越えられるが spawn は越えられない**。async は単一スレッドなので共有してもメモリ安全（ただし await を跨ぐ interleave は JS 同様の**論理ハザード**＝プログラマ責任）。spawn は実スレッドで、`Ref` の共有は **データ競合＋非 atomic な参照カウント破壊**になるため不可。

### CoW 値は論理コピーのまま共有できる

`Array`/`String`/`Dictionary` のような **CoW 値型は内部バッファを参照カウントで共有**します。これらは `Ref` を含まない sendable 型なので spawn を越えられ、値意味論上は送信側と受信側に独立したコピーが渡ります。ただし、読み取り中の物理バッファを境界でディープコピーする必要はありません。

コンパイラは spawn へ共有され得るバッファを allocation 時点から atomic カウントで作ります。spawn は子スレッドを開始する前に論理コピーの retain を完了し、その後は両スレッドが同じ不変バッファを安全に参照できます。どちらかが値を変更すると、通常の CoW と同じく独立バッファを作ってから書き込むため、観測意味論は常に独立コピーです。

- spawn 起動時に到達グラフのコピーや動的昇格は行いません。大きな不変データを渡しても、境界コストはデータサイズに比例しません。
- spawn され得る一経路があるだけでも、その allocation のカウント操作は生成時から atomic になります。実行時にその経路を通らなかった場合も同じです。
- sendability は別途構造的に検査します。atomic カウントは CoW バッファの寿命を安全にするだけで、`Ref` のような共有可変値を sendable にはしません。

## spawn — スレッドの起動

`spawn { … }` は新スレッドを起動してブロックをそこで実行します（起動したスレッド自身もシングルスレッド + イベントループ＝Web Worker 的に内部で `async`/`await` 可）。

```plew
val n = 100                                  // コピー可能
val handle = spawn { give heavy(input: n) }  // n は暗黙コピーでキャプチャ
val result = await handle.join()             // handle: JoinHandle[T]、join(): Promise[T]、await で T
spawn { background() }                        // 束縛しなければ detached
```

- **ベアの `spawn { }` のキャプチャはコピー可能かつ sendable な値のみ（暗黙コピー＝スナップショット）**。`borrow`/`inout`・`Ref`・nonsendable 値を触れば**コンパイルエラー**（nonsendable の根へのフィールド経路を示す）。**`unique` のキャプチャもエラー**（ブロックへ move する構文は当面持たない＝additive）。`spawn` 自体が明示的なスレッド境界なので、ブロックに別の `sendable` 修飾は重ねません。
- **`unique` をスレッドへ渡すには `spawn fn`**（下記）の `move` 引数を使う。
- **戻りはハンドル構造体 `JoinHandle[T]`**（宣言側は `struct JoinHandle[sendable T]`）：`join() -> Promise[T]` を持つ（名前はその役割＝「join するためのハンドル」に由来。スレッドそのものを走らせるのは `spawn` で、この値はそれへのハンドル。Rust の `JoinHandle` と同名）。spawn ブロックが `give`/`return` する `T` もスレッド境界を越えるため sendable 必須。`await handle.join()` で結果。**ランタイムは全スレッド完了まで生存**してから終了する（Go の「main 終了で goroutine kill」footgun を避ける）。
- **`spawn` 内 `panic` はプロセス全体を停止**する（panic=abort・スレッド単位の unwind/catch は無い）。スレッド単位で扱いたい失敗は `spawn fn … -> Result[T, E]` のように **`Result` を自分で返す**＝そのとき `join()` は `Promise[Result[T, E]]` を返す（T がそうだから）。**`join()` が暗黙に `Result` で包むことはありません** ── Rust の `JoinHandle::join() -> Result<T, _>` は panic を捕捉して `Err` 化するが、Plew は panic がプロセスを落とすので捕捉対象が無く、`Promise[T]` を正直にそのまま返す（hidden meaning を作らない）。

### spawn からのトップレベルアクセス

トップレベル `val`/`mut val` と `assoc val` は、プロセスの root event loop に属します。spawn ごとの複製・再初期化は行わず、spawn 本体から直接にも名前付き関数経由にもアクセスできません。

```plew
val config = loadConfig()

fn currentLimit() -> I64 {
    return config.limit
}

spawn {
    print(currentLimit())
    // エラー：currentLimit → config と root-isolated な値へ到達する
}
```

禁止されるのはトップレベル**値への到達**であって、名前付き関数そのものではありません。トップレベル/`assoc` 値へ到達せず、引数とローカル値だけで完結する関数は spawn から呼べます。コンパイラは spawn を根として呼び出しグラフを推移検査し、違反時は呼び出し位置から禁止された値までの経路を示します。これにより、名前付き関数を挟んで sendability 検査を回避することはできません。

### spawn fn — 引数で値を渡してスレッド起動

`unique` を含む値をスレッドへ明示的に渡すには、`async fn` と平行な宣言形 **`spawn fn`** を使う（キャプチャでなく**引数**で境界を越える）。

```plew
spawn fn worker(input: move Data) -> JoinHandle[Report] {   // input を move でスレッドへ
    return analyze(input: input)                            // 戻り値は JoinHandle に自動ラップ
}
val handle = worker(input: move data)                    // 呼び出しがスレッドを起動
val report = await handle.join()
```

- `spawn fn g(...) -> JoinHandle[T]` は呼ぶとスレッドを起動しハンドルを返す（`async fn ... -> Promise[T]` と平行・本体の `return e` を `JoinHandle[T]` に自動ラップ）。**宣言された fn はキャプチャを持たず引数だけ**＝何が境界を越えるか完全に明示。
- 引数は `move`（unique・所有移動）／copy（コピー可能）。いずれも型は sendable でなければならず、`borrow`/`inout`・`Ref`/nonsendable 値は渡せない（境界規則どおり）。戻りの `T` も `JoinHandle` 宣言の `[sendable T]` 制約により sendable 必須。

## sendable / nonsendable

**spawn 境界を越えられるのは sendable 値だけ**です。

- `Ref` は組み込みの **`nonsendable struct`**。ユーザー定義型も `nonsendable struct` と宣言できる。nonsendable フィールド・enum payload・generic 実引数を含む型には性質が構造的に自動伝播する（→ [値・変数・所有権](../01-basics/03-values.md#sendable--nonsendableスレッド間の移送可能性)）。
- 通常の `fn(...)` は sendable 保証を持たず、スレッドへ送る関数値は宣言地点で `sendable fn(...)` と明示する。`sendable fn` から `fn` への保証消去だけ暗黙に許す（→ [関数](../01-basics/04-functions.md#sendable-クロージャ)）。
- spawn のキャプチャ／チャネルで送る値は **sendable であること**。違反は**そのキャプチャ／送信地点でコンパイルエラー**（nonsendable 宣言または関数フィールドへの経路を示す）。
- ジェネリックで spawn するときは `[sendable T]` で T の sendability を明示的に要求する（→ [ジェネリクス](../02-type-system/06-generics.md)）。
- async（単一スレッド）では nonsendable 値や通常の `fn` も自由＝この制約は **spawn 境界だけ**に効く。

## チャネル

スレッド間で値を送るにはチャネルを使います（「共有」ではなく「送信」＝*share memory by communicating*）。**具体的な型・API はコアライブラリ送り**ですが、モデルは確定しています：

- チャネルは**スレッド安全なプリミティブ**で、概念上 `struct Sender[sendable T]` / `struct Receiver[sendable T]` と宣言する。ハンドル自体も sendable（`Ref` と違い内部が atomic なので spawn を越えられる）。
- **送る値は sendable であること**（move/copy で渡る・`Ref` や通常の `fn` は送れない）。これで送信経由でもスレッド間に共有可変が生まれず race-free を保つ。
- 方針は**複数 Sender**。所有権で「Receiver は単一所有」を強制はしない（複数箇所が持て、`receive()` も複数回呼べる）。`receive() -> Result[T, ChannelClosed]` 方向。マルチコンシューマの意味論は core-lib 設計で詰める。

## 並行安全性 ── 実質 race-free

Plew は次を保証します：

- **スレッド間に共有可変状態が存在しない**：spawn は sendable な値の論理コピーまたは unique な所有移動だけを受け取り、借用・`Ref` は越えません。トップレベル/`assoc` 値にも[直接・推移とも到達できません](#spawn-からのトップレベルアクセス)。CoW の物理バッファなど、sendable な不変ストレージはスレッド間で共有できますが、その allocation は[静的に atomic カウントを選ぶ](#参照カウント方式の静的選択)ため寿命管理も競合しません。よって**データ競合が原理的に起きず、UB も無い**。`Mutex` / `sync val` も不要。
- **唯一の注意は async の interleave**：単一スレッドで 1 つの `Ref` を複数の async タスクが触ると、await を跨いで状態が割り込まれ得る（**メモリ安全だが論理ハザード**＝JS の共有オブジェクトと同じ・プログラマ責任）。
- どうしてもスレッド間で可変共有したい稀なケースは、atomic 参照カウントの thread-safe 共有型（Rust の `Arc`/`Mutex` 相当）を **additive** に後付けする想定。大半はチャネルで足りる。
