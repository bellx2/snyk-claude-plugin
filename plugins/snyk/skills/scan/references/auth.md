# snyk の導入と認証

SKILL.md の「1. 事前確認」で、未インストールまたは未認証と分かったときに読む。

## 未インストールのとき

レポート作成に進まず以下のメッセージを出して終了する：

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

## 自動認証フロー（未認証時）

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
