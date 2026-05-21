# Claude Code セットアップ: 名刺スキャン & 挨拶メール下書き

このドキュメントは Claude Code セッションに貼り付けて使うセットアッププロンプトです。
利用者環境にクラウドRoutineを2つ作成します。

## このプロンプトでセットアップされるもの

1. **名刺スキャン → CSV書き出し** （毎日 09:00 JST）
   - Google Driveの名刺画像フォルダから画像を読み取り、`customer_list.csv` に追記
   - 判読不安(「要確認」)・エラーがあれば通知先メールに Gmail で通知
2. **挨拶メール下書き作成** （毎日 10:00 JST、任意）
   - 上記CSVから「未処理」行を読み、Gmail で挨拶メールの下書きを作成

両RoutineともGitHub公開リポジトリ [`megtea/claude-mail`](https://github.com/megtea/claude-mail) を clone し、その中の Skill 定義（SKILL.md / templates/greeting.md）に従って動作します。

利用者ごとに変わるのは:
- Google Drive のフォルダID
- 通知先メールアドレス
- （任意）挨拶メールテンプレートの中身

---

## Claude への指示

**重要: あなた（Claude）はこのドキュメント以下の手順を順番に実行してください。各ステップを完了するまで次に進まないでください。**

### Step 0: 前提条件の確認

利用者環境で以下が必要です:

- Claude.ai にログイン済み（Routines 機能を使うため）
- Google Drive / Gmail MCP コネクタが接続済み（https://claude.ai/customize/connectors ）

利用者にひと声「Drive・Gmail コネクタは接続済みですか?」と確認してください。未接続の場合は接続を案内して停止。

### Step 1: 必要情報の収集

`AskUserQuestion` で以下4点を順番に質問してください。

| # | 質問 | 形式 | 補足 |
|---|---|---|---|
| 1 | 名刺画像が入ったGoogle DriveのフォルダIDは? | 自由入力 | Drive URL `https://drive.google.com/drive/folders/{ID}` の `{ID}` 部分 |
| 2 | 「要確認」やエラーが出たときの通知先メールアドレスは? | 自由入力 | 自分宛運用を想定 |
| 3 | CSV保存先のフォルダID(任意)は? | 自由入力 | 空欄なら名刺画像フォルダと同じ場所 |
| 4 | 挨拶メール下書きRoutineも一緒に作りますか? | 選択 | 「両方作成 (推奨)」/「名刺スキャンのみ」 |

選択肢で AskUserQuestion を呼ぶとき、自由入力には「他」フォールバックを利用させてください。

### Step 2: 環境情報のロード

a. `schedule` スキルを呼び出してください（引数は `list`）。これにより、利用者の `environment_id`(anthropic_cloud kind) と MCP コネクタ(Gmail / Google-Drive)の UUID と URL がコンテキストに読み込まれます。**この情報は利用者ごとに異なるため、megtea の値を流用しないこと。**

b. `ToolSearch` で `select:RemoteTrigger` を呼び出して `RemoteTrigger` ツールをロード。

`schedule` の出力に Drive / Gmail コネクタが見つからなければ、未接続なので Step 0 に戻ってください。

### Step 3: 名刺スキャンRoutineを作成

`RemoteTrigger` を `action="create"` で呼び出します。`body` は以下のテンプレートに従い、`{...}` のプレースホルダを Step 1 / Step 2 の値で置換してください。

```jsonc
{
  "name": "名刺スキャン → CSV書き出し",
  "cron_expression": "0 0 * * *",      // = 09:00 JST 毎日
  "enabled": true,
  "job_config": {
    "ccr": {
      "environment_id": "{利用者の anthropic_cloud 環境ID}",
      "session_context": {
        "allowed_tools": ["Bash", "Read", "Write", "Edit", "Glob", "Grep"],
        "sources": [{"git_repository": {"url": "https://github.com/megtea/claude-mail"}}]
      },
      "events": [{
        "data": {
          "uuid": "{新規に生成した小文字 v4 UUID}",
          "session_id": "",
          "type": "user",
          "parent_tool_use_id": null,
          "message": {
            "role": "user",
            "content": "{下記「Routine A プロンプト本文」を、{名刺画像フォルダID}・{CSV保存先フォルダID}・{通知先メールアドレス}で置換したもの}"
          }
        }
      }]
    }
  },
  "mcp_connections": [
    {"connector_uuid": "{利用者のGmailコネクタUUID}",       "name": "Gmail",         "url": "{利用者のGmailコネクタURL}"},
    {"connector_uuid": "{利用者のGoogle-DriveコネクタUUID}", "name": "Google-Drive",  "url": "{利用者のGoogle-DriveコネクタURL}"}
  ]
}
```

レスポンスの `trigger.id`(例: `trig_xxxxxxxxxxxxxxxxxxxxxxxx`)を控えてください。

### Step 4: 挨拶下書きRoutineを作成（Step 1 質問4 で「両方作成」を選んだ場合のみ）

同様に `RemoteTrigger` `action="create"` を呼び出します。Step 3 との差分は以下:

- `name`: `挨拶メール下書き作成`
- `cron_expression`: `0 1 * * *` （= 10:00 JST 毎日、名刺スキャンの1時間後）
- `events[].data.uuid`: **新規生成した別のUUID**
- `events[].data.message.content`: 「Routine B プロンプト本文」を `{CSV保存先フォルダID}` で置換したもの
- `mcp_connections`: Gmail と Google-Drive の2つ（Routine A と同じ）

### Step 5: 動作検証

Routine Aを `RemoteTrigger` `action="run"` で1回手動実行してください。

- **HTTP 200** が返れば、リポジトリのclone権限と起動環境はOK。利用者には「クラウド側で実行を開始しました。数分後にRoutine画面のRunログで結果を確認できます」と伝える。
- **HTTP 400 で `github_repo_access_denied`** が返った場合、利用者に「Claude.ai の設定で GitHub の再認可が必要です。https://claude.ai/settings からGitHub連携を確認してください」と案内する。リポジトリ `megtea/claude-mail` は public なので通常このエラーは出ないはずだが、念のため。

### Step 6: 完了報告

以下のフォーマットで利用者に報告してください:

```
✅ セットアップ完了

【Routine 1】名刺スキャン → CSV書き出し
- ID: trig_xxxx
- 次回実行: 明日 09:00 JST
- URL: https://claude.ai/code/routines/{trigger_id}

【Routine 2】挨拶メール下書き作成 (作成した場合のみ)
- ID: trig_yyyy
- 次回実行: 明日 10:00 JST
- URL: https://claude.ai/code/routines/{trigger_id}

【テスト実行の結果】
- 名刺スキャンRoutineをdispatchしました。数分後にRoutine画面でログを確認してください。

【テンプレートを差し替えたい場合】
挨拶メールの文面を変えたい場合は、リポジトリ `megtea/claude-mail` を fork して
`.claude/skills/greeting-draft/templates/greeting.md` を編集 → push し、
Routine 2 の `sources` をその fork URL に更新してください。
```

---

## Routine A プロンプト本文（名刺スキャン）

以下を、`{名刺画像フォルダID}`・`{CSV保存先フォルダID}`・`{通知先メールアドレス}` のプレースホルダを Step 1 の利用者回答で置換して、`events[].data.message.content` に渡してください。

CSV保存先フォルダIDが空欄だった場合は「（未指定の場合は名刺画像フォルダと同じ場所に作成）」をそのまま残すこと。

````
# 名刺スキャン → CSV書き出し Routine

## 【最初に必ず実行】
このRoutineは GitHub リポジトリ `megtea/claude-mail` を clone した状態で起動する。
作業に入る前に、必ず以下を Read してその内容に従うこと:

- `.claude/skills/business-card-scan/SKILL.md`

SKILL.md には CSVの列定義・抽出ルール・誤読対策・重複処理の防止・処理結果の通知・やらないこと、が定義されている。
本プロンプトと SKILL.md に矛盾があった場合は SKILL.md を優先する。
本プロンプトはRoutine固有のパラメータと処理フローを定義するに留める。

## 【設定】※提供先はこのブロックのみ書き換える
- 名刺画像フォルダID: {名刺画像フォルダID}
- 出力CSVファイル名: customer_list.csv
- CSV保存先フォルダID: {CSV保存先フォルダID}
- 対象画像拡張子: .jpg .jpeg .png .heic .webp
- 1回の実行で処理する最大枚数: 30
- 通知先メールアドレス: {通知先メールアドレス}

## 【処理フロー】
SKILL.md の手順に従って、以下を実行する。

1. 設定の「名刺画像フォルダ」内にある画像ファイル一覧を Google Drive コネクタで取得する。
2. 設定の「出力CSV」が既に存在すれば読み込む。存在しなければ、SKILL.md に定義されたヘッダー行のみのCSVを新規作成する。
3. CSVの「名刺画像ファイル名」列に既に記録されている画像はスキップする（SKILL.md「重複処理の防止」）。
4. 未処理の画像について、1枚ずつ内容を読み取り、SKILL.md の「抽出ルール」に従って項目を抽出する。
5. 抽出結果をCSVに1行ずつ追記する（「ステータス」「交流場所」「交換日」「紹介者」列の扱いは SKILL.md に従う）。
6. 1回の実行で処理する枚数は設定の上限を超えない。超える分は次回実行に回す。
7. 「要確認」ステータスの行や、抽出に失敗してスキップした画像が1件以上あれば、設定の「通知先メールアドレス」宛てに Gmail コネクタで通知メールを送信する（SKILL.md「処理結果の通知」の仕様に従う）。要確認・エラーがゼロの場合は送らない。
8. 完了後、今回の処理結果を要約して報告する（新規追記件数、要確認件数、スキップ件数、通知メールの送信有無）。

## 【厳守事項】
- SKILL.md の「やらないこと」セクションを必ず守る。名刺の差出人本人へのメールは一切送らない（送信するのはあくまで設定の「通知先メールアドレス」だけ）。
- CSVの既存行は絶対に上書き・並べ替え・削除しない。必ず末尾への追記のみ行う。
- 判読に自信が持てない場合は、SKILL.md の指示通り「ステータス」列を「要確認」とする。
````

---

## Routine B プロンプト本文（挨拶メール下書き）

`{CSV保存先フォルダID}` のプレースホルダを Step 1 の利用者回答で置換してください（空欄だった場合は名刺画像フォルダと同じIDを使用）。

````
# 挨拶メール下書き作成 Routine

## 【最初に必ず実行】
このRoutineは GitHub リポジトリ `megtea/claude-mail` を clone した状態で起動する。
作業に入る前に、必ず以下を Read してその内容に従うこと:

- `.claude/skills/greeting-draft/SKILL.md`
- `.claude/skills/greeting-draft/templates/greeting.md`

SKILL.md には処理手順・差し込み項目・厳守事項・やらないこと、が定義されている。
templates/greeting.md には件名と本文テンプレートが定義されている。
本プロンプトと SKILL.md に矛盾があった場合は SKILL.md を優先する。
本プロンプトはRoutine固有のパラメータと処理フローを定義するに留める。

## 【設定】※提供先はこのブロックのみ書き換える
- 入力CSVファイル名: customer_list.csv
- CSV保存先フォルダID: {CSV保存先フォルダID}
- メールテンプレート: .claude/skills/greeting-draft/templates/greeting.md
- 1回の実行で作成する下書きの最大件数: 20

## 【処理フロー】
SKILL.md の手順に従って、以下を実行する。

1. 設定の「入力CSV」を Google Drive コネクタで読み込む。
2. 各行の「ステータス」列を確認し、「未処理」の行だけを対象とする。「下書き済」「送信済」「要確認」の行はスキップする。
3. 「メールアドレス」列が空欄の行はスキップし、報告に含める。
4. 対象行ごとに、templates/greeting.md の差し込み項目（`{{氏名}}` 等）をCSVの値で置き換え、Gmail コネクタで下書きを1件作成する。件名は templates/greeting.md の frontmatter の「件名:」を使用する。
5. 下書き作成に成功した行は、CSVの「ステータス」列を「下書き済」に更新する（他の列は変更しない）。
6. 1回の実行で作成する件数は設定の上限を超えない。超える分は次回に回す。
7. 完了後、今回の処理結果を要約して報告する（下書き作成件数、スキップ件数とその理由）。

## 【厳守事項】
- SKILL.md の「やらないこと」「厳守事項」セクションを必ず守る。
- 作成するのは「下書き」のみ。メールの送信・送信予約は絶対に行わない。
- 「ステータス」が「未処理」以外の行には下書きを作らない（二重作成の防止）。
- CSVの更新は「ステータス」列のみ。他の列・他の行は変更しない。
````

---

## メールテンプレートをクライアント独自にしたい場合

デフォルトでは `megtea/claude-mail` の `templates/greeting.md` が使われます（汎用的なプレースホルダ入り）。
クライアント独自のSNSや会社紹介文に差し替える場合は、以下の手順を追加で踏んでください:

1. `gh repo fork megtea/claude-mail` で利用者のアカウントに fork
2. fork した repo を clone し、`.claude/skills/greeting-draft/templates/greeting.md` を編集して push
3. Routine 2 の `sources` を `https://github.com/{利用者のGitHubユーザ名}/claude-mail` に更新

このセットアッププロンプト自体ではテンプレート差し替えまでは行いません（個別カスタマイズが必要なため）。
