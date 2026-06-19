# Snyk CLI を用いた脆弱性スキャン & 継続監視の Claude Code プラグイン

Snyk CLI を用いた脆弱性スキャン (snyk:scan) と継続監視 (snyk:monitor) を提供する Claude Code プラグインです。

> **非公式プラグイン**: Snyk 社公式ではありません。利用には [Snyk CLI](https://docs.snyk.io/snyk-cli) のインストールと認証が必要です。

## インストール手順

### 1. snyk CLI のインストール

OSや環境に応じていずれかを選択：

**macOS（推奨: Homebrew）**

```bash
brew tap snyk/tap
brew install snyk
```

**npm（クロスプラットフォーム）**

```bash
npm install -g snyk
```

**バイナリ直接ダウンロード**

公式リリースから取得：[https://github.com/snyk/cli/releases](https://github.com/snyk/cli/releases)

### 2. snyk 認証

```bash
snyk auth
```

ブラウザが開き、Snykアカウントとの連携画面が表示されます。
組織で利用している既存のSnykアカウントでログインしてください。

認証確認：

```bash
snyk config get api
```

トークンが表示されれば認証済みです。

### 3. Snyk Code を有効化

`snyk code test` を利用するには、Snyk組織側で Snyk Code 機能を ON にしておく必要があります。

1. [https://app.snyk.io/](https://app.snyk.io/) にログイン
2. 対象の Organization を選択
3. **Settings** → **Snyk Code** を開く
4. **Enable Snyk Code** をトグルして ON にする

組織管理者権限が必要です。権限がない場合は管理者に依頼してください。

### 4. マーケットプレースを追加

Claude Code を起動した状態で以下を実行：

```
/plugin marketplace add git@github.com:bellx2/snyk-claude-plugin.git
```

### 5. プラグインをインストール

```
/plugin install snyk@snyk-security
```

### 6. 動作確認

Claude Code内で「snykで脆弱性スキャンして」などと依頼すれば、
`snyk:scan` skill が自動的に呼び出されます。継続監視を登録したい場合は
「snyk monitorに登録して」などで `snyk:monitor` skill が呼び出されます。

## 更新

マーケットプレースを更新すると、インストール済みのプラグインも最新版に反映されます。

```
/plugin marketplace update snyk-security
```

`/plugin` →「Marketplaces」タブから `Enable auto-update` を有効化しておくと、以降は自動で更新されます。

## アンインストール

```
/plugin uninstall snyk@snyk-security
```
