# 文字列補間・書式化の実装とレビュー記録

## 現在の契約と実装状態

採用済みの言語契約は仕様本文へ反映済みです。この文書は実装・検証課題を保持し、仕様を重複定義しません。実装の進捗とは区別して、以下の契約と検証項目を完成条件とします。

- [Format・TextOutput・型付きOptions・String factory・補間構文と評価順](../spec/01-basics/02-basic-types.md#変数展開)
- [空enumとInfallible](../spec/02-type-system/05-structs-enums.md#空の列挙型)
- [空match](../spec/03-expressions/11-control-flow.md#網羅性rust-流)
- [Resultのok・err・value](../spec/03-expressions/13-error-handling.md#resultの値の取り出し)
- [factoryと型省略構築](../spec/02-type-system/05-structs-enums.md#factory)

## 実装前の確認と対象外

String factoryは `pub impl[T] String where T: Format` 内に置き、`factory(value: T, options: T.Options = </>)` と宣言します。既存のgeneric impl・関連型・デフォルト式・trait factory要件を組み合わせた実装対応を検証します。

unique対応の拡張と、補間を一行で直接出力する簡潔なAPIは今回の対象外です。既存の型制限を維持し、直接出力にはwriteと子要素のformatを使います。

## 第三者レビュー：設計判断と実装確認事項

仕様整合性、型・所有権・エラー処理、性能・実装の3観点を独立にレビューし、親レビューで照合した。静的な仕様・ソース確認であり、コンパイル実験・性能測定は未実施。単一Formatの設計を覆す矛盾は確認していないが、以下を解消してから実装計画を確定する。レビュー後の合意は仕様本文へ反映し、以下からその契約を参照する。

### 仕様反映で解消した記述不整合

1. **公開境界**：本文の `FormatOptions` 宣言例は非pubだが、pub Formatの関連型制約に現れるため `pub trait FormatOptions` へ修正済み。[公開API閉包性](../spec/04-execution/15-modules.md#公開-api-閉包性)から導ける修正。
2. **String factoryのgeneric構文**：`factory[T] ... where` は既存仕様・parserで確認できない。既存の [impl型パラメータ](../spec/02-type-system/06-generics.md#impl-の型パラメータ)を使い、`pub impl[T] String where T: Format { factory(value: T, options: T.Options = </>) { ... } }` と表せる。仕様本文はこのimpl記法へ修正済み。新たなfactory固有generic構文は不要。この組合せの実装対応は別途検証する。

### レビューを受けて確定した設計判断

3. **receiverと値の保持**：補間も通常の同期読み借用receiver規則に統一する。snapshotや未定義動作を追加しない。詳細はspec/02の「補間の評価順と寿命」。
4. **uniqueの対象範囲**：今回は既存のgeneric制限に従い、対応拡張は別課題とする。詳細はspec/02の「Formatと出力先の契約」。共有wrapperを同じ意味・コストの代替と仮定しない。
5. **失敗後の処理**：出力先の契約の範囲でFormat実装者が決める。即時伝播や後続write禁止を一律に要求しない。詳細はspec/02の「writeの共通契約と実装側の責務」。

### 既決方針から導ける実装条件

6. **trait identity**：補間は選択したFormat準拠のwitnessと同じ準拠のOptionsを使う。ベア名 `.format` を再探索してinherentな同名メソッドへ解決してはいけない。[メソッド解決](../spec/02-type-system/07-methods-impl.md#メソッドのオーバーロード)のinherent優先と区別する。
7. **寿命と早期終了**：暗黙Options構築は[デフォルト引数規則](../spec/01-basics/04-functions.md#デフォルト引数)に従う独立した完全式。内部だけの一時値と結果値の寿命を区別する。tryの早期returnでは未完成builderも通常の破棄対象。panicは既存のabort規則に従い、巻き戻し破棄を新たに保証しない。
8. **空enum・空match**：Infallibleという名前の特例でなく、一般の空enumを扱う。現行の `compiler/src/Frontend.pw` の `matchHasEnumOrStruct`、`Codegen/Check/Return.pw` の0arm非発散判定、`Mid/Build.pw` の0arm拒否を含め、型の空性から到達不能CFGまで通す必要がある。空でない型の空matchは拒否する。`match createEmpty() {}`でもscrutineeの副作用・発散を消してはならない。空enum導入と特殊なABI最適化は別の作業。
9. **構築バッファ**：共通出力先を使うだけでは線形時間を保証しない。各writeがString連結なら二乗時間になり得る。[既存のbuilder方針](../spec/01-basics/02-basic-types.md#構築ビルダ)に従い、一つの伸長バッファへ追記し、prefixの反復コピー、既知UTF-8の不要な再検査、finish時の全量再コピーを避ける。初期は内部型でもよく、公開builder APIの追加は必須ではない。
10. **動的文字列と確保**：現行のリテラルは静的rawbufで無確保だが、整数は動的配列、浮動小数点は一時バッファから所有Stringへの確保・コピーを使う。任意のTextOutputは受け取ったStringを保持してよいため、動的なスタック領域を無条件にStringとして渡すことはできない。非escape証明やインライン格納などが必要。writeScalarを再導入する判断ではなく、数値の確保・コピーの検証条件として残す。
11. **特殊化とResultコスト**：型×出力先の特殊化はインライン化に有利だが、コード量・コンパイル時間・JIT初回費用を増やし得る。出力先に依存しない大きな処理の共有を検討する。`Result[(), Infallible]`の不要なtag・分岐・copy/dropが消えるかも確認する。特定ABIを意味論として保証しない。
12. **将来の存在型**：既存のany呼び出し規則はgenericメソッドを一律禁止していない。`any Format[Options=...]`でgenericな出力先を扱うwitnessは、将来の存在型実装との接点。初期実装の停止理由とはしないが、常に静的dispatchできるとは説明しない。

### 検証へ落とす項目

- 公開FormatOptions・trait factory要件・関連型射影・generic impl上のfactory・省略Optionsを別モジュールから利用する。
- 同名inherent formatがある型でもFormat準拠を選択する。
- receiverをOptions生成から変更するケース、関数結果・添字結果・共有セルのケースを固定する。
- Optionsの暗黙/明示評価、内部一時値の破棄、tryで後続補間を実行しないことを確認する。
- ユーザー定義空enumの空matchを受理し、Bool・Unit等の空matchを拒否する。副作用を持つscrutineeを消さない。
- Resultのok/err/valueを既存generic規則で検査する。名前の既存衝突は確認されていない。
- 大量断片の構築、整数・浮動小数点・既存String、2種類以上の出力先で、確保・コピー・コード量・残留失敗分岐を確認する。
- 標準型ごとのOptionsとデフォルト表示、Stringリテラルの波括弧移行、macro生成ソースのエスケープ、run/check/buildの同一意味を実装計画に含める。数値リテラルの既定型を補間だけで新設しない。
