# opencode-trade Context Mode README & 運用手順（PoC）

この文書は、`off / tools / shadow` のみを扱う `context-mode` PoC を
ローカルパッチとして運用する前提で、実装内容・検証・ロールバック・反映手順を
1 枚に集約したものです。上流 PR は作成・更新しません。

## 1. 目的

- `trade-memory` の権威は維持しつつ、`context-mode` を補助的な一時インデックスとして
  利用する。
- 大きなログや調査出力でコンテキストが圧迫される状況の検証を、最小差分で行う。
- 例外は上位へ伝播させず、既存ワークフローを壊さない fail-open 構成を維持する。

## 2. 方針（変更しない）

- 対応 mode は `off` / `tools` / `shadow` のみ。
- `on` / `strict` は未実装として扱い、`off` フォールバック。
- wrapper は top-level import で `context-mode` を直接読まず、delegate は起動時に遅延ロード。
- `tool.execute.after` 以外の hook は公開しない。
- `tool` は `ctx_` プレフィックス付きで、`execute` が関数のものだけを公開。
- 破損 delegate / 例外発生時も起動継続。
- rollback は
  1) `OPENCODE_TRADE_CONTEXT_MODE=off`
  2) 必要時 `OPENCODE_PURE=1`

## 3. 実装・テスト対象（現状）

- `.opencode/plugins/trade-context-mode.ts`
- `packages/opencode/test/plugin/trade-context-mode.test.ts`
- `packages/opencode/test/fixture/trade-context-mode-delegate-plugin.ts`
- `packages/opencode/test/fixture/trade-context-mode-delegate-invalid-tool-plugin.ts`
- `packages/opencode/test/fixture/trade-context-mode-delegate-nonfn-plugin.ts`

## 4. 環境変数

- `OPENCODE_TRADE_CONTEXT_MODE`
  - `off`（既定・最短復帰）
  - `tools`
  - `shadow`
  - `on` / `strict`（未実装で off 扱い）
- `OPENCODE_TRADE_CONTEXT_MODE_DELEGATE`
  - delegate plugin への path（相対 / 絶対）
- `OPENCODE_PURE=1`
  - `off` でも問題が残る場合の最終隔離

## 5. 運用チェック手順（最短）

### 5.1 オフ（既定）

```sh
export OPENCODE_TRADE_CONTEXT_MODE=off

cd packages/opencode
bun test test/plugin/trade-context-mode.test.ts
bun test test/plugin/loader-shared.test.ts test/plugin/trade-context-mode.test.ts
```

期待: `trade-context-mode.test.ts` と `loader-shared` の関連テストが pass。

### 5.2 tools

```sh
export OPENCODE_TRADE_CONTEXT_MODE=tools
export OPENCODE_TRADE_CONTEXT_MODE_DELEGATE=./test/fixture/trade-context-mode-delegate-plugin.ts

cd packages/opencode
bun test test/plugin/trade-context-mode.test.ts
```

期待:

- `ctx_*` 以外の tool は登録されない。
- `tool.execute.before` / `tool.execute.after` / `system.transform` / `experimental.*` は出ない。
- malformed な `ctx_` tool は除外される。

### 5.3 shadow

```sh
export OPENCODE_TRADE_CONTEXT_MODE=shadow
export OPENCODE_TRADE_CONTEXT_MODE_DELEGATE=./test/fixture/trade-context-mode-delegate-plugin.ts

cd packages/opencode
bun test test/plugin/trade-context-mode.test.ts
```

期待:

- `tool.execute.after` を fail-open で委譲。
- 委譲 hook が非関数でも起動継続。
- delegate import 失敗時も `{}` フォールバック＋警告のみ。

### 5.4 失敗時ロールバック

```sh
export OPENCODE_TRADE_CONTEXT_MODE=off
# それでも不具合が残る場合のみ
export OPENCODE_PURE=1
```

### 5.5 未実装 mode のガード確認

`on` / `strict` を指定しても delegate import なし・`{}` のみが返ることを確認。

## 6. レビュー向け 1 画面チェック

- off / 未設定 / 空白: `{}` のみ、delegate import なし
- tools: `ctx_` tool のみ公開、`ctx_` 以外は除外
- shadow: `tool.execute.after` だけ fail-open 委譲
- delegate 不在: 起動継続、fallback が成立
- `on` / `strict`: off と同等、delegate import なし
- 全テスト pass（`trade-context-mode.test.ts`, `loader-shared` 同時実行）

## 7. 変更反映（PR なし）

上流へ送らずローカルに適用する前提です。

```sh
git diff --binary origin/dev..HEAD > /tmp/context-mode-local.patch
git checkout -b <target-branch> origin/dev
git apply --verbose /tmp/context-mode-local.patch
```

履歴をそのまま反映する場合:

```sh
git cherry-pick 4dc4749f3 d5378bb6c
```

差分（最終）:

- `.opencode/plugins/trade-context-mode.ts`
- `docs/opencode-trade-context-mode-readme.md`
- `packages/opencode/test/plugin/trade-context-mode.test.ts`
- `packages/opencode/test/fixture/trade-context-mode-delegate-plugin.ts`
- `packages/opencode/test/fixture/trade-context-mode-delegate-invalid-tool-plugin.ts`
- `packages/opencode/test/fixture/trade-context-mode-delegate-nonfn-plugin.ts`

## 8. PR本文（必要時コピー）

```markdown
## Summary

- [x] `off / tools / shadow` のみ扱う `context-mode` wrapper PoC を追加。
- [x] `ctx_` tool のみ公開し、`tool.execute.after` を fail-open 委譲で追加。
- [x] `on` / `strict` 未実装時は off フォールバック。

## Changes

- Add: `.opencode/plugins/trade-context-mode.ts`
- Update: `packages/opencode/test/plugin/trade-context-mode.test.ts`
- Add: `packages/opencode/test/fixture/trade-context-mode-delegate-*.ts`

## Verification

- [x] `cd packages/opencode && bun test test/plugin/trade-context-mode.test.ts`
- [x] `cd packages/opencode && bun test test/plugin/loader-shared.test.ts test/plugin/trade-context-mode.test.ts`

## Risk & Rollback

- Rollback: `OPENCODE_TRADE_CONTEXT_MODE=off` → 必要なら `OPENCODE_PURE=1`
- context-mode 本体の本格導入は PoC 外（現時点）
```

## 9. 禁止事項（初期フェーズ）

- `opencode.jsonc` の `plugin: ["context-mode"]` 直書き
- `trade-memory` の主役化・置換
- `on` / `strict` の導入
- `tool.execute.before` の公開
- `ctx_` 以外 tool を tools mode で扱う前提
