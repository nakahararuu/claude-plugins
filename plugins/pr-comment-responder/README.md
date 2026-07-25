# pr-comment-responder

GitHub PR のコメントを監視し、質問への回答・コード修正・返信投稿をまとめて行う Claude Code スキル。

## 機能

- PR の review comments と issue comments を両方チェック
- 未対応コメントを自動検出（"by ClaudeCode" マーカーで既返信を除外）
- 質問には説明を返信
- コード修正が必要な場合は fix → commit → push → commit リンク付き返信
- 全返信に `— by ClaudeCode` マーカーを付与

## インストール

```bash
claude plugin install pr-comment-responder@nakahararuu-claude-plugins
```

## 使い方

一度だけ実行：

```
/pr-comment-responder https://github.com/owner/repo/pull/123
```

ループ監視：

```
/loop 1m /pr-comment-responder https://github.com/owner/repo/pull/123
```

## 返信例

**コード修正の場合：**
```
{原因の説明と修正内容}

Fixed in commit `abc1234` (https://github.com/owner/repo/commit/abc1234...) — by ClaudeCode
```

**質問への回答：**
```
{回答}

— by ClaudeCode
```
