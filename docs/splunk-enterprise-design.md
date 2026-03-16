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
│  AIツールのログ → collect.py --emit-otel              │
│  (Claude Code, Copilot, Cline, etc.)                  │
│                                                       │
│       ↓ OTLP (gRPC/HTTP)                              │
│                                                       │
│  OTel Collector                                       │
│  ├── receiver: filelog / otlp                         │
│  ├── processor: attributes (user, host)               │
│  └── exporter: splunk_hec                             │
└───────────────┬─────────────────────────────────────┘
                ↓ HEC (HTTPS)
┌───────────────────────────────────────────────────────┐
│  Splunk Enterprise                                     │
│                                                       │
│  index: ai_prompt_logs                                │
│  sourcetype: ai_prompt_review                         │
│                                                       │
│  フィールド:                                           │
│  ├── user        (メンバー名)                          │
│  ├── tool        (Claude Code, Copilot, etc.)         │
│  ├── project     (プロジェクト名)                      │
│  ├── prompt_text (プロンプト本文)                      │
│  ├── prompt_len  (文字数)                              │
│  ├── session_id  (セッション識別子)                    │
│  └── _time       (タイムスタンプ)                      │
└───────────────┬───────────────────────────────────────┘
                ↓ SPL via MCP
┌───────────────────────────────────────────────────────┐
│  Claude Code + Splunk Enterprise MCP                   │
│                                                       │
│  /prompt-review-org スキル                             │
│  ├── ステップ1: SPLクエリでデータ取得                   │
│  ├── ステップ2: LLM分析（既存ロジック流用）             │
│  └── ステップ3: 組織レポート生成                        │
└───────────────────────────────────────────────────────┘
```

---

## 1. ログスキーマ設計

### Splunk イベント形式

```json
{
  "_time": 1710335400,
  "user": "tanaka",
  "tool": "Claude Code",
  "project": "hello-splunk",
  "prompt_text": "OTel Collectorの設定をHEC exporterに変更して",
  "prompt_len": 28,
  "session_id": "a1b2c3d4-e5f6-7890",
  "host": "tanaka-macbook"
}
```

### フィールド定義

| フィールド | 型 | 説明 | 例 |
|-----------|-----|------|-----|
| `_time` | epoch | メッセージのタイムスタンプ | `1710335400` |
| `user` | string | メンバー識別子 | `tanaka` |
| `tool` | string | AIツール名 | `Claude Code` |
| `project` | string | プロジェクト名 | `hello-splunk` |
| `prompt_text` | string | プロンプト本文（500文字上限） | `OTel Collectorの設定を...` |
| `prompt_len` | int | プロンプトの文字数 | `28` |
| `session_id` | string | セッション識別子 | `a1b2c3d4-...` |
| `host` | string | 送信元ホスト名 | `tanaka-macbook` |

### Splunk 設定

```
# indexes.conf
[ai_prompt_logs]
homePath   = $SPLUNK_DB/ai_prompt_logs/db
coldPath   = $SPLUNK_DB/ai_prompt_logs/colddb
thawedPath = $SPLUNK_DB/ai_prompt_logs/thaweddb

# props.conf
[ai_prompt_review]
TIME_FORMAT = %s
SHOULD_LINEMERGE = false
KV_MODE = json
```

---

## 2. データパイプライン

### 2a. collect.py の拡張

既存の collect.py に `--emit-otel` オプションを追加。既存の JSON stdout 出力はそのまま残す。

```
python collect.py --emit-otel --user tanaka
```

動作:
1. 既存と同じくローカルログを収集
2. 収集したメッセージを OTel Log として OTLP エンドポイントに送信
3. `--user` で指定されたユーザー名をリソース属性に付与

必要な追加パッケージ:
- `opentelemetry-sdk`
- `opentelemetry-exporter-otlp`

### 2b. OTel Collector 設定

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 100

  # クレデンシャルのマスク処理（collect.pyで実施済みだが二重防御）
  attributes:
    actions:
      - key: prompt_text
        action: hash
        # 注: 本番ではredactionプロセッサの検討も

exporters:
  splunkhec:
    token: "${SPLUNK_HEC_TOKEN}"
    endpoint: "https://splunk.example.com:8088"
    source: "ai_prompt_review"
    sourcetype: "ai_prompt_review"
    index: "ai_prompt_logs"
    tls:
      insecure_skip_verify: false

service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [splunkhec]
```

### 2c. 運用方法（メンバー側）

各メンバーのPCで定期実行（cron / タスクスケジューラ）:

```bash
# 毎日1回、前日分のログを送信
0 9 * * * python collect.py --emit-otel --user $(whoami) --days 1
```

---

## 3. スキル設計（prompt-review-org）

### 前提: Splunk Enterprise MCP

