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
- 時間のかかる対応（複数ファイルの改修、環境構築、ビルド・テストなど）はバックグラウンドのサブエージェントに任せ、受付返信（`— by ClaudeCode (in progress)`）だけ先に投稿。完了したらサブエージェントが同じスレッドに結果を返信する

### todo-comment-responder

コードに書かれた TODO コメントを指示として読み取り、1つの TODO につき1コミットで実装していくスキル。

- `git diff HEAD`（未コミットの差分）から新しく追加された TODO コメントを検出
- TODO を1つずつ完全に処理してからコミット（同じファイル内に複数 TODO があっても、コミットが混ざらない）
- コミット済みになった TODO は差分から消えるため、次回実行時は自動的にスキップされる（重複対応の心配は不要）
- 意図が曖昧な TODO には、コミットせずにコード上へ確認コメントを追記
- 時間のかかりそうな TODO を含む場合は、その回の TODO 一式をバックグラウンドのサブエージェントに任せる（1 TODO 1 コミットの規約はそのまま）

## `/loop` での動作（バックグラウンド委譲）

`/loop` の 1 回（tick）が長引くと、その間は新しいコメントや TODO に反応できません。そこで両スキルとも、メインのループは「取得・振り分け・すぐ終わる対応・サブエージェントの起動・報告」だけを行い、時間のかかる作業は Agent ツールの `run_in_background: true` で起動したサブエージェントに任せます。

- インラインで対応するもの: 質問への回答、数行程度の局所的な修正（確認は lint や速い単体テスト程度）
- サブエージェントに任せるもの: 複数ファイルにまたがる改修、依存インストールやサービス起動などの環境構築、ビルド・テストスイート・Docker での再現が必要なもの、長い調査
- 作業ツリーの競合を避けるため、同時に動くサブエージェントは 1 つだけ。動いている間、メインはコードを変更せず、次のコード変更系の依頼は完了後の tick に回す
- PR コメントの受付返信は `— by ClaudeCode (in progress)` で終わる。サブエージェントが最終返信を投稿しないまま終了した場合や、セッションが再起動された場合は、次の tick で未対応として拾い直す
- Agent ツールが使えない環境では従来どおりインラインで対応する

## インストール

```bash
claude plugin install comment-loop@nakahararuu-claude-plugins
```

## 使い方

**PR コメントに対応する場合：**

現在チェックアウトしているブランチに紐づく PR を `gh pr view` で自動的に特定します。

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
