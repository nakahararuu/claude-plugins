# comment-loop

ユーザと Claude Code が **コメント** を介して非同期に対話しながら開発を進めるための Claude Code スキル集。`/loop` と組み合わせて、エージェントループの中で継続的に使うことを想定しています。

## 収録スキル

### pr-comment-responder

GitHub PR のコメントを監視し、質問への回答・コード修正・返信投稿をまとめて行うスキル。

- PR の review comments と issue comments を両方チェック
- 未対応コメントを自動検出（"by ClaudeCode" マーカーで既返信を除外）
- 質問には説明を返信
- コード修正が必要な場合は fix → commit → push → commit リンク付き返信
- 全返信に `— by ClaudeCode` マーカーを付与

### todo-comment-responder

コードに書かれた TODO コメントを指示として読み取り、1つの TODO につき1コミットで実装していくスキル。

- `git diff HEAD`（未コミットの差分）から新しく追加された TODO コメントを検出
- TODO を1つずつ完全に処理してからコミット（同じファイル内に複数 TODO があっても、コミットが混ざらない）
- コミット済みになった TODO は差分から消えるため、次回実行時は自動的にスキップされる（重複対応の心配は不要）
- 意図が曖昧な TODO には、コミットせずにコード上へ確認コメントを追記

## インストール

```bash
claude plugin install comment-loop@nakahararuu-claude-plugins
```

## 使い方

**PR コメントに対応する場合：**

現在チェックアウトしているブランチに紐づく PR を `gh pr view` で自動的に特定します。URL は渡せません（コミットを push できるのは現在のブランチの PR だけであり、別の PR を指定できてしまうと挙動が予測しにくくなるため）。

```
/pr-comment-responder
```

```
/loop 1m /pr-comment-responder
```

**コード中の TODO コメントに対応する場合：**

```
/todo-comment-responder
```

```
/loop 1m /todo-comment-responder
```

## 返信・コミット例

**PR でのコード修正の場合：**
```
{原因の説明と修正内容}

Fixed in commit `abc1234` (https://github.com/owner/repo/commit/abc1234...) — by ClaudeCode
```

**PR での質問への回答：**
```
{回答}

— by ClaudeCode
```

**TODO コメントの実装の場合（コミットメッセージ）：**
```
todo: retry logic に exponential backoff を追加

Resolves: "TODO: add exponential backoff to the retry logic"
```
