# HTML レポートの作り方

SKILL.md の「5. レポート出力」で HTML 版を書くときに読む。Markdown 版（`report-markdown.md`）を先に完成させてから作る。

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
