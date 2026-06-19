---
name: monitor
description: snyk-cliの`snyk monitor`を実行し、プロジェクトの依存関係スナップショットをSnykダッシュボードへ登録する。継続的に新規脆弱性を監視したい場合に使用する。ユーザーが「snyk monitor」「継続監視」「Snykダッシュボードに登録」などを依頼したときに使う。
tools: Bash, Read, Grep, Glob
---

# snyk:monitor

`snyk monitor` でプロジェクトのスナップショットを Snyk へアップロードし、ダッシュボードで継続監視できるようにする。スキャン結果の取得が目的であれば代わりに `snyk:scan` を使う。

## 前提

- `snyk` コマンドがインストール済みで `snyk auth` 済みであること
- 未インストール / 未認証の場合は実行に進まずユーザーに案内して終了する（`snyk:scan` と同じ事前確認手順）

## 引数

| 引数 | 動作 |
|------|------|
| 引数なし | カレントディレクトリで `snyk monitor` を実行 |
| `iac [path]` | `snyk iac test --report [path]`（IaC 用の monitor 相当） |
| `container <image>` | `snyk container monitor <image>` |

## 実行手順

1. `snyk --version` と `snyk config get api` で事前確認
2. 対応するマニフェスト（`package.json` / `go.mod` 等）を検出
3. `snyk monitor` を実行し、出力を取得
4. 出力からスナップショットURL (`https://app.snyk.io/...`) を抽出してユーザーに伝える
5. エラー時は終了コードと stderr を併記して報告

## 注意

- `monitor` は結果をローカルに保存せずダッシュボード側に記録する。スキャン結果のMarkdownレポートが欲しい場合は `snyk:scan` を使う
- 組織が未設定の場合は `--org=<organization-id>` の指定をユーザーに促す
