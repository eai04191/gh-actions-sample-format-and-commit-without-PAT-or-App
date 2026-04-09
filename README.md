# gh-actions-sample-format-and-commit-without-PAT-or-App

PATやGitHub Appを使わずに、GitHub ActionsからのコミットでCIチェックを回すサンプル。

## 背景

- `GITHUB_TOKEN` によるpushは `on: push` / `on: pull_request` をトリガーしない（無限ループ防止の仕様）
- 通常はPATやGitHub Appトークンで回避するが、組織だとAppの上限（50個）があり、リポごとにAppを分けたい場合に制約になる
- `workflow_dispatch` は `GITHUB_TOKEN` から呼んでも発火できることを利用して、外部リソースなしで解決する

## 仕組み

3つのワークフローで構成される。

### 1. `check-format-caller.yml`（トリガー: `on: pull_request`）

PRイベントを受けて、`gh workflow run` で `check-format.yml` を `workflow_dispatch` 経由で呼び出す。PRイベントからdispatchイベントへの変換役。

### 2. `check-format.yml`（トリガー: `workflow_dispatch`）

Prettier によるフォーマットチェックを実行する。`workflow_dispatch` ではPRのChecksに自動で紐付かないため、[myrotvorets/set-commit-status-action](https://github.com/myrotvorets/set-commit-status-action) を使ってCommit Status APIで対象shaにステータスを直接書き込む。

### 3. `fix-format.yml`（トリガー: `workflow_dispatch`、手動実行）

Prettierでフォーマットを修正し、`git-auto-commit-action` でコミット。変更があれば、新しいコミットに対して `check-format.yml` をdispatchしてステータスチェックも付ける。

## フロー

```
PR作成/更新 → check-format-caller → (dispatch) → check-format → Commit Statusに結果書き込み
手動で fix-format 実行 → フォーマット修正コミット → (dispatch) → check-format → Commit Statusに結果書き込み
```

## ポイント

- `GITHUB_TOKEN` のみで完結。PAT・GitHub Appの管理不要
- `workflow_dispatch` が `GITHUB_TOKEN` でも発火可能なことを利用
- Commit Status API（Checks APIではなく）で手動でステータスを設定
