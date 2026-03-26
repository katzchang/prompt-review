# SPLクエリ集

prompt-review-org スキルで使用するSPLクエリの定義。
すべてのクエリは `mcp__splunk-mcp-server__splunk_run_query` で実行する。

## 共通: ベースフィルタ

すべてのクエリに共通するベースフィルタ。ユーザーのプロンプト直接入力のみを抽出する。

```
index=claude_code sourcetype="claude:session" type="user"
  NOT "tool_result" NOT "tool_use_id"
| spath output=prompt_text path="message.content"
| where isnotnull(prompt_text)
  AND NOT match(prompt_text, "^<")
  AND NOT match(prompt_text, "^\[Request")
```

**フィルタの意味:**
- `type="user"` — ユーザーイベントのみ
- `NOT "tool_result" NOT "tool_use_id"` — ツール実行結果の返却を除外
- `spath output=prompt_text path="message.content"` — content が文字列のものを抽出
- `NOT match(prompt_text, "^<")` — `<local-command-stdout>` 等のシステムタグを除外
- `NOT match(prompt_text, "^\[Request")` — 中断通知を除外

## 1. サマリー統計クエリ

全体像の把握に使用。ユーザー（host）別の集計。

```spl
index=claude_code sourcetype="claude:session" type="user"
  NOT "tool_result" NOT "tool_use_id"
| spath output=prompt_text path="message.content"
| where isnotnull(prompt_text)
  AND NOT match(prompt_text, "^<")
  AND NOT match(prompt_text, "^\[Request")
| eval project=replace(cwd, ".*/", ""),
       prompt_len=len(prompt_text)
| stats
    count AS total_messages,
    dc(sessionId) AS sessions,
    dc(project) AS projects,
    avg(prompt_len) AS avg_prompt_len,
    values(project) AS project_list
  by host
| sort -total_messages
```

**パラメータ:**
- `earliest_time`: 引数から算出（デフォルト `-7d`）
- `row_limit`: 100（デフォルト）

## 2. プロジェクト別集計クエリ

プロジェクト単位の活動量を把握。

```spl
index=claude_code sourcetype="claude:session" type="user"
  NOT "tool_result" NOT "tool_use_id"
| spath output=prompt_text path="message.content"
| where isnotnull(prompt_text)
  AND NOT match(prompt_text, "^<")
  AND NOT match(prompt_text, "^\[Request")
| eval project=replace(cwd, ".*/", "")
| stats count AS messages, dc(host) AS users, dc(sessionId) AS sessions by project
| sort -messages
```

**パラメータ:**
- `earliest_time`: 引数から算出
- `row_limit`: 50

## 3. プロンプト本文取得クエリ

定性分析用。プロンプトの全文を時系列で取得。

```spl
index=claude_code sourcetype="claude:session" type="user"
  NOT "tool_result" NOT "tool_use_id"
| spath output=prompt_text path="message.content"
| where isnotnull(prompt_text)
  AND NOT match(prompt_text, "^<")
  AND NOT match(prompt_text, "^\[Request")
  AND len(prompt_text) > 5
| eval project=replace(cwd, ".*/", "")
| table _time, host, sessionId, project, gitBranch, prompt_text
| sort _time
```

**パラメータ:**
- `earliest_time`: 引数から算出
- `row_limit`: 200

**特定ユーザーに絞る場合:**
ベースフィルタの直後に `host="*<フィルタ値>*"` を追加する。

## 4. 時系列推移クエリ（日別）

活動の推移を可視化するための補助クエリ。必要に応じて実行する。

```spl
index=claude_code sourcetype="claude:session" type="user"
  NOT "tool_result" NOT "tool_use_id"
| spath output=prompt_text path="message.content"
| where isnotnull(prompt_text)
  AND NOT match(prompt_text, "^<")
  AND NOT match(prompt_text, "^\[Request")
| timechart span=1d count by host
```

**パラメータ:**
- `earliest_time`: `-30d`（推移を見るため長めの期間）
