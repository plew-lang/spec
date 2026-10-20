# 非同期処理とメモリ管理

Plew は JavaScript と同様の**シングルプロセス・シングルスレッド + イベントループ**を基盤とします。重い処理を別スレッドで動かしたいときだけ `spawn` でスレッドを立てます。メモリは **ARC（自動参照カウント）** で管理し、通常実行中の所有者喪失による破棄は決定的（[`deinit`](../01-basics/03-values.md#deinit)）、循環は [`WeakRef[R]`](../01-basics/03-values.md#ref--mutableref--weakref共有参照) で断ち切ります。AOTの参照カウントは非atomicを基本とし、複数スレッドから共有され得るallocationをコンパイラが生成時からatomicにします。JITは原則atomicとし、意味とruntime実装は共通です。

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

- 戻り型は`Promise[T]`と明示し、本体の`return e`（e: T）をその結果へ包みます。戻り型を省略したasync fnは`Promise[()]`を返します。Promiseは同じイベントループに所属し、Tによらずnonsendableです。
- Promiseは結果Tを所有する`autounique`型です。コピー可能なTならPromiseもコピー可能で、各awaitへ結果の論理コピーを返します。uniqueなTならPromiseもuniqueで、一度だけ消費して結果の所有権を取り出します。所有localでは`await move promise`と明示し、以後その束縛は使えません。新しい所有結果を直接awaitする場合も通常の所有権規則に従います。
- awaitは中断し得る地点であり、必ずイベントループへ制御を返す操作ではありません。完了済みPromiseは待たずに結果を受け取り、そのまま続行します。未完了の場合は待機を登録して中断し、完了後に所属ループで再開します。チャネル等もこの共通規則に従います。
- asyncは単一worker内の協調的実行で、別スレッドを起動しません。uniqueな所有値はawaitを跨いで保持できますが、borrow/inoutの束縛はasync境界を越せません。`async diverge fn`は認めません。

### async メソッドと self

asyncメソッドはselfを借用できません。非消費ならコピー可能なSelfをcopy-selfで渡し、消費するなら`async move fn`で所有します。inout async selfはありません。allowuniqueなSelfの消費指定は[具体化時のモード適応](../02-type-system/06-generics.md#所有モードとコピー可能な具体化)に従います。

同じworker内で状態を共有したい場合は、コピー可能なMutableRef等のハンドルを使います。await前後で他の処理が状態を変える論理的なinterleaveはあり、これはデータ競合やメモリ破損とは別です。共有先のuniqueな本体をasync selfへ暗黙moveすることはできません。

### unique結果と未受領結果

Promiseの公開ハンドルと、進行中の処理・待機者が持つ内部所有を区別します。公開ハンドルのdropは取消ではありません。未受領の結果は、所属ループ上で結果を一度だけ破棄する責務として残ります。unique Promiseの消費awaitでは受領権が待機activationへ移り、元ハンドルを失ったことを放棄と誤認しません。一般のPromise取消や実行時のunique二重取得を許すAPIは提供しません。

## メモリ管理（ARC）

- **ARC（自動参照カウント）で管理**：通常実行中にスコープを抜けて最後の所有者／共有参照が消えると**即座に**解放され、`deinit` が走る（決定的破棄＝ファイル・ソケット等の資源解放に使える）。ただし**`panic` 時は abort で巻き戻さないので `deinit` は走らない**（OS 管理資源は OS が回収するが、ユーザー定義の意味的 cleanup は保証しない・[制御構造 § 発散する関数と panic](../03-expressions/11-control-flow.md#発散する関数と-panic) 参照）。決定的破棄が保証されるのは通常実行中の正常経路（スコープ離脱・`return`・`try`/`Result` 伝播・再代入など）だけで、プロセス終了時のトップレベル/`assoc val` 所有根には保証しない。
- **循環は `WeakRef[R]` で**：参照カウントは循環を回収しないので、親子の逆リンク等は弱参照で断ち切る。**循環が生じ得るのは共有参照グラフだけ**（値世界＝`Array`/`String`/`Dictionary`・[自動箱化された再帰値型](../02-type-system/05-structs-enums.md#再帰的な値型)は構造上つねに木／DAG）なので、純粋 ARC は値世界を取りこぼさない。
  > **将来 additive：循環の自動回収。** 共有参照グラフは小さく隔離され（trace対象は標準共有セル・通信／completion stateと共有環境などの共有identityグラフ）、単一イベントループではターン間が天然のセーフポイントになり、フルマネージドなので、ARC の上に **共有参照グラフ限定のサイクルコレクタ**（Bacon–Rajan の trial deletion・CPython/PHP で実証）を idle 実行で非破壊に足せる。実スレッドを越え得るFrozenRef / MutexRef等のcell は atomic refcount 対象になり得るため、回収実装は refcount mode と同じく allocation の所属・共有可能性を内部で区別する。**ゴミ循環を検出したときの挙動**は、(1) retain path 付きで **loud に報告**（dedup・診断フック経由・**全ビルド共通＝release 含む**＝`assert`/overflow panic と同じく「意味論を変えるビルドを持たない」原則の一貫）、(2) **メモリを回収**、(3) **循環メンバの `deinit` は走らせない**。`deinit` の決定的契約は正常路（スコープ離脱・`return`・`try`/`Result` 伝播・再代入など）のみで保証し、`panic`・`Process.exit`・リークした循環・process/runtime teardown のトップレベル/`assoc val` root は契約外として deinit を飛ばす（[panic と発散](../03-expressions/11-control-flow.md#発散する関数と-panic) と対称）。**deinit の有無で挙動を分けない**ので「構造体に `deinit` を足したら循環がリークし始める」崖は生じず、循環メンバの deinit が予期しない時刻に走る驚きも生じない（走らないだけ）。重大度は値の嘘ではない（資源の無駄＋deinit skip）ので panic でなくログ。手動の弱参照は「正しさのため必須」から「決定性・性能の opt-in」へ格下げされ、共有参照で `File` 等を意図せず循環に閉じ込めた場合はメモリは回収され fd は閉じず loud に報告される（直し方は弱参照で輪を切る→正常路で deinit が走る）。実装順は reporter 先行→回収追加（どちらも非破壊）。

### 参照カウント方式の静的選択

以下のwhole-programによる方式選択はAOTの規則です。JITの基準は本節末尾に区別します。参照カウントの atomic 性はユーザー向けの型修飾ではなく、コンパイラ内部の所有方式です。概念上は同じ `T` に対して、thread-confined な `RC[T]` と、複数スレッドから寿命を共有できる `AtomicRC[T]` を使い分けます。

- 実スレッド境界（`spawn`・チャネル）へ到達しないと証明された allocation は、軽量な**非 atomic カウント**で作る。
- 境界の両側に alias が残り得る allocation は、最初から**atomic カウント**で作る。
- unique値のmoveでも内部CoW別名を考慮します。完全に独立した移送は非atomicを維持できますが、共有が残る部分には生成元からatomicを要求します。
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
- 値意味論と共有identityを区別します。worker内の共有可変はMutableRef、worker間の共有可変はMutexRefのロック契約へ接続します（[参照型](../01-basics/03-values.md#ref--mutableref--weakref共有参照)）。

JITでは、後から要求される本体のために既存allocationの方式を変更しないよう、通常の管理allocationと暗黙のcopy/drop・weak操作を原則atomic RCで統一します。証明済みの非atomic化は追加の最適化であり、安全性の前提にしません。AOTでは共有可能性を生成元・別名・引数と結果の関係へ伝え、合流するallocationは生成時からatomicへ揃えます。必要な物理特殊化だけを作り、型の意味・overload解決やbuild/runのruntime実装は分けません。spawn等の共有境界がないAOTプログラムでは通常allocationをすべて非atomicとします。immortalな所有根は別の寿命方針です。

## 境界を越えるもの（move / copy / 共有参照 / 借用）

| 値 | async（同じworker） | spawn・チャネル・worker結果 |
|---|---|---|
| コピー可能かつsendable | 論理コピー | 論理コピー。CoWの物理共有は可能 |
| uniqueかつsendable | move | moveで所有権移送 |
| nonsendable | 使用可能 | 不可 |
| Ref / MutableRef、それらへ昇格するWeakRef | 同じworker内で使用可能 | 不可 |
| FrozenRef[T] / MutexRef[T]とそのWeakRef | 使用可能 | Tがsendableならハンドルを共有 |
| MutexGuard[T] | 所有してawait可能 | 不可 |
| Promise[T] | 所属ループで使用 | 不可 |
| JoinHandle[T] | uniqueな所有 | Tがsendableならmove可能 |
| 通常fnの値 | 使用可能 | 不可。明示sendable fnの保証が必要 |
| 存在型 | 保持するコピー能力に従う | 明示sendable保証とそのコピー／move能力が必要 |
| borrow / inoutの束縛 | async境界は越せない | 不可 |
| FFI raw pointer / opaque handle | 外部契約に従う | 既定sendable。外部の寿命・同期はFFI利用者の責任 |

送信可能性は型全体の性質であり、コピー・move・捕捉の適合性は別に検査します。nonsendableをFrozenRef/MutexRefで包んでも送れません。同期のborrow/inout呼び出しから戻った後に独立した所有値を送ることと、アクセス権自体を送ることは別です。

### CoW 値は論理コピーのまま共有できる

String・Array・Dictionary等の送信は論理コピーであり、内部バッファの物理共有を許します。共有され得るallocationと、その到達する内部バッファは生成時からatomic RCで保持し、送信側のretainを子workerの使用開始前に終えます。変更時は通常のCoWで独立した値を保ちます。境界でのディープコピーや既存allocationの動的昇格は原則として要求しません。

moveは外側の所有を移すだけで、内部CoWバッファに残る別名まで消す保証ではありません。共有が残る部分は同じRC方式で扱います。FrozenRef/MutexRefのハンドル送信は、通常値のCoWとは別に意図した共有identityを保持します。

## spawn — スレッドの起動

spawnブロックは新workerを起動し、そのworkerも単一スレッドとイベントループを持ちます。

```plew
val n = 100I64
val handle = spawn { give heavy(input: n) }
val result = await handle.join()
```

暗黙捕捉できるのはコピー可能かつsendableな不変valだけです。mut valは読み取りだけでも直接捕捉できず、必要なら外側で不変valへ明示コピーします。borrow/inoutやunique値の暗黙捕捉は認めません。spawnブロックへのmove捕捉構文は提供せず、unique入力はspawn fnのmove引数を使います。通常クロージャへのunique move captureもこの機能には含めません。

JoinHandle[allowunique sendable T]はjoin権を一つに保つuniqueかつsendableなハンドルです。`join()`はhandleを消費し、**joinを呼んだworker**のループを結果の完了先として登録し、Promise[T]を返します。unique結果はそのPromiseの消費awaitで受領します。

worker完了は本体のgive/returnだけではなく、自身のイベントループがdrainして終了した時点です。joinしないhandleのdropはworkerを中断せず、dropしたworkerのループを未受領結果の破棄先に登録します。完了先のループ・コード・runtime資源は配送と一度だけの受領／破棄が終わるまで保持します。作成workerとjoin/dropしたworkerを同一と仮定しません。

### spawn からのトップレベルアクセス

トップレベル/`assoc` 値はプロセス全体で一度だけ初期化され、spawn ごとの複製・再初期化は行いません。ただし、spawn からの読み取り可否は可変性と sendability で決まります。

- sendable な不変 `val` / `assoc val` は、コピー可能か unique かを問わず、spawn 本体から直接にも名前付き関数経由にも読み取れる。トップレベル名へのアクセスはキャプチャや境界越しの借用ではなく、プロセス寿命が保証された共有不変ストレージの直接参照として扱う。
- 不変 unique 値に許されるのは**所有権を消費しない操作だけ**である。通常の不変 `fn` 呼び出しとコピー可能なフィールドのコピーはできるが、値全体の by-value 受け渡し、`move`、`inout fn` / `move fn`、unique フィールドの move-out はできない。グローバルの place が唯一の所有者であり続け、spawn からの読み取りによって所有者は増えない。
- `mut val` は、読み取りだけでも spawn から到達できない。
- nonsendable な `val` / `assoc val` は spawn から到達できない。

不変トップレベル/`assoc val` のストレージと、それが不変値として固定的に所有する backing はプロセスと同じ寿命を持つ **immortal allocation** です。immortal な所有辺には寿命管理の retain/release が不要で、spawn から読まれても atomic RC へ昇格しません。この寿命方針はsendabilityとは独立です。nonsendableな不変rootも、固定的に所有するbackingはimmortalにできます。`mut val`の現在値や、共有セル内の差し替えにより所有を失い得るallocationは、この固定backingとしてのretain/release省略の対象ではなく、通常実行中の再代入では旧値が通常どおりdropされます。ただし現在値もトップレベル所有根であるため、プロセス終了時に user `deinit` やフィールド破棄は保証しません。これは Swift/Rust と同じく、終了時専用の finalization 順序・閉店モード・`deinit` 依存グラフを言語仕様に持ち込まないためです。外部資源や OS には分からない意味的な片付けが必要な値は、トップレベル所有根ではなく `main` の通常スコープ、明示的な `close`/`shutdown`、または with-style のライフサイクル API に置きます。CoW 値をローカルへコピーしても immortal backing をそのまま共有でき、ローカル側を変更するときだけ通常どおり新しい backing を作ってから書き込みます。`MutableRef` の中へ後から格納される値など、可変な内部状態の allocation は immortal へ伝染せず通常どおり管理しますが、トップレベル所有根の共有参照が最後まで保持している cell と pointee は process exit で user `deinit` を保証しません。immortal であることを表す sentinel、header、静的 provenance などは観測できない内部方式です。

トップレベル初期化からworkerを起動する場合、workerとその呼び先がアクセスし得るglobalを、起動元の静的な初期化依存へ含めます。必要なglobalが初期化済みになってから起動元の本体を開始するため、workerは初期化子を実行したりrootへforceを要求したりせず、同じ初期化済みの値を読みます。依存循環はコンパイルエラーです（→ [トップレベルawaitと並行初期化](15-modules.md#トップレベル-await-と並行初期化)）。

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

### spawn fn — 引数で値を渡してスレッド起動

spawn fnは捕捉を持たず、呼び出し時にworkerを起動します。コピー可能なsendable引数は論理コピー、uniqueなsendable引数はmoveで渡します。borrow/inout引数は使えません。本体のreturn eはJoinHandleの結果Tを返し、宣言戻り型はJoinHandle[T]です。

```plew
spawn fn processFile(file: move File) -> JoinHandle[Report] {
    return analyze(file: move file)
}
val handle = processFile(file: move file)
```

入力と結果の双方にsendableを要求し、uniqueであること自体から送信可否を導きません。JoinHandle自体の別workerへのmoveにも同じ型規則を使います。

## sendable / nonsendable

型の構造的な導出、参照型、関数値・存在型の保証消去、FFIの責任は[値・所有権の送信規則](../01-basics/03-values.md#sendable--nonsendableスレッド間の移送可能性)を正本とします。直接の送信だけでなく、global経由の到達と暗黙のdeinit等も検査します。違反時は問題の型・フィールド経路、または禁止されたglobalまでの呼び出し経路を示します。

## チャネル

`@Std/Sync`が同期・通信の公開型を所有します。多送信者・一受信workerのMPSCです。Sender/BoundedSenderはコピー可能で、payload Tがsendableならハンドルもsendableです。Receiverもコピー可能ですが常にnonsendableで、作成したworker内に置きます。Receiverを共有して同じworkerから複数回受信でき、一つのメッセージを複数workerへ配信する機能ではありません。

```plew
fn channel[allowunique sendable T]() -> (sender: Sender[T], receiver: Receiver[T])
fn boundedChannel[allowunique sendable T](capacity: U64)
    -> (sender: BoundedSender[T], receiver: Receiver[T])
```

各呼び出しは新しいtransportと、現在のworkerに属するReceiverを作ります。端点の独立した公開factoryはありません。コピーは同じtransportの共同所有であり、キュー複製ではありません。端点pairは両方コピー可能な無名レコードです。仕事を受けるworker自身がReceiverを作り、連絡用Senderを経由して仕事用Senderを起動元へ返せます。joinはworker完了を待つため、起動通知の代わりにはしません。

### 送受信とclose

```plew
// Sender[T]
fn send(value~: move T) -> Result[(), SendError[T]]
// Receiver[T]
fn receive() -> Promise[Result[T, ChannelClosed]]
fn close()
```

型引数Tはcopyable/uniqueのsendable型を許します。コピー可能値は通常の値引数、uniqueな所有localは明示moveで渡します。端点自身は消費しません。

- 基礎sendは上限なしのバッファへ同期的に受理し、受信・処理完了は待ちません。成功後の受領／破棄責任はチャネルにあります。失敗時は未送信の値をSendError[T]が所有して返し、元のunique束縛を復活させません。SendErrorは`Closed(value: T)`を持つautounique enumで、消費matchで所有権を回復できます。
- receiveは呼び出し時に一回の予約を登録し、その結果Promiseを返します。空でもworkerを同期停止せず、完了は受信workerへ配送します。ChannelClosedはコピー可能な通常のエラー値です。値が受理された順と有効な予約の登録順をFIFOで扱い、異なるworkerの実時間上の呼び出し開始順やOSの公平性までは保証しません。
- closeは新しい送信受付を止めます。既に受理した値は失わず、残りの受信でdrainします。空かつ受付終了なら残る予約をClosedで終端させます。closeは処理中の値やユーザー処理の完了を待つbarrierではありません。
- 最後の送信側所有者を失った場合も、受理済み値をdrainしてからClosedになります。受信側が消滅した場合は以後のsendが失敗し、未受領の受理済み値を、その消滅を処理するworkerでキューから分離し、内部ロック解除後に一度だけ破棄します。公開Receiverのdropだけで、登録済み受信操作の内部所有まで消したことにはしません。
- 受理・割当て・切断・破棄の競合は一つの所有先へ確定します。ユーザーdeinitや完了continuationは内部ロックを解除し状態を確定してから実行し、再入に中間状態を見せません。

### 明示取消

```plew
// Receiver[T]
fn receiveCancellable() -> CancellableChannelOperation[Result[T, ReceiveError]]
// ChannelCancellation
fn cancel() -> Bool
```

CancellableChannelOperation[allowunique T]は、`Registered(result: Promise[T], cancellation: ChannelCancellation)`を持つ公開autounique enumです。uniqueな場合はenum全体の消費matchで結果Promiseと取消制御を別localに取り出します。uniqueなPromiseを無名レコードへ格納したり、structから部分moveしたりはしません。Registeredは未完了状態を保証せず、操作が既に完了していても返ります。

ReceiveErrorはClosed/Cancelledのコピー可能enumです。ChannelCancellationは、一回の操作の開始workerに所属するコピー可能かつnonsendableな制御で、公開factoryや結果受領権を持ちません。コピーは同じ制御権を共有します。uniqueなPromiseを消費awaitしている間も別に保持できます。制御の保持だけで終端済みの予約・結果payloadを延命しません。

cancelは未割当て操作をその呼び出しが取消へ変更した場合だけtrueを返します。割当て後、Closedによる終端後、既に取消済みならfalseです。取消が勝てば予約FIFOから外し、値を取得せずCancelledで一度だけ完了します。割当てが勝てば元の結果が値を所有し、再キューや再配送はしません。受理済みでもまだこの予約へ割当てていない値はキューまたは他の予約に残ります。遅延通知は予約identity・終端状態を照合し、再利用された登録へアクセスしません。

Promise・制御・wrapperのdropは暗黙取消ではありません。取消後も必要な完了配送／結果破棄が終わるまでpending責務を保持します。timeoutを扱う上位処理も取消の勝敗を確認し、既に割当て済みの結果は受領または一度だけ破棄する責任を維持します。一般のPromise取消は導入しません。

### 容量制限付き送信

```plew
// BoundedSender[T]
fn send(value~: move T) -> Promise[Result[(), SendError[T]]]
fn sendCancellable(value~: move T)
    -> CancellableChannelOperation[Result[(), CancellableSendError[T]]]
```

容量は構築時に固定し、受理済みでバッファ内に待つ値の数を制限します。未受理の待機送信・受信予約・既に配送した処理中の値の数までは制限しません。満杯なら非同期に待ち、スレッドを同期停止したり、値を捨てたり、Fullを返したりしません。容量0は待機受信への直接割当てでだけ送信受理が成立するrendezvousです。

未受理の操作がpayloadと結果配送の責任を所有し、受理時に一度だけキュー／受信操作へ移します。結果Promiseはsendを開始したworkerに所属します。close/受信側消滅が先なら未送信のTをClosedとして返します。未終端送信は送信側の内部所有も保持するので、最後の公開BoundedSenderを失っても待機送信を切断扱いしません。受理・取消・失敗の確定後はその内部所有を解放し、完了Promiseや残存取消制御だけで隠れた送信口を延命しません。

有効な待機送信は登録FIFOで受理し、新しい直接配送が古いバッファ値や先行送信を追い越すことはできません。容量0の送信受理と受信への割当ては同時に確定します。受信取消が先ならsendは未受理のまま、ペア成立が先なら双方の取消は不成立です。

CancellableSendError[T]は`Closed(value: T)` / `Cancelled(value: T)`を持つautounique enumです。sendCancellableの制御も開始workerに属し、受理前に取消が勝てば待機送信を外してpayloadを取消結果へ移します。受理済みなら、成功をawaitで観測する前でも取消はfalseです。非取消sendの結果をdropしても取消は起こりません。呼び出し側は待機操作を無制限に積まず、受理・受信・処理完了のどこを制限するか明示します。

## 並行安全性 ── 実質 race-free

安全なPlewのsendable値は、通常値なら値意味論、uniqueなら所有権移送、共有identityならFrozenRef/MutexRefの権限・同期を維持します。無同期な共有可変identityを持つRef/MutableRef、特定ループに属するPromise/Receiver、Guardや借用はworkerを越えません。atomic RCは共有本体の寿命を保護し、payloadの排他の代わりにはなりません。

共有本体は最後の所有者を失うworkerで一度だけ破棄し、暗黙deinit・外部資源解放もsendable検査の対象とします。workerからのglobalアクセスと間接呼び出しの制約も失わせません。FFIの外部メモリ・別名・同期は別の責任境界です。メモリ安全性は、デッドロックの不在、処理順の業務上の正しさ、async interleaveによる論理ハザードの不在を保証しません。
