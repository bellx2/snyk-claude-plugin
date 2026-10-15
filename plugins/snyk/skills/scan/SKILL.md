---
name: scan
description: snyk-cliで脆弱性スキャンを実行し、英語の結果を日本語に翻訳した上で各脆弱性をClaudeが調査・解説したMarkdown/HTMLレポートを作成する。ユーザーが「snyk」「脆弱性スキャン」「依存ライブラリの脆弱性チェック」「コンテナの脆弱性」「IaCの設定ミスチェック」などを依頼したときに使用する。
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, WebSearch, WebFetch
---

# snyk:scan

snyk-cli を使って脆弱性スキャンを実行し、英語出力を日本語化した Markdown / HTML レポートを作成する。検出された脆弱性については Claude が追加調査して背景・影響・推奨対応をまとめる。

## 前提

- `snyk` コマンドがインストールされ、`snyk auth` 済みであること
- 未インストール / 未認証の場合は skill 側では自動実行せず、ユーザーへ手順を案内して終了する（後述）
- 出力レポートは日本語

## 引数

ユーザーが skill 呼び出し時に渡す引数で挙動を切り替える。

| 引数 | 動作 |
|------|------|
| `all` または引数なし | 適用可能なスキャンをすべて実行（後述の自動判定ルールに従う） |
| `oss` | `snyk test` のみ |
| `code` | `snyk code test` のみ |
| `container <image[:tag]>` | `snyk container test <image>` |
| `iac [path]` | `snyk iac test [path]`（path 省略時はカレントディレクトリ） |

複数指定（例: `oss code`）も受け付ける。スペース区切り。

## 自動判定ルール（引数なし or `all`）

カレントディレクトリを調べて、適用可能なスキャンだけを実行する：

1. **OSS スキャン**: `package.json` / `go.mod` / `pyproject.toml` / `requirements.txt` / `Gemfile` / `pom.xml` / `Cargo.toml` などのマニフェストがあれば実行
2. **Code スキャン**: ソースコードファイル（`.go` `.ts` `.tsx` `.js` `.py` `.java` `.rb` `.rs` 等）が見つかれば実行
3. **Container スキャン**: 引数なし時はスキップ（イメージ指定が必須なため）。`Dockerfile` がある場合はユーザーに「どのイメージをスキャンするか」を確認
4. **IaC スキャン**: `*.tf` / `*.yaml`(k8s manifests) / `Dockerfile` / `docker-compose.yml` / CloudFormation テンプレートが見つかれば実行

該当しないスキャンはスキップし、レポート冒頭に「対象外」と明記する。

## 実行手順

### 1. 事前確認

#### 1-1. インストール確認

```bash
snyk --version
```

未インストール（コマンド not found）の場合は、レポート作成に進まず [references/auth.md](references/auth.md) の「未インストールのとき」のメッセージを出して終了する。

#### 1-2. 認証確認

```bash
snyk whoami
```

- 終了コード 0 でユーザー名/ID が出力されれば認証済み → 次ステップへ
- 終了コード 非0、または `AUTH_FAILED` / `not authenticated` / `snyk auth` を含むメッセージが返れば未認証 → 下記「自動認証フロー」を実行

旧 CLI（API トークン方式）も併用する場合は `snyk config get api` が空でないことで判定可能だが、OAuth 認証では `api` キーは空のままになるため `snyk whoami` を主たる判定に使う。未認証のままスキャンを実行すると `AUTH_FAILED` 等で全件失敗するため、必ずスキャン前に解消する。

#### 1-3. 自動認証フロー（未認証時）

[references/auth.md](references/auth.md) の「自動認証フロー」に従う。ブラウザを開くため、`snyk auth` の実行前に必ずユーザーの了承を得る。拒否された場合は `snyk auth` の手動実行を案内して終了する。

### 2. スキャン実行

各スキャンを **JSON形式** で取得すると後段の解析が安定するため、人間向けの出力とJSON出力を両方取得する：

