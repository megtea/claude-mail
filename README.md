# claude-mail

Claude Code 用の skills / routines 置き場。

## Skills

- [`business-card-scan`](.claude/skills/business-card-scan/SKILL.md) — Google Drive 上の名刺画像を読み取り、抽出結果を CSV に追記する。
- [`greeting-draft`](.claude/skills/greeting-draft/SKILL.md) — CSV を読み込み、未処理の相手に Gmail 挨拶メール下書きを作成する。テンプレートは [`templates/greeting.md`](.claude/skills/greeting-draft/templates/greeting.md)。

## Routines

- 毎日 9:00 (JST) — 「名刺スキャン → CSV書き出し」: `business-card-scan` Skill を実行
- 毎日 10:00 (JST) — 「挨拶メール下書き作成」: `greeting-draft` Skill を実行

いずれも本リポジトリを clone した状態で起動する。

## クライアント環境へのデプロイ

クライアントの Claude Code で同じ仕組みをセットアップするには、[SETUP_PROMPT.md](SETUP_PROMPT.md) を参照。
クライアントには以下の一行をClaude Codeに貼り付けてもらえばOK:

```
このURLの内容を取得して、書かれた指示に従ってクラウドRoutineをセットアップしてください: https://raw.githubusercontent.com/megtea/claude-mail/main/SETUP_PROMPT.md
```
