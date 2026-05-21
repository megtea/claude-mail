# claude-mail

Claude Code 用の skills / routines 置き場。

## Skills

- [`business-card-scan`](.claude/skills/business-card-scan/SKILL.md) — Google Drive 上の名刺画像を読み取り、抽出結果を CSV に追記する。

## Routines

毎日 9:00 (JST) に「名刺スキャン → CSV書き出し」Routine が起動し、本リポジトリを clone した上で `business-card-scan` Skill の手順に従って処理を行う。
