# claude-plugins

nakahararuu が作成した Claude Code 用の skills / hooks / MCP サーバーをまとめて配布するプラグインマーケットプレイスです。

## マーケットプレイスの追加

Claude Code 上で以下を実行します。

```
/plugin marketplace add nakahararuu/claude-plugins
```

追加後、プラグイン一覧の確認やインストールは `/plugin` コマンド（プラグイン管理 UI）から行えます。CLI から直接インストールする場合は次のようにします。

```
/plugin install <plugin-name>@nakahararuu-claude-plugins
```

## 収録プラグイン

| プラグイン | 説明 |
| --- | --- |
| [bell-notify](plugins/bell-notify) | Claude Code が承認・入力を必要としたとき、およびタスク完了時に端末のベルを鳴らす hooks |
| [comment-loop](plugins/comment-loop) | PR コメントやコード中の TODO コメントを介して、agent-loop の中で Claude と対話しながら開発を進める skill 集（pr-comment-responder / todo-comment-responder） |

## プラグインの追加方法（開発者向け）

1. `plugins/<plugin-name>/` にプラグイン本体を作成する（`.claude-plugin/plugin.json` は必須。`commands/`, `agents/`, `skills/`, `hooks/hooks.json`, `.mcp.json` は規約に沿って配置すれば自動検出されます）。
2. リポジトリルートの `.claude-plugin/marketplace.json` の `plugins` 配列に、`name` / `description` / `source`（例: `"./plugins/<plugin-name>"`）を追加する。
3. 変更をコミットして push すれば、マーケットプレイスを追加済みのユーザーに反映されます。
