---
name: scan
description: snyk-cliで脆弱性スキャンを実行し、英語の結果を日本語に翻訳した上で各脆弱性をClaudeが調査・解説したMarkdown/HTMLレポートを作成する。ユーザーが「snyk」「脆弱性スキャン」「依存ライブラリの脆弱性チェック」「コンテナの脆弱性」「IaCの設定ミスチェック」などを依頼したときに使用する。
tools: Bash, Read, Write, Edit, Grep, Glob, WebSearch, WebFetch
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

未インストール（コマンド not found）の場合は、レポート作成に進まず以下のメッセージを出して終了する：

```
snyk CLI がインストールされていません。以下のいずれかでインストールしてください。

  macOS (Homebrew):
    brew tap snyk/tap
    brew install snyk

  npm (クロスプラットフォーム):
    npm install -g snyk

  バイナリ直接ダウンロード:
    https://github.com/snyk/cli/releases

インストール後、`snyk auth` で認証してから再度依頼してください。
```

#### 1-2. 認証確認

```bash
snyk whoami
```

- 終了コード 0 でユーザー名/ID が出力されれば認証済み → 次ステップへ
- 終了コード 非0、または `AUTH_FAILED` / `not authenticated` / `snyk auth` を含むメッセージが返れば未認証 → 下記「自動認証フロー」を実行

旧 CLI（API トークン方式）も併用する場合は `snyk config get api` が空でないことで判定可能だが、OAuth 認証では `api` キーは空のままになるため `snyk whoami` を主たる判定に使う。未認証のままスキャンを実行すると `AUTH_FAILED` 等で全件失敗するため、必ずスキャン前に解消する。

#### 1-3. 自動認証フロー（未認証時）

`snyk auth` は実行するとブラウザを自動で開き、認証完了まで標準出力でプロンプトを表示しつつ待機する。Claude 側で URL を抽出・別途オープンする必要は通常ないため、以下の最小手順を取る。

1. ユーザーに「`snyk auth` を実行してブラウザで認証を進めて良いか」確認する（ブラウザを開く副作用があるため）
2. 了承を得たら `snyk auth` を **バックグラウンド実行**（`Bash` ツールの `run_in_background: true`、ログを `/tmp/snyk-auth.log` 等にリダイレクト）
3. 念のため標準出力からURLを抽出してチャットに提示（自動オープンが失敗した場合の手動コピー用）
   - URLパターン: `https://app\.snyk\.io/oauth2/authorize\?[^ ]+` （新OAuth）または `https://snyk\.io/login\?token=[^ ]+`（旧トークン）
   - 出力遅延に備え 10 秒程度ポーリング
4. ユーザーに「ブラウザでログインしてください。完了を待ちます。」と伝える
5. バックグラウンドプロセスを最大 5 分待機する。`snyk auth` は認証完了で正常終了するため、タイムアウト時はプロセスを終了させ、手動実行を案内
6. 完了後、再度 `snyk whoami` を実行して終了コード 0 を確認してからスキャンに進む

実装例：

```bash
# バックグラウンド実行（run_in_background: true）
snyk auth 2>&1 | tee /tmp/snyk-auth.log
```

URL抽出（フォールバック用）：

```bash
for i in 1 2 3 4 5 6 7 8 9 10; do
  URL=$(grep -oE 'https://(app\.snyk\.io/oauth2/authorize|snyk\.io/login)\?[^ ]+' /tmp/snyk-auth.log | head -1)
  [ -n "$URL" ] && break
  sleep 1
done
[ -n "$URL" ] && echo "ブラウザが自動で開かない場合は次のURLを開いてください: $URL"
```

ユーザーが認証実行を拒否した場合は、`snyk auth` の手動実行を案内して終了する。

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

レポートのフォーマット：

