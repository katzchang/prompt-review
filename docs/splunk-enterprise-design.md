# prompt-review Splunk Enterprise 連携 設計書

## 概要

既存の prompt-review スキル（個人のローカルログ → 個人レポート）を、Splunk Enterprise を中心とした組織利用に拡張する。

**変わること:**
- データ収集: ローカルファイル読み取り → Splunk Enterprise からSPLクエリで取得
- 分析対象: 個人 → 組織全体（ユーザー横断）
- レポート: 個人向け → 組織向け（個人レポートも生成可能）

**変わらないこと:**
- LLMによるプロンプト内容の定性分析（技術理解度、プロンプティング力等）
- レポートの日本語出力

---

## アーキテクチャ

```
┌─────────────────────────────────────────────────────┐
│  各メンバーのPC                                       │
│                                                       │
│  Claude Code セッション JSONL                         │
│  (~/.claude/projects/*/*.jsonl)                       │
│                                                       │
│       ↓ filelog receiver                              │
│                                                       │
│  OTel Collector                                       │
│  ├── receiver: filelog (JSONL監視)                    │
│  ├── processor: batch                                │
│  └── exporter: splunk_hec                            │
└───────────────┬─────────────────────────────────────┘
                ↓ HEC (HTTPS)
┌───────────────────────────────────────────────────────┐
│  Splunk Enterprise                                     │
│                                                       │
│  index: claude_code                                   │
│  sourcetype: claude:session                           │
│                                                       │
│  生データ（セッションJSONL そのまま）:                   │
│  ├── type         (user / assistant / system / ...)   │
│  ├── message.content  (プロンプト本文 ※文字列時)       │
│  ├── sessionId    (セッション識別子)                    │
│  ├── cwd          (作業ディレクトリ → プロジェクト名)   │
│  ├── gitBranch    (ブランチ名)                         │
│  ├── host         (送信元ホスト → ユーザー識別)         │
│  ├── timestamp    (イベントタイムスタンプ)               │
│  └── entrypoint   (cli / vscode)                      │
└───────────────┬───────────────────────────────────────┘
                ↓ SPL via Splunk Enterprise MCP
┌───────────────────────────────────────────────────────┐
│  Claude Code + Splunk Enterprise MCP                   │
│                                                       │
│  /prompt-review-org スキル                             │
│  ├── ステップ1: SPLクエリでデータ取得                   │
│  ├── ステップ2: LLM分析（既存ロジック流用）             │
│  └── ステップ3: 組織レポート生成                        │
└───────────────────────────────────────────────────────┘
```

**設計方針**: collect.py による前処理は行わず、Claude Code のセッション JSONL を OTel Collector で
そのまま Splunk に送信する。ユーザープロンプトの抽出・整形は SPL クエリ側で行う。

---

## 1. Splunk 上のデータ構造

### 既存環境（検証済み）

| 項目 | 値 |
|------|-----|
| index | `claude_code` |
| sourcetype | `claude:session` |
| source | `otel-collector` |
| 送信方式 | OTel Collector filelog receiver → Splunk HEC |

### イベント種別（type フィールド）

| type | 件数目安 | 内容 |
|------|---------|------|
| `user` | 数百件 | ユーザー入力（プロンプト + tool_result返却） |
| `assistant` | 数百件 | アシスタント応答 |
| `file-history-snapshot` | 数十万件 | ファイル変更スナップショット（大量） |
| `system` | 数十件 | システムメッセージ |
| `progress` | 数十件 | 進捗通知 |
| `last-prompt` | 数件 | 最終プロンプト記録 |

### ユーザープロンプトの識別方法

Claude Code の `type="user"` イベントには2種類ある:

1. **ユーザーの直接入力** — `message.content` が**文字列**
2. **tool_result の返却** — `message.content` が**配列**（`tool_use_id` を含む）

分析対象は (1) のみ。以下で除外が必要:
- `tool_result` / `tool_use_id` を含むイベント（ツール実行結果）
- `<local-command-stdout>` 等のシステムタグで始まるテキスト
- `[Request interrupted` で始まるテキスト（中断通知）

---

## 2. データパイプライン

### OTel Collector 設定

```yaml
# otel-collector-config.yaml
receivers:
  filelog/session:
    include:
      - ${HOME}/.claude/projects/*/*.jsonl
    start_at: end
    operators:
      - type: json_parser
        timestamp:
          parse_from: attributes.timestamp
          layout: '%Y-%m-%dT%H:%M:%S.%LZ'
    resource:
      source: claude-code-session

processors:
  batch:
    timeout: 5s
    send_batch_size: 100

exporters:
  splunkhec:
    token: "${SPLUNK_HEC_TOKEN}"
    endpoint: "https://splunk.example.com:8088"
    source: "otel-collector"
    sourcetype: "claude:session"
    index: "claude_code"
    tls:
      insecure_skip_verify: false

service:
  pipelines:
    logs:
      receivers: [filelog/session]
      processors: [batch]
      exporters: [splunkhec]
```

