# bell-notify

Claude Code が承認待ち・入力待ちになったとき、またはタスクへの応答を終えたときに、端末のベル（BEL文字）を鳴らす hooks だけのプラグインです。

## 何をするか

- `Notification` イベント（ツール実行の承認確認や、入力待ちで一定時間アイドルになったときなど）でベルを鳴らします。
- `Stop` イベント（Claude Code が応答を終えたとき）でベルを鳴らします。

`/dev/tty` に BEL文字（`\a`）を書き込むだけで、外部コマンドや依存ライブラリは使用しません。tty が存在しない環境（CI など）では黙って何もしません。

## 必要な設定

多くの端末エミュレータではデフォルトでベル音（またはビジュアルベル）が鳴りますが、鳴らない場合は端末側の設定でベルを有効にしてください（例: iTerm2 では *Preferences > Profiles > Terminal > Notifications* で `Audio bell` を有効化）。

## インストール

このプラグインが含まれるマーケットプレイスを追加してからインストールします。詳細はリポジトリルートの README を参照してください。

```
/plugin marketplace add nakahararuu/claude-plugins
/plugin install bell-notify@nakahararuu-claude-plugins
```