```markdown
# Snyk 脆弱性スキャン報告書

## 1. 基本情報

| 項目 | 内容 |
|------|------|
| 報告日 | YYYY-MM-DD |
| 実行日時 | YYYY-MM-DD HH:MM:SS |
| 対象プロジェクト | プロジェクト名（ディレクトリ: /path/to/project） |
| 実行スキャン種別 | OSS / Code / IaC（Container は対象外） |
| Snyk CLI バージョン | x.y.z |

## 2. 全体所見

（3〜5 行で「全体の検出状況」「最も影響の大きいリスク」「推奨される次アクション」を非エンジニアにも伝わる言葉で記述）

**総合判定**: 要緊急対応 / 要対応 / 経過観察 / 問題なし

判定基準:
- 要緊急対応: 緊急(critical)が 1 件以上、または到達可能な高(high)が存在
- 要対応: 高(high)が 1 件以上
- 経過観察: 中(medium)以下のみ
- 問題なし: 検出ゼロ

## 3. 検出サマリー

| スキャン種別 | 緊急 | 高 | 中 | 低 | 合計 |
|------------|----|----|----|----|----|
| OSS        | 0  | 2  | 5  | 1  | 8  |
| Code       | -  | 1  | 0  | 2  | 3  |
| Container  | 対象外 | | | | |
| IaC        | 0  | 0  | 1  | 0  | 1  |
| **合計**   | **0** | **3** | **6** | **3** | **12** |

## 4. リスク評価

緊急 / 高 の各脆弱性についてビジネス影響を 1〜2 行で評価。

| # | 重大度 | 対象 | 概要 | 想定影響 | 到達可能性 |
|---|------|------|------|--------|----------|
| 1 | 高 | lodash@4.17.20 | プロトタイプ汚染 | 任意コード実行の可能性 | 到達可能 |
| 2 | 高 | src/api/login.ts:42 | SQLインジェクション | 認証情報漏洩 | 到達可能 |

## 5. 推奨アクション

### 5.1 即時対応（緊急 / 高）

| # | 内容 | 対象 | 想定工数 |
|---|------|------|--------|
| 1 | lodash を 4.17.21 以上にアップグレード | package.json | 小（< 1h） |
| 2 | パラメータバインディングへの修正 | src/api/login.ts | 中（半日） |

### 5.2 計画対応（中）

| # | 内容 | 対象 | 想定工数 |
|---|------|------|--------|
| 1 | ... | ... | ... |

### 5.3 経過観察（低）

- （脆弱性ID と対象を 1 行ずつ列挙、対応はバージョン更新サイクルに合わせる）

## 6. 詳細

### 6.1 OSS 脆弱性

#### [緊急] パッケージ名@バージョン — 脆弱性タイトル（日本語訳）

- **ID**: SNYK-XXX / CVE-YYYY-NNNNN
- **概要**: （英語説明を日本語訳した本文）
- **修正方法**: バージョン X.Y.Z 以上にアップグレード
- **Claudeによる追加調査**:
  - 何が問題か: ...
  - 本プロジェクトでの影響: `src/foo.ts` でこのAPIを使用しており、外部入力が到達するため悪用される可能性あり
  - 推奨対応: ...

（以下、重大度の高い順に列挙）

### 6.2 Code 脆弱性

#### [高] ファイル:行 — 脆弱性タイトル

（同じ構造）

### 6.3 IaC 脆弱性

#### [中] リソース名 — タイトル

（同じ構造）

## 7. 次回スキャン推奨

- 推奨実施日: YYYY-MM-DD（前回から N 日後、または依存更新後）
- 推奨頻度: 週次 / 月次 / リリース前（プロジェクト規模に応じて記載）

## 8. 付録: スキャン実行ログ

- OSS: 終了コード 1, 8 件検出（実行時間 12s）
- Code: 終了コード 1, 3 件検出（実行時間 45s）
- IaC: 終了コード 1, 1 件検出（実行時間 5s）
```

#### HTML 版

Markdown と **同じ章立て・同じ内容** を HTML に変換して `./reports/snyk/YYYYMMDD.html` に保存する。デザイン方針：

- シンプル・読みやすさ優先（装飾過多にしない）
- 外部リソース不要（`<style>` を `<head>` 内に埋め込む）
- システムフォント、白背景、適度な余白
- 総合判定は目立つバッジで表示（色は下表）
- 重大度は行頭またはバッジで色分け
- テーブルは横スクロール可能（`overflow-x: auto`）
- 印刷時も崩れにくい（`@media print` で余白調整）

