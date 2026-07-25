# 非同期処理とメモリ管理

Plew は JavaScript と同様の**シングルプロセス・シングルスレッド + イベントループ**を基盤とします。重い処理を別スレッドで動かしたいときだけ `spawn` でスレッドを立てます。メモリは **ARC（自動参照カウント）** で管理し、通常実行中の所有者喪失による破棄は決定的（[`deinit`](../01-basics/03-values.md#deinit)）、循環は [`WeakRef`](../01-basics/03-values.md#ref--weakref共有可変) で断ち切ります。参照カウントの更新は通常は非 atomic で、複数スレッドから共有され得る allocation だけをコンパイラが atomic にします。

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

- **戻り型は `Promise[T]` と明示して書き**、本体の `return e`（`e: T`）はコンパイラが `Promise[T]` に包みます（TypeScript と同じ）。`await` はその `Promise[T]` を `T` に開きます。`Promise[T]` はイベントループに所属するため、`T` が sendable でも **常に nonsendable** です。`Promise[T]` は JavaScript の Promise と同じく同一イベントループ上の完了を共有するコピー可能 handle であり、`await` は handle を消費しません。同じ `Promise[T]` は複数回 `await` でき、完了後は completion cell の結果を各 `await` ごとにコピーして返します（v1 の `T` はコピー可能型に限定）。
- 戻り型を省略した `async fn`（`main` など）は、値を返さない `Promise` を返します。
- **`async` は単一スレッド上の協調的中断**で、別スレッドは起きません。所有権・借用は同期コードと同じに効きます（`unique` 値を await を跨いで保持してもよい）。

### async メソッドと self

`async fn` のメソッドは **self を借用できません**（借用は async 境界を越えられない＝`inout fn` の async 版は不可）。よって：

- **非消費の async メソッドは self をコピー**する（Self がコピー可能なときのみ）。＝Self がコピー可能必須で、generic struct なら型引数もコピー可能（[`allowUnique` でない](../02-type-system/06-generics.md)）。
- **消費する async メソッドは `async move fn`**（self を move）。
- **await を跨いで self を可変共有したいオブジェクトは `Ref` 裏打ち**にする ── `ref->asyncMutate()` は `Ref` 越しに self を触り、`Ref` は async 境界を越えられる（単一スレッドでメモリ安全・interleave は JS 同様の論理ハザード）。ステートフルな async オブジェクトは Ref 裏打ち＝JS のオブジェクトと同じ姿。

### unique 結果と `allowUnique`（v1 は不可・将来）

v1 では **`Promise[T]`/`JoinHandle[T]` を含むコアの generic 型の型引数はすべてコピー可能な型に限定**（`allowUnique` 未導入）。これは型引数の `unique` 制約であって、コンテナ自身のコピー可否や sendability とは別です。`Promise[T]` は単一スレッド内で使うため nonsendable な `T` も許容します。一方、別スレッドから結果を受け取る `JoinHandle` は概念上 **`unique struct JoinHandle[sendable T]`** で、`T` の sendability を要求し、ハンドル自身は sendable です。ただし v1 では **unique 値の spawn 境界 move 自体を持たない**ため、`JoinHandle` をさらに別スレッドへ move することはできません。帰結：

- async/spawn の**本体は unique な `T` を結果として返せない**（`-> Promise[unique]` / `-> JoinHandle[unique]` 不可）。`spawn` 呼び出し自体が呼び出し側に作る外側の `JoinHandle[T]` は unique ですが、そのハンドルを spawn 境界越しに move することも v1 では不可です。async で unique を持ち回るなら **`Ref` 包み**（`Ref` は async を越えられる）。
- **unique を generic に入れるのは常に `Ref` 包み**（`Optional[Ref[File]]`・`Array[Ref[File]]`）。`Ref` はコピー可能なので通常のコピー可能なコレクションになり、`match`/peek/反復が普通に効く（move-out 専用 API も `take()` も不要）。
- by-value の unique を generic で扱う（`Optional[unique]`・`Promise[unique]`・`Array[unique]`）には **借用束縛＝ライフタイム**が要り（要素を取り出さず借用で覗く操作のため）、v1 の非 escape 借用では実装できない。**`allowUnique` は将来の additive**（保持系＝Optional/Promise が先・Array/Iterable は escaping borrow 導入後）。それまでは `Ref` 包みで代替。

## メモリ管理（ARC）

- **ARC（自動参照カウント）で管理**：通常実行中にスコープを抜けて最後の所有者／`Ref` が消えると**即座に**解放され、`deinit` が走る（決定的破棄＝ファイル・ソケット等の資源解放に使える）。ただし**`panic` 時は abort で巻き戻さないので `deinit` は走らない**（OS 管理資源は OS が回収するが、ユーザー定義の意味的 cleanup は保証しない・[制御構造 § panic と発散](../03-expressions/11-control-flow.md#panic-と発散) 参照）。決定的破棄が保証されるのは通常実行中の正常経路（スコープ離脱・`return`・`try`/`Result` 伝播・再代入など）だけで、プロセス終了時のトップレベル/`assoc val` 所有根には保証しない。
- **循環は `WeakRef` で**：参照カウントは循環を回収しないので、親子の逆リンク等は `WeakRef` で断ち切る。**循環が生じ得るのは `Ref` グラフだけ**（値世界＝`Array`/`String`/`Dictionary`・[自動箱化された再帰値型](../02-type-system/05-structs-enums.md#再帰的な値型)は構造上つねに木／DAG）なので、純粋 ARC は値世界を取りこぼさない。
  > **将来 additive：循環の自動回収。** `Ref` グラフは小さく隔離され（trace 対象は `Ref` ボックス＋`mut val` 参照キャプチャしたクロージャだけ）、しかも **`Ref` は非 atomic かつ nonsendable**（spawn を越えない＝per-thread）・**単一イベントループ**（ターン間が天然のセーフポイント）・フルマネージドなので、ARC の上に **`Ref` グラフ限定のサイクルコレクタ**（Bacon–Rajan の trial deletion・CPython/PHP で実証）を **per-thread・idle 実行**で非破壊に足せる。**ゴミ循環を検出したときの挙動**は、(1) retain path 付きで **loud に報告**（dedup・診断フック経由・**全ビルド共通＝release 含む**＝`assert`/overflow panic と同じく「意味論を変えるビルドを持たない」原則の一貫）、(2) **メモリを回収**、(3) **循環メンバの `deinit` は走らせない**。`deinit` の決定的契約は正常路（スコープ離脱・`return`・`try`/`Result` 伝播・再代入など）のみで保証し、`panic`・`Process.exit`・リークした循環・process/runtime teardown のトップレベル/`assoc val` root は契約外として deinit を飛ばす（[panic と発散](../03-expressions/11-control-flow.md#panic-と発散) と対称）。**deinit の有無で挙動を分けない**ので「構造体に `deinit` を足したら循環がリークし始める」崖は生じず、循環メンバの deinit が予期しない時刻に走る驚きも生じない（走らないだけ）。重大度は値の嘘ではない（資源の無駄＋deinit skip）ので panic でなくログ。手動 `WeakRef` は「正しさのため必須」から「決定性・性能の opt-in」へ格下げされ、`Ref[File]` 等を意図せず循環に閉じ込めた場合はメモリは回収され fd は閉じず loud に報告される（直し方は `WeakRef` で輪を切る→正常路で deinit が走る）。実装順は reporter 先行→回収追加（どちらも非破壊）。

### 参照カウント方式の静的選択

参照カウントの atomic 性はユーザー向けの型修飾ではなく、コンパイラ内部の所有方式です。概念上は同じ `T` に対して、thread-confined な `RC[T]` と、複数スレッドから寿命を共有できる `AtomicRC[T]` を使い分けます。

- 実スレッド境界（`spawn`・チャネル）へ到達しないと証明された allocation は、軽量な**非 atomic カウント**で作る。
- 境界の両側に alias が残り得る allocation は、最初から**atomic カウント**で作る。
- v1 では unique 値の spawn 境界 move は持たない。将来 `allowUnique` と境界 move を追加する場合、親側 alias が残らない完全移送だけは非 atomic のまま移送できる余地がある。
- 不変トップレベル/`assoc val` が固定的に所有する allocation はプロセス寿命の **immortal** とし、非 atomic/atomic のいずれの retain/release も行わない。spawn から読まれても寿命は変わらない。

判定は明示された `spawn`/チャネル境界を根とする whole-program の may-analysis です。ローカル変数、フィールド、クロージャ、関数の引数/戻り値、generic、trait の witness、`any P` への注入と取り出しを通して共有制約を allocation まで逆伝播します。条件分岐の一経路だけでも共有され得るなら、その allocation は最初から atomic です。

同じ値スロットや retain/release 点へ atomic と非 atomic の両 allocation が合流し得る場合は、その合流へ到達する allocation をすべて atomic とします。これにより、各 retain/release 点の方式は静的に一意となり、実行時の mode tag や mode 分岐を要しません。呼び出し文脈の specialization はこの同質性を保つための最適化であり、動的な合流を未判定のまま残す手段ではありません。

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
| sendable な値（コピー可能） | ○ コピー（CoW＝遅延・バッファ共有） | ○ 論理コピー（CoW バッファは atomic カウントで物理共有可） |
| sendable な `unique` 値 | ○ `move` | ✗（v1 は unique の境界 move 自体を持たない） |
| nonsendable な値 | ○ | ✗ |
| `Ref[T]` / `WeakRef[T]` | ○（共有・メモリ安全） | ✗（per-thread の共有可変グラフへ到達するため） |
| `fn(...)` | ○ | ✗（sendable 保証なし） |
| `sendable fn(...)` | ○ | ○（キャプチャ環境も静的に RC 方式を選択） |
| `Promise[T]` | ○ | ✗（イベントループに所属し、`T` に関係なく nonsendable） |
| `JoinHandle[T]`（`T` は sendable） | ○ `move` | ✗（unique handle の境界 move は将来） |
| 借用 `borrow`/`inout` | ✗ | ✗ |

畳むと **2 つの規則**：

- **借用は同期専用 ── どの境界も越えない**。`async`/`spawn` 関数の引数に `borrow`/`inout` は使えません（指す先のライフタイムが境界をまたぐ複雑さ＝Rust の async 借用問題を避けるため）。async では **`move`（必要なら戻り値で返す round-trip）・コピー・`Ref`** を使い、spawn では v1 の間 **コピー可能かつ sendable な値の論理コピー**だけを使います。
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
val result = await handle.join()             // join() が unique handle を消費し、呼出側ループの Promise[T] を返す
spawn { background() }                        // 束縛しなければ detached
```

- **ベアの `spawn { }` のキャプチャはコピー可能かつ sendable な値のみ（暗黙コピー＝スナップショット）**。`borrow`/`inout`・`Ref`・nonsendable 値を触れば**コンパイルエラー**（nonsendable の根へのフィールド経路を示す）。**`unique` のキャプチャもエラー**（v1 では spawn 境界を越える move 自体を持たない）。`spawn` 自体が明示的なスレッド境界なので、ブロックに別の `sendable` 修飾は重ねません。
- **`spawn fn` の引数も v1 ではコピー可能かつ sendable な値のみ**です。`spawn fn` は「キャプチャでなく引数として何をスレッド境界へコピーするか」を明示する構文であり、unique を move する構文ではありません。
- **戻りは sendable かつ unique なハンドル `JoinHandle[T]`**（概念上 `unique struct JoinHandle[sendable T]`）。ハンドルは「結果を一度だけ受領または破棄する責任」を所有します。`move fn join() -> Promise[T]` は self を消費し、**呼び出したスレッドのイベントループ**を結果の完了先として登録します。完成した `T` はそのループの `Promise[T]` へ move されるため、コピーや二重 join はできません。ハンドル自身は sendable な設計ですが、v1 では unique 値を spawn 境界へ move できないため、作成された `JoinHandle` は作成側スレッドで join/drop します（将来 `allowUnique` と境界 move を導入すれば責任を別スレッドへ移せる）。
- **worker の完了は、spawn 本体が `give`/`return` した時点ではなく、その後 worker 自身のイベントループも drain してスレッドが終了可能になった時点**です。本体が先に `T` を生成した場合、ランタイムの同期された completion cell が `T` を所有したまま worker loop の drain を待ちます。本体が登録したタイマ・タスク・リスナが残る限り `join()` は完了せず、永続リスナなら明示的に解除されるまで待ち続けます。したがって `JoinHandle` は本体の戻り値だけでなく、実スレッドの寿命を join します。
- **join せずに `JoinHandle` を drop すると detached** になります。drop はハンドルを消費し、**drop が実行されたスレッドのイベントループ**を「結果を破棄する完了先」として登録します。worker がすでに完了していれば completion cell 内の `T` をそのループへ move して破棄し、実行中なら worker の本体とイベントループが完走した後に同じことを行います。join/drop と worker 完了が競合しても同期された atomic な状態遷移により、完了先の登録と `T` の move または破棄はちょうど一度だけです。drop 自体は待たずに戻りますが、この破棄継続は完了まで所有側イベントループの pending work として残り、ループは park して通知を待てます。
- spawn ブロックが `give`/`return` する `T` はスレッド境界を越えるため sendable 必須です。`join()` が返した `Promise[T]` の全コピーを結果観測前に破棄した場合も暗黙キャンセルはせず、同じイベントループ上で結果だけを破棄します。**ランタイムはハンドルの有無にかかわらず全スレッド完了と登録済み完了先の処理まで生存**してから終了します（Go の「main 終了で goroutine kill」footgun を避ける）。
- **`spawn` 内 `panic` はプロセス全体を停止**する（panic=abort・スレッド単位の unwind/catch は無い）。スレッド単位で扱いたい失敗は `spawn fn … -> Result[T, E]` のように **`Result` を自分で返す**＝そのとき `join()` は `Promise[Result[T, E]]` を返す（T がそうだから）。**`join()` が暗黙に `Result` で包むことはありません** ── Rust の `JoinHandle::join() -> Result<T, _>` は panic を捕捉して `Err` 化するが、Plew は panic がプロセスを落とすので捕捉対象が無く、`Promise[T]` を正直にそのまま返す（hidden meaning を作らない）。

### spawn からのトップレベルアクセス

トップレベル/`assoc` 値はプロセス全体で一度だけ初期化され、spawn ごとの複製・再初期化は行いません。ただし、spawn からの読み取り可否は可変性と sendability で決まります。

- sendable な不変 `val` / `assoc val` は、コピー可能か unique かを問わず、spawn 本体から直接にも名前付き関数経由にも読み取れる。トップレベル名へのアクセスはキャプチャや境界越しの借用ではなく、プロセス寿命が保証された共有不変ストレージの直接参照として扱う。
- 不変 unique 値に許されるのは**所有権を消費しない操作だけ**である。通常の不変 `fn` 呼び出しとコピー可能なフィールドのコピーはできるが、値全体の by-value 受け渡し、`move`、`inout fn` / `move fn`、unique フィールドの move-out はできない。グローバルの place が唯一の所有者であり続け、spawn からの読み取りによって所有者は増えない。
- `mut val` は、読み取りだけでも spawn から到達できない。
- nonsendable な `val` / `assoc val` は spawn から到達できない。

不変トップレベル/`assoc val` のストレージと、それが不変値として固定的に所有する backing はプロセスと同じ寿命を持つ **immortal allocation** です。immortal な所有辺には寿命管理の retain/release が不要で、spawn から読まれても atomic RC へ昇格しません。`mut val` や nonsendable なトップレベル値はこの retain/release 省略の対象ではなく、通常実行中の再代入では旧値が通常どおり drop されます。ただし現在値もトップレベル所有根であるため、プロセス終了時に user `deinit` やフィールド破棄は保証しません。これは Swift/Rust と同じく、終了時専用の finalization 順序・閉店モード・`deinit` 依存グラフを言語仕様に持ち込まないためです。外部資源や OS には分からない意味的な片付けが必要な値は、トップレベル所有根ではなく `main` の通常スコープ、明示的な `close`/`shutdown`、または with-style のライフサイクル API に置きます。CoW 値をローカルへコピーしても immortal backing をそのまま共有でき、ローカル側を変更するときだけ通常どおり新しい backing を作ってから書き込みます。`Ref` の中へ後から格納される値など、可変な内部状態の allocation は immortal へ伝染せず通常どおり管理しますが、トップレベル所有根の `Ref` が最後まで保持している cell と pointee は process exit で user `deinit` を保証しません。immortal であることを表す sentinel、header、静的 provenance などは観測できない内部方式です。

トップレベル初期化中に spawn worker が不変トップレベル/`assoc val` を読む場合も、worker 側で初期化子を実行しません。未初期化の値なら root event loop 上の同じ memoize サンクを force し、worker は完了を待ってから初期化済みの immortal 値を読みます。worker を含む待ち合わせの輪ができた場合は、トップレベル初期化循環として検知した時点で panic します（→ [トップレベル await と並行初期化](15-modules.md#トップレベル-await-と並行初期化)）。

```plew
val config: Config = loadConfig() // Config は sendable
mut val requestCount = 0

fn currentLimit() -> I64 {
    return config.limit
}

fn incrementRequestCount() {
    requestCount += 1
}

spawn {
    print(currentLimit())      // OK：config は不変かつ sendable
    incrementRequestCount()   // エラー：incrementRequestCount → requestCount
}
```

名前付き関数に `concurrent` 修飾は付けません。コンパイラは関数の通常の解析時に effect summary を作り、spawn を根とする**実行依存グラフ**で保存済み summary を推移検査します。実行依存には直接/間接呼び出しだけでなく、動的な trait witness、operator、factory、デフォルト式、フィールド初期化子、`deinit`/drop glue など、そのコードから暗黙に実行され得るユーザーコードを含みます。違反時は spawn の呼び出し位置から禁止された値までの経路を示します。これにより、名前付き関数や暗黙実行を挟んで検査を回避することはできません。

### spawn fn — 引数で値をコピーしてスレッド起動

キャプチャでなく引数でスレッド境界を越える値を明示したいときは、`async fn` と平行な宣言形 **`spawn fn`** を使います。v1 では引数はコピー可能かつ sendable な値に限られ、`move` 引数や `unique` 値は渡せません。

```plew
spawn fn worker(input: Data) -> JoinHandle[Report] {   // input を論理コピーでスレッドへ
    return analyze(input: input)                       // 戻り値は JoinHandle に自動ラップ
}
val handle = worker(input: data)                         // 呼び出しがスレッドを起動
val report = await handle.join()
```

- `spawn fn g(...) -> JoinHandle[T]` は呼ぶとスレッドを起動しハンドルを返す（`async fn ... -> Promise[T]` と平行・本体の `return e` を `JoinHandle[T]` に自動ラップ）。**宣言された fn はキャプチャを持たず引数だけ**＝何が境界を越えるか完全に明示。
- 引数はコピー可能かつ sendable でなければならず、`move`・`borrow`/`inout`・`Ref`/nonsendable 値は渡せない（境界規則どおり）。戻りの `T` も `JoinHandle` 宣言の `[sendable T]` 制約により sendable 必須で、v1 ではコピー可能型に限られます。

## sendable / nonsendable

**spawn 境界を越えられるのは sendable 値だけ**です。

- `Ref` は組み込みの **`nonsendable struct`**。ユーザー定義型も `nonsendable struct` / `nonsendable enum` と宣言できる。nonsendable フィールド・enum payload・generic 実引数を含む型には性質が構造的に自動伝播する（→ [値・変数・所有権](../01-basics/03-values.md#sendable--nonsendableスレッド間の移送可能性)）。
- `Promise[T]` はイベントループに所属するため常に nonsendable。`JoinHandle[T]` は `T` に sendable を要求するスレッド安全な組み込みハンドルで、ハンドル自身も sendable ですが、join 権を一つに保つため unique です。
- 通常の `fn(...)` は sendable 保証を持たず、スレッドへ送る関数値は宣言地点で `sendable fn(...)` と明示する。`sendable fn` から `fn` への保証消去だけ暗黙に許す（→ [関数](../01-basics/04-functions.md#sendable-クロージャ)）。
- 無印の存在型 `any P` は sendability 保証を消去するため nonsendable。境界を越える存在型は `any sendable P` と明示し、注入する具体型も sendable でなければならない。`any sendable P` → `any P` の保証消去だけ暗黙に許す（→ [トレイト](../02-type-system/08-traits.md#存在型の-sendability)）。
- spawn のキャプチャ／チャネルで送る値は **sendable であること**。違反は**そのキャプチャ／送信地点でコンパイルエラー**（nonsendable 宣言または関数フィールドへの経路を示す）。
- ジェネリックで spawn するときは `[sendable T]` で T の sendability を明示的に要求する（→ [ジェネリクス](../02-type-system/06-generics.md)）。
- async（単一スレッド）では nonsendable 値や通常の `fn` も自由＝この制約は **実スレッド境界（spawn・チャネル）だけ**に効く。

## チャネル

スレッド間で値を送るにはチャネルを使います（「共有」ではなく「送信」＝*share memory by communicating*）。**具体的な型・API はコアライブラリ送り**ですが、モデルは確定しています：

- チャネルは**スレッド安全なプリミティブ**で、概念上 `struct Sender[sendable T]` / `struct Receiver[sendable T]` と宣言する。ハンドル自体も sendable（`Ref` と違い内部が atomic なので spawn を越えられる）。
- **v1 で送る値はコピー可能かつ sendable であること**（論理コピーで渡る・`unique`、`Ref`、通常の `fn` は送れない）。core generic はまだ `allowUnique` を持たず、spawn 境界の unique move 自体も持たないため、チャネルでの unique move は将来 additive とする。これで送信経由でもスレッド間に共有可変が生まれず race-free を保つ。
- 方針は**複数 Sender**。所有権で「Receiver は単一所有」を強制はしない（複数箇所が持て、`receive()` も複数回呼べる）。`receive() -> Result[T, ChannelClosed]` 方向。マルチコンシューマの意味論は core-lib 設計で詰める。

## 並行安全性 ── 実質 race-free

Plew は次を保証します：

- **ユーザー可視の通常値として共有可変状態を作れない**：spawn はコピー可能かつ sendable な値の論理コピーだけを受け取り、unique・借用・`Ref` は越えません。[可変／nonsendable なトップレベル/`assoc` 値にも直接・推移とも到達できません](#spawn-からのトップレベルアクセス)。sendable な不変グローバルは immortal、CoW の物理バッファなど通常の sendable な不変ストレージは必要に応じて[静的に atomic カウントを選ぶ](#参照カウント方式の静的選択)ため、いずれも寿命管理が競合しません。atomic refcount とチャネル内部の同期状態はユーザー値として観測・変更できないランタイム機構です。よって**データ競合が原理的に起きず、UB も無い**。`Mutex` / `sync val` も不要。
- **唯一の注意は async の interleave**：単一スレッドで 1 つの `Ref` を複数の async タスクが触ると、await を跨いで状態が割り込まれ得る（**メモリ安全だが論理ハザード**＝JS の共有オブジェクトと同じ・プログラマ責任）。
- どうしてもスレッド間で可変共有したい稀なケースは、atomic 参照カウントの thread-safe 共有型（Rust の `Arc`/`Mutex` 相当）を **additive** に後付けする想定。大半はチャネルで足りる。