### 組織展開時の追加考慮

- **ユーザー識別**: 現状は `host` フィールドでホスト名から識別。組織利用では OTel Collector の
  resource processor で `user` 属性を明示的に付与することを推奨
- **他AIツール対応**: 現状は Claude Code のみ。Copilot Chat, Cline 等を追加する場合は
  collect.py の OTel 出力オプション追加、または各ツール用の filelog receiver を追加

---

## 3. SPLクエリ設計（検証済み）

### 基本: ユーザープロンプト抽出

```spl
index=claude_code sourcetype="claude:session" type="user"
  NOT "tool_result" NOT "tool_use_id"
| spath output=prompt_text path="message.content"
| where isnotnull(prompt_text)
  AND NOT match(prompt_text, "^<")
  AND NOT match(prompt_text, "^\[Request")
| eval project=replace(cwd, ".*/", ""),
       prompt_len=len(prompt_text)
```

このベースクエリを `base_user_prompts` として以下の各クエリで参照する。

### 3a. プロンプト本文取得（定性分析用）

```spl
index=claude_code sourcetype="claude:session" type="user"
  NOT "tool_result" NOT "tool_use_id"
| spath output=prompt_text path="message.content"
| where isnotnull(prompt_text)
  AND NOT match(prompt_text, "^<")
  AND NOT match(prompt_text, "^\[Request")
  AND len(prompt_text) > 20
| eval project=replace(cwd, ".*/", "")
| table _time, host, sessionId, project, gitBranch, prompt_text
| sort _time
| head 200
```

特定ユーザー（ホスト）に絞る場合: `host="tanaka-macbook"` を追加

### 3b. サマリー統計（定量分析用）

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

### 3c. プロジェクト別集計

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

### 3d. 時系列推移（日別）

```spl
index=claude_code sourcetype="claude:session" type="user"
  NOT "tool_result" NOT "tool_use_id"
| spath output=prompt_text path="message.content"
| where isnotnull(prompt_text)
  AND NOT match(prompt_text, "^<")
  AND NOT match(prompt_text, "^\[Request")
| timechart span=1d count by host
```

### 3e. 短文肯定応答の除外（分析精度向上）

既存スキルと同様、短文の肯定応答を除外する追加フィルタ:

```spl
| where prompt_len > 20
  OR NOT match(prompt_text, "^(y|yes|はい|うん|ok|sure|yep|yeah|進めて|やって|do it|go|go ahead|proceed|それで|お願いします|いいよ|大丈夫|ありがとう|thanks|thx)$")
```

---

## 4. スキル設計（prompt-review-org）

### 前提: Splunk Enterprise MCP

MCP ツール `mcp__splunk-mcp-server__splunk_run_query` を使用してSPLクエリを実行する。

### SKILL.md 構成

```
.claude/skills/prompt-review-org/
├── SKILL.md                    # スキル定義
└── references/
    ├── spl-queries.md          # SPLクエリ集（上記セクション3の内容）
    └── report-template-org.md  # 組織レポートテンプレート
```

### ステップ1: SPLクエリでデータ取得

1. サマリー統計（3b）を実行 → ユーザー・プロジェクトの全体像を把握
2. プロジェクト別集計（3c）を実行 → プロジェクト活動の概観
3. プロンプト本文（3a）を実行 → 定性分析用のテキストデータ取得

### ステップ2: 分析

取得したデータを既存スキルと同じ分析フレームワークで処理する。

**定量分析（SPLの集計結果から）:**
- ユーザー別アクティビティランキング
- プロンプト長の分布・推移
- プロジェクト別活動量
- セッション数・頻度

**定性分析（プロンプト本文から、既存ロジック流用）:**
- 技術理解度マップ（ユーザー別）
- プロンプティングパターン分析
- AI依存度分析
- 成長の軌跡

### ステップ3: レポート生成

`reports/prompt-review-org-YYYY-MM-DD.md` に出力。

---

## 5. 組織レポートテンプレート（追加セクション）

既存の個人レポートテンプレートに以下を追加:

### 組織サマリー（新規）

```markdown
## 1. 組織サマリー

- **分析期間**: YYYY-MM-DD 〜 YYYY-MM-DD
- **アクティブユーザー数**: N名
- **総メッセージ数**: N件（短文肯定応答M件を除外）
- **アクティブプロジェクト数**: N件

### ユーザー別アクティビティ

| ユーザー(host) | メッセージ数 | セッション数 | プロジェクト数 | 平均プロンプト長 |
|---------------|-------------|-------------|--------------|----------------|

### プロジェクト別アクティビティ

| プロジェクト | メッセージ数 | ユーザー数 | セッション数 | 主な作業内容 |
|-------------|-------------|-----------|-------------|-------------|
```

### ユーザー別分析（既存を拡張）