重大度・判定の色：

| 区分 | 背景色 | 文字色 |
|------|--------|--------|
| 要緊急対応 / 緊急 | `#fef2f2` | `#991b1b` |
| 要対応 / 高 | `#fff7ed` | `#9a3412` |
| 経過観察 / 中 | `#fefce8` | `#854d0e` |
| 問題なし / 低 | `#f0fdf4` | `#166534` |

HTML 骨格（中身はスキャン結果で埋める）：

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Snyk 脆弱性スキャン報告書 — YYYY-MM-DD</title>
  <style>
    :root {
      --text: #1a1a1a;
      --muted: #6b7280;
      --border: #e5e7eb;
      --bg: #f9fafb;
      --card: #ffffff;
    }
    * { box-sizing: border-box; }
    body {
      margin: 0;
      padding: 2rem 1rem;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "Hiragino Sans", "Noto Sans JP", sans-serif;
      font-size: 15px;
      line-height: 1.6;
      color: var(--text);
      background: var(--bg);
    }
    .container { max-width: 960px; margin: 0 auto; }
    header {
      background: var(--card);
      border: 1px solid var(--border);
      border-radius: 8px;
      padding: 1.5rem 2rem;
      margin-bottom: 1.5rem;
    }
    header h1 { margin: 0 0 0.25rem; font-size: 1.5rem; }
    header .meta { color: var(--muted); font-size: 0.875rem; }
    section {
      background: var(--card);
      border: 1px solid var(--border);
      border-radius: 8px;
      padding: 1.25rem 1.5rem;
      margin-bottom: 1rem;
    }
    section h2 {
      margin: 0 0 1rem;
      font-size: 1.125rem;
      border-bottom: 1px solid var(--border);
      padding-bottom: 0.5rem;
    }
    section h3 { margin: 1.25rem 0 0.75rem; font-size: 1rem; }
    .verdict {
      display: inline-block;
      padding: 0.35rem 0.75rem;
      border-radius: 6px;
      font-weight: 600;
      font-size: 0.9375rem;
    }
    .summary-text { margin: 0.75rem 0; }
    .summary-text p { margin: 0.5rem 0; }
    table {
      width: 100%;
      border-collapse: collapse;
      font-size: 0.875rem;
    }
    th, td {
      border: 1px solid var(--border);
      padding: 0.5rem 0.75rem;
      text-align: left;
    }
    th { background: var(--bg); font-weight: 600; }
    .table-wrap { overflow-x: auto; margin: 0.75rem 0; }
    .badge {
      display: inline-block;
      padding: 0.15rem 0.5rem;
      border-radius: 4px;
      font-size: 0.8125rem;
      font-weight: 600;
    }
    .badge-critical, .verdict-critical { background: #fef2f2; color: #991b1b; }
    .badge-high, .verdict-high { background: #fff7ed; color: #9a3412; }
    .badge-medium, .verdict-medium { background: #fefce8; color: #854d0e; }
    .badge-low, .verdict-low { background: #f0fdf4; color: #166534; }
    .badge-none, .verdict-none { background: #f3f4f6; color: #374151; }
    .vuln {
      border-left: 4px solid var(--border);
      padding: 0.75rem 1rem;
      margin: 0.75rem 0;
      background: var(--bg);
      border-radius: 0 6px 6px 0;
    }
    .vuln-critical { border-left-color: #991b1b; }
    .vuln-high { border-left-color: #9a3412; }
    .vuln-medium { border-left-color: #854d0e; }
    .vuln-low { border-left-color: #166534; }
    .vuln h4 { margin: 0 0 0.5rem; font-size: 0.9375rem; }
    .vuln ul { margin: 0.25rem 0; padding-left: 1.25rem; }
    .vuln li { margin: 0.25rem 0; }
    dl { margin: 0; }
    dt { font-weight: 600; margin-top: 0.5rem; }
    dd { margin: 0.15rem 0 0 0; color: var(--muted); }
    footer {
      text-align: center;
      color: var(--muted);
      font-size: 0.8125rem;
      margin-top: 2rem;
    }
    @media print {
      body { background: white; padding: 0; }
      section, header { break-inside: avoid; box-shadow: none; }
    }
  </style>
</head>
<body>
  <div class="container">
    <header>
      <h1>Snyk 脆弱性スキャン報告書</h1>
      <p class="meta">報告日: YYYY-MM-DD ｜ 対象: プロジェクト名</p>
    </header>

    <section id="overview">
      <h2>1. 基本情報</h2>
      <div class="table-wrap">
        <table><!-- 基本情報テーブル --></table>
      </div>
    </section>

    <section id="executive">
      <h2>2. 全体所見</h2>
      <div class="summary-text"><!-- 3〜5 行の要約 --></div>
      <p><span class="verdict verdict-high">総合判定: 要対応</span></p>
      <!-- 判定基準は <details><summary>判定基準</summary>...</details> で折りたたみ可 -->
    </section>

    <section id="summary">
      <h2>3. 検出サマリー</h2>
      <div class="table-wrap">
        <table><!-- 検出サマリーテーブル --></table>
      </div>
    </section>

    <section id="risk">
      <h2>4. リスク評価</h2>
      <div class="table-wrap">
        <table><!-- 緊急/高のみ --></table>
      </div>
    </section>

    <section id="actions">
      <h2>5. 推奨アクション</h2>
      <h3>5.1 即時対応（緊急 / 高）</h3>
      <!-- テーブル -->
      <h3>5.2 計画対応（中）</h3>
      <!-- テーブル -->
      <h3>5.3 経過観察（低）</h3>
      <!-- 箇条書き -->
    </section>

    <section id="details">
      <h2>6. 詳細</h2>
      <h3>6.1 OSS 脆弱性</h3>
      <div class="vuln vuln-high">
        <h4><span class="badge badge-high">高</span> パッケージ名@バージョン — タイトル</h4>
        <dl>
          <dt>ID</dt><dd>SNYK-XXX / CVE-YYYY-NNNNN</dd>
          <dt>概要</dt><dd>...</dd>
          <dt>修正方法</dt><dd>...</dd>
          <dt>Claude による追加調査</dt>
          <dd>
            <ul>
              <li><strong>何が問題か:</strong> ...</li>
              <li><strong>本プロジェクトでの影響:</strong> ...</li>
              <li><strong>推奨対応:</strong> ...</li>
            </ul>
          </dd>
        </dl>
      </div>
      <!-- 6.2 Code / 6.3 IaC も同構造 -->
    </section>

    <section id="next">
      <h2>7. 次回スキャン推奨</h2>
      <ul><!-- 推奨日・頻度 --></ul>
    </section>

    <section id="appendix">
      <h2>8. 付録: スキャン実行ログ</h2>
      <ul><!-- 終了コード・件数・実行時間 --></ul>
    </section>

    <footer>Generated by snyk:scan — Snyk CLI x.y.z</footer>
  </div>
</body>
</html>
```

HTML 生成時の注意：

- Markdown を先に完成させ、その内容を HTML に反映する（内容の不一致を避ける）
- ユーザー入力やスキャン結果に `<` `>` `&` `"` が含まれる場合は HTML エスケープする
- 中/低が多数の場合も Markdown と同様に集約表示可（緊急/高のみ `.vuln` ブロックを展開）

### 6. 最終出力

Markdown / HTML 両方のファイルパスをユーザーに伝える。チャットには「総合判定」と「検出サマリー表」のみを表示する（全文はファイルに任せる）。HTML はブラウザで開いて確認できる旨も一言添える。緊急 / 高 が 1 件でもあれば対応が必要である旨を明確に伝える。

## 注意

- スキャンには数十秒〜数分かかるので、必要に応じて `Bash` の timeout を 300000 ms 程度に設定
- 大規模プロジェクトで脆弱性が 50 件超になる場合は、中/低を集約表示（テーブルで列挙）し、緊急/高のみを詳細展開
- snyk のトークン期限切れエラーが出たら、レポートを作らずユーザーに `snyk auth` を促して終了
- JSON 出力が `--json` 非対応プランで失敗する場合は、テキスト出力からのパースにフォールバック
