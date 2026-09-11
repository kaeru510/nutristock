# NutriStock 開発メモ

Artifact: https://claude.ai/code/artifact/c3e95274-3a59-4b08-98cd-92ffbac7a7ad
ソース: `nutristock.html`（このフォルダ内。編集後は同じURLに再公開する）

## できること / できないこと

| 機能 | 状態 | 備考 |
|---|---|---|
| パントリー管理（賞味期限順・バッジ表示） | ✅ 動作 | |
| 栄養ダッシュボード（自動目標計算・メーター） | ✅ 動作 | Mifflin-St Jeor式ベースの概算 |
| 期限×不足栄養素の献立提案 | ✅ 動作 | タンパク源＋野菜の組み合わせ提案ロジック |
| 食事記録（手入力） | ✅ 動作 | |
| 歩数シミュレータ（プリセット/手入力） | ✅ 動作 | 実機ヘルスケア連携は不可（下記） |
| バーコードスキャン（JANコード読み取り） | ⚠️ デスクトップブラウザのみ | スマホ（Artifact経由）はカメラ権限がブロックされ不可（下記） |
| バーコード→商品名の自動学習 | ✅ 動作 | 商品DBには接続不可なので、一度手入力すれば次回から自動入力される方式 |
| AI写真解析（食事記録） | ❌ 現状不可 | 実装済みだがアカウント側の制限で保留中（下記） |
| Apple ヘルスケア連携 | ❌ 不可能 | 原理的にWeb Artifactからは到達不可 |

## プラットフォーム側の制約（コードでは解決不可）

### 1. Apple ヘルスケア連携
HealthKitはネイティブiOSアプリ専用APIで、Webページから使う公式手段が存在しない。歩数の自動取得は不可能（ネイティブアプリ化以外に道がない）。

### 2. AI写真解析（sample capability）が使えない
`claude.use("sample")` が常にnullを返す。VS Code拡張・デスクトップブラウザ・モバイルブラウザすべてで再現。
- claude.ai設定 > 機能 > 「AI搭載のアーティファクト」はON済み（原因ではなかった）
- 2026-09-11にAnthropicへフィードバック送信済み、回答待ち
- アカウント/組織側の機能有効化状況に起因すると思われる（原因確定はできず）
- 直った場合、コード変更なしにそのまま動くはず

### 3. スマホでバーコードのカメラが起動しない
エラー: `NotAllowedError` / `Permissions policy violation: camera is not allowed in this document`
- Artifactはclaude.aiのページ内にiframeとして埋め込まれる仕組み
- iframe内でカメラ(getUserMedia)を使うには、親ページ側が `allow="camera"` を明示的に渡す必要がある
- デスクトップブラウザでは動く／モバイルでは（このiframeの許可設定により）ブロックされる
- ページ側のコードでは解決不可。Anthropicへの報告事項

## 実装上の技術メモ（ハマったポイント）

### db snapshotのデータは凍結（frozen）されている
`dbDoc.onSnapshot` で届く `snap.data()` は読み取り専用。`Object.assign(defaultState(), data)` のような浅いコピーでは、ネストしたオブジェクト（`steps`, `pantry` 等）が凍結されたまま使い回されてしまい、後から `state.steps[today] = x` のような書き込みをした瞬間に `TypeError: object is not extensible` で例外が発生する。
→ 必ず `JSON.parse(JSON.stringify(data))` 等でディープクローンしてから使うこと。

### 複数タブ/複数端末での上書き対策
`dbDoc.set(state)` で毎回全データを丸ごと上書きすると、別タブの古いローカルデータが最新の変更を消してしまうことがある。
→ 変更した項目だけ `dbDoc.update({変更したキー: 値})` で部分更新する（`persist(fields)` のように変更フィールドを明示する）。

### db接続前のクリックが握りつぶされる競合
ページ表示直後、`claude.use("db")` の接続が完了する前にユーザーが操作すると、その変更がローカルにしか反映されず、直後に届く「まだ書き込まれていない古いサーバーデータ」で上書きされてしまうことがある。
→ 接続未完了時の変更は `hasUnsyncedLocalChange` フラグで検知し、初回スナップショット受信時にサーバーデータで上書きせず、ローカルの変更を優先してpushする。

## Artifact runtime capability の一般的な制約（今後の開発で再度当たりうる）
- `db` capability を宣言すると、そのArtifactは**組織内限定にしかできない**（一般公開不可）
- 直接開いたローカルファイル（`file://`）では `window.claude` が一切存在せず、db・sample等すべて使えない（localStorageのみのフォールバック動作になる）
- capabilityの利用可否はビューア（VS Code拡張／ブラウザ／モバイル）によって異なりうる