```markdown
## N. ユーザー別分析: [ホスト名/ユーザー名]

### 技術理解度マップ
（既存テンプレートと同じ）

### プロンプティング力の評価
（既存テンプレートと同じ）

### AI活用スタイル
（既存テンプレートと同じ）
```

### 組織横断の傾向（新規）

```markdown
## N. 組織横断の傾向

### 共通の強みパターン
- チーム全体で見られる効果的なプロンプティングパターン

### 共通の改善ポイント
- 複数ユーザーに共通する改善可能なパターン

### 推奨アクション
1. チーム向け施策（ベストプラクティス共有等）
2. 個別フォローが望ましいケース
```

---

## 6. 引数設計

### 組織レポート（一括分析）
```
/prompt-review-org                    # 全ユーザー、過去7日分
/prompt-review-org 30                 # 過去30日分
```

### 個人レポート（ユーザー指定）
```
/prompt-review-org tanaka             # 特定ユーザーのみ、過去7日分
/prompt-review-org tanaka 30          # 特定ユーザー × 過去30日分
```

ユーザー指定時は組織横断セクションを省略し、既存の個人向けスキル（prompt-review）と
同等の深さで個人分析レポートを生成する。出力ファイル名は
`reports/prompt-review-org-YYYY-MM-DD-<user>.md` とする。

---

## 7. 実装ステップ

1. ~~Splunk Enterprise 環境準備~~ → **完了**
   - index `claude_code` 作成済み
   - HEC 設定済み
   - sourcetype `claude:session` で受信中

2. ~~データパイプライン構築~~ → **完了**
   - OTel Collector filelog receiver でセッション JSONL を監視
   - Splunk HEC exporter で送信
   - データ受信確認済み

3. ~~Splunk Enterprise MCP 追加~~ → **完了**
   - `mcp__splunk-mcp-server__splunk_run_query` で SPL 実行可能
   - 接続確認済み（Splunk Enterprise 9.4.3）

4. ~~スキル作成~~ → **完了**
   - `prompt-review-org/SKILL.md` 作成済み
   - SPLクエリ集作成済み
   - 組織レポートテンプレート作成済み
   - 動作確認済み（2026-03-26）

5. **個人レポート対応**
   - ユーザー指定時に個人特化レポートを生成
   - 出力ファイル名: `prompt-review-org-YYYY-MM-DD-<user>.md`
   - 組織横断セクションの省略ロジック

6. **ユーザー識別の改善**
   - OTel Collector resource processor で `user` 属性を付与
   - `host` ベース → `user` ベースに切り替え
   - SPLクエリ・レポートテンプレートの `host` → `user` 更新

7. **テスト・調整**
   - 実データでの動作確認
   - SPLクエリのチューニング
   - レポート品質の調整

---

## 8. プライバシー・セキュリティ考慮

社内利用前提だが、以下は検討が必要:

- **アクセス制御**: `claude_code` index へのアクセスをマネージャー/管理者に限定
- **シークレット検出**: 生ログにはクレデンシャルが含まれる可能性あり。
  SPL側での検出・マスク、または OTel Collector の redaction processor での事前マスクを検討
- **データ保持期間**: Splunk 側の retention policy で管理
- **利用目的の周知**: メンバーにログ収集の目的・範囲を事前に説明
- **アシスタント応答**: `type="assistant"` にもコード等が含まれる。分析対象は `type="user"` に限定

---

## 9. TODO: 将来の拡張

### データ拡充

- [ ] OTel resource processor で `user`, `team`, `role` 属性を付与
- [ ] `team` / `role` は Splunk ルックアップテーブルでの管理も検討（異動対応）

### assistant メッセージを活用した分析

現状は `type="user"` のみ分析対象だが、`type="assistant"` も活用すると以下が可能:

- [ ] **イテレーション回数分析**: セッション内の user/assistant 往復数 → 1回で伝わるか、何度もやり取りするか
- [ ] **ツール利用パターン**: assistant の `tool_use.name`（Read/Edit/Bash等） → どんな作業にAIを使っているか
- [ ] **エラー率**: assistant 応答中のエラー言及 → 手戻りの多い領域の特定
- [ ] **トークン消費**: `usage.input_tokens`, `output_tokens` → ユーザー/プロジェクト別のAIコスト可視化
- [ ] **応答時間**: `durationMs` → 重いタスクの特定

### 組織横断の高度な分析

- [ ] **「イテレーション回数 × 技術領域」クロス分析** — 組織として学習コストが高い技術の特定
- [ ] **チーム間ベストプラクティス比較** — あるチームの効果的パターンを他チームに横展開
- [ ] **トークン消費のプロジェクト別集計** — AIコストの可視化・予算配分の根拠
- [ ] **オンボーディング効果測定** — 新メンバーのプロンプト品質・AI活用度の立ち上がり速度
- [ ] **経験年数 × AI活用パターン** — シニア/ジュニアでの活用の違い