Splunk Enterprise MCP サーバーが以下のツールを提供する想定:
- `splunk_search` — SPLクエリを実行して結果を返す

MCP が提供するツール名は実際のサーバーに合わせて調整する。

### SKILL.md 構成

```
.claude/skills/prompt-review-org/
├── SKILL.md                    # スキル定義
└── references/
    ├── spl-queries.md          # SPLクエリ集
    └── report-template-org.md  # 組織レポートテンプレート
```

### ステップ1: SPLクエリでデータ取得

collect.py は不要。代わりにSplunk MCP経由でSPLクエリを実行する。

**全体サマリー取得:**
```spl
index=ai_prompt_logs sourcetype=ai_prompt_review earliest=-7d
| stats count by user, tool, project
| sort -count
```

**特定ユーザーのプロンプト本文取得:**
```spl
index=ai_prompt_logs sourcetype=ai_prompt_review user="tanaka" earliest=-7d
| where prompt_len > 20
| table _time, tool, project, prompt_text
| sort _time
| head 200
```

**組織全体の定量指標:**
```spl
index=ai_prompt_logs sourcetype=ai_prompt_review earliest=-7d
| stats
    count AS total_messages,
    dc(user) AS active_users,
    dc(project) AS active_projects,
    avg(prompt_len) AS avg_prompt_length,
    values(tool) AS tools_used
  by user
| sort -total_messages
```

**ツール利用分布:**
```spl
index=ai_prompt_logs sourcetype=ai_prompt_review earliest=-7d
| stats count by user, tool
| chart sum(count) over user by tool
```

**時系列推移（日別）:**
```spl
index=ai_prompt_logs sourcetype=ai_prompt_review earliest=-30d
| timechart span=1d count by user
```

### ステップ2: 分析

取得したデータを既存スキルと同じ分析フレームワークで処理する。

**定量分析（SPLの集計結果から）:**
- ユーザー別アクティビティランキング
- ツール別利用分布
- プロンプト長の分布・推移
- プロジェクト別活動量

**定性分析（プロンプト本文から、既存ロジック流用）:**
- 技術理解度マップ（ユーザー別）
- プロンプティングパターン分析
- AI依存度分析
- 成長の軌跡

### ステップ3: レポート生成

`reports/prompt-review-org-YYYY-MM-DD.md` に出力。

---

## 4. 組織レポートテンプレート（追加セクション）

既存の個人レポートテンプレートに以下を追加:

### 組織サマリー（新規）

```markdown
## 1. 組織サマリー

- **分析期間**: YYYY-MM-DD 〜 YYYY-MM-DD
- **アクティブユーザー数**: N名
- **総メッセージ数**: N件
- **利用ツール**: Claude Code, Copilot Chat, ...

### ユーザー別アクティビティ

| ユーザー | メッセージ数 | 主要ツール | 主要プロジェクト | 平均プロンプト長 |
|---------|-------------|-----------|----------------|----------------|

### ツール利用分布

| ツール | 利用者数 | メッセージ数 | 割合 |
|--------|---------|-------------|------|
```

### ユーザー別分析（既存を拡張）

```markdown
## N. ユーザー別分析: [ユーザー名]

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

## 5. 引数設計

```
/prompt-review-org                    # 全ユーザー、過去7日分
/prompt-review-org 30                 # 過去30日分
/prompt-review-org tanaka             # 特定ユーザーのみ
/prompt-review-org tanaka 30          # 特定ユーザー × 過去30日分
```

---

## 6. 実装ステップ

1. **Splunk Enterprise 環境準備**
   - index `ai_prompt_logs` 作成
   - HEC トークン発行
   - sourcetype `ai_prompt_review` 設定

2. **データパイプライン構築**
   - collect.py に `--emit-otel` オプション追加
   - OTel Collector 設定・デプロイ
   - テストデータ送信・確認

3. **Splunk Enterprise MCP 追加**
   - `~/.claude.json` にMCPサーバー設定追加
   - SPLクエリ実行の動作確認

4. **スキル作成**
   - `prompt-review-org/SKILL.md` 作成
   - SPLクエリ集作成
   - 組織レポートテンプレート作成

5. **テスト・調整**
   - 実データでの動作確認
   - SPLクエリのチューニング
   - レポート品質の調整

---

## 7. プライバシー・セキュリティ考慮

社内利用前提だが、以下は検討が必要:

- **アクセス制御**: `ai_prompt_logs` indexへのアクセスをマネージャー/管理者に限定
- **シークレット検出**: collect.py の既存のシークレットスキャンはそのまま機能する（Splunkに送信前にマスク）
- **データ保持期間**: Splunk側のretention policyで管理
- **利用目的の周知**: メンバーにログ収集の目的・範囲を事前に説明