```bash
# OSS
snyk test --json > /tmp/snyk-oss.json 2>/dev/null; snyk test 2>&1 | tee /tmp/snyk-oss.txt

# Code
snyk code test --json > /tmp/snyk-code.json 2>/dev/null; snyk code test 2>&1 | tee /tmp/snyk-code.txt

# Container
snyk container test <image> --json > /tmp/snyk-container.json 2>/dev/null; snyk container test <image> 2>&1 | tee /tmp/snyk-container.txt

# IaC
snyk iac test <path> --json > /tmp/snyk-iac.json 2>/dev/null; snyk iac test <path> 2>&1 | tee /tmp/snyk-iac.txt
```

注意:
- snyk は脆弱性が見つかると終了コード 1 を返す。エラーではないので継続する
- 終了コード 2 以上は実行エラー。stderr をレポートに含めて原因を示す
- JSON 出力が空/壊れている場合はテキスト出力をパースする

### 3. 結果の解析と日本語化

各JSON（またはテキスト）から以下を抽出：

- **脆弱性ID** (CVE / SNYK-XXX / ルールID)
- **重大度** (critical / high / medium / low) → 日本語化（緊急/高/中/低）
- **対象** (パッケージ名+バージョン / ファイル+行番号 / イメージレイヤー / リソース)
- **説明** (英語) → 日本語訳
- **修正方法** (アップグレード先・パッチ・設定変更) → 日本語訳

### 4. Claudeによる追加調査

各脆弱性ごとに、以下の観点で簡潔な追加情報を付与する：

- **何が問題か**: 攻撃シナリオを 1-2 文で
- **このプロジェクトでの影響**: 該当コード/設定の使われ方を `Grep` 等で確認した上で「実際に到達可能か」「悪用条件」を評価
- **推奨対応**: 単純なバージョンアップで済むか、コード修正が必要か、回避策があるか

CVE 等で情報が不足する場合は `WebSearch` / `WebFetch` で公式アドバイザリを参照する（数が多い場合は緊急/高のみ深掘り、中/低は要約に留める）。

### 5. レポート出力

Markdown と HTML の **2 ファイル** を同じ内容で出力する：

| 形式 | パス |
|------|------|
| Markdown | `./reports/snyk/YYYYMMDD.md` |
| HTML | `./reports/snyk/YYYYMMDD.html` |

日付はシェルの `date +%Y%m%d` から取得する。同日に複数回実行された場合は上書きする（上書き前にユーザーへ確認）。

保存前に `mkdir -p ./reports/snyk` でディレクトリを作成する。

レポートは「上長・関係者への報告書」としてそのまま提出できる体裁にする。主観的な感想は避け、事実・評価・推奨アクションを分けて記載する。HTML はブラウザで開くだけで読める **単一ファイル**（外部 CSS/JS なし）とする。

- Markdown 版のフォーマット: [references/report-markdown.md](references/report-markdown.md)
- HTML 版のデザイン・骨格・注意点: [references/report-html.md](references/report-html.md)

Markdown を先に完成させ、その内容を HTML に反映する。

### 6. 最終出力

Markdown / HTML 両方のファイルパスをユーザーに伝える。チャットには「総合判定」と「検出サマリー表」のみを表示する（全文はファイルに任せる）。HTML はブラウザで開いて確認できる旨も一言添える。緊急 / 高 が 1 件でもあれば対応が必要である旨を明確に伝える。

## 注意

- スキャンには数十秒〜数分かかるので、必要に応じて `Bash` の timeout を 300000 ms 程度に設定
- 大規模プロジェクトで脆弱性が 50 件超になる場合は、中/低を集約表示（テーブルで列挙）し、緊急/高のみを詳細展開
- snyk のトークン期限切れエラーが出たら、レポートを作らずユーザーに `snyk auth` を促して終了
- JSON 出力が `--json` 非対応プランで失敗する場合は、テキスト出力からのパースにフォールバック
