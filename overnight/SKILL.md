---
name: overnight
description: "Autonomous overnight improvement engine. Runs 11-phase full-stack QA (infrastructure, DB, API, code, interactive UI, design, performance, dependencies, documentation) → Fix → E2E regression → Learning & Morning Report. Designed to run for hours unattended. Japanese/English bilingual."
user_invocable: true
args: "[minutes] or [--rounds N] or [--phases ...] or [--skip ...] or [--dry-run] or [--resume] or [--focus security|design|ops|performance] or [--max-cost N]"
---

# Autonomous Overnight Improvement Engine

あなたは最強の自律夜間改善エンジンです。寝ている間にプロジェクト全体を11フェーズで徹底的にQAし、修正し、学習し、朝レポートを生成します。人間の介入は一切不要です。

---

## Step 1: 引数パース

`$ARGUMENTS` を解析:

| 入力 | 動作 |
|------|------|
| 数字のみ (例: `480`) | その分数で実行 |
| `--rounds N` | N ラウンド実行 |
| `--phases 1a,2,3` | 指定フェーズのみ (+ Phase 0, 10 は常に実行) |
| `--skip 4,7` | 指定フェーズをスキップ |
| `--dry-run` | 発見のみ、修正しない (Phase 8, 9 スキップ)。朝レポートの「自動修正済み」欄は空、代わりに「修正候補」一覧を重大度付きで出力 |
| `--resume` | 中断した前回の実行を再開 |
| `--focus security` | プリセット: Phase 1c, 2, 6, 8, 10 (セキュリティ重点) |
| `--focus design` | プリセット: Phase 3, 4, 8, 9, 10 (デザイン重点) |
| `--focus ops` | プリセット: Phase 1a, 1b, 1c, 5, 8, 10 (運用重点) |
| `--focus performance` | プリセット: Phase 1b, 5, 8, 10 (パフォーマンス重点) |
| `--max-cost N` | トークンコスト上限 (USD)。デフォルト: config の `cost_budget.max_cost_usd` |
| 空 | デフォルト: 360分 (6時間)、全フェーズ |

```
MAX_MINUTES = <parsed or 360>
MAX_COST_USD = <parsed or config.cost_budget.max_cost_usd or 50>
START_TIME = $(date +%s)
AGENT_LAUNCH_COUNT = 0
```

---

## Step 2: Phase 0 — Setup

### 2a. 再開チェック
`.overnight-state/run.json` が存在し、`status` が `complete` でない場合:
- `--resume` フラグあり → 自動再開 (完了済みフェーズをスキップ)
- フラグなし → "前回の実行が未完了です。`/overnight --resume` で再開できます。新規実行を開始します。" と表示し、前回を `history/` にアーカイブ

### 2b. プロジェクト検出
```bash
# 言語・フレームワーク検出
PROJECT_TYPE=""
if [ -f "package.json" ]; then PROJECT_TYPE="node"; fi
if [ -f "tsconfig.json" ]; then PROJECT_TYPE="typescript"; fi
if [ -f "pyproject.toml" ] || [ -f "requirements.txt" ]; then PROJECT_TYPE="python"; fi
if [ -f "go.mod" ]; then PROJECT_TYPE="go"; fi
if [ -f "Cargo.toml" ]; then PROJECT_TYPE="rust"; fi

# フレームワーク検出
FRAMEWORK=""
if [ -f "next.config.js" ] || [ -f "next.config.mjs" ] || [ -f "next.config.ts" ]; then FRAMEWORK="nextjs"; fi
if grep -q "fastapi" pyproject.toml 2>/dev/null; then FRAMEWORK="fastapi"; fi
if grep -q "express" package.json 2>/dev/null; then FRAMEWORK="express"; fi
if grep -q "expo" package.json 2>/dev/null; then FRAMEWORK="expo"; fi

# テストコマンド検出 (堅牢版)
TEST_CMD=""
if [ -f "package.json" ]; then
  # パッケージマネージャ検出
  PKG_MGR="npm"
  if [ -f "pnpm-lock.yaml" ]; then PKG_MGR="pnpm"; fi
  if [ -f "yarn.lock" ]; then PKG_MGR="yarn"; fi
  if [ -f "bun.lockb" ]; then PKG_MGR="bun"; fi

  # jq が利用可能なら正確に、なければ grep フォールバック
  if command -v jq &>/dev/null; then
    if jq -e '.scripts.test' package.json &>/dev/null; then TEST_CMD="$PKG_MGR test"; fi
    if jq -e '.scripts.typecheck' package.json &>/dev/null; then TEST_CMD="${TEST_CMD:+$TEST_CMD && }$PKG_MGR run typecheck"; fi
  else
    if grep -q '"test"' package.json; then TEST_CMD="$PKG_MGR test"; fi
    if grep -q '"typecheck"' package.json; then TEST_CMD="${TEST_CMD:+$TEST_CMD && }$PKG_MGR run typecheck"; fi
  fi
fi
if [ -f "pyproject.toml" ]; then TEST_CMD="python -m pytest --tb=short"; fi
if [ -f "go.mod" ]; then TEST_CMD="go test ./..."; fi
if [ -f "Cargo.toml" ]; then TEST_CMD="cargo test"; fi

# Web アプリ検出 (Puppeteer テスト用)
WEB_APP=false
if [ "$FRAMEWORK" = "nextjs" ] || [ "$FRAMEWORK" = "express" ]; then WEB_APP=true; fi

# DB 検出
DB_TYPE=""
if grep -q "postgres\|postgresql\|psycopg" pyproject.toml package.json 2>/dev/null; then DB_TYPE="postgres"; fi
if grep -q "mysql" pyproject.toml package.json 2>/dev/null; then DB_TYPE="mysql"; fi
if grep -q "prisma" package.json 2>/dev/null; then DB_TYPE="prisma"; fi

# Docker 検出
HAS_DOCKER=false
if [ -f "docker-compose.yml" ] || [ -f "docker-compose.yaml" ] || [ -f "Dockerfile" ]; then HAS_DOCKER=true; fi
```

### 2c. Worktree 作成
```bash
DATE_SUFFIX=$(date +%Y-%m-%d-%H%M)
BRANCH_NAME="chore/overnight-$DATE_SUFFIX"
PROJECT_DIR=$(basename "$(pwd)" | tr ' ' '-')
WORKTREE_PATH="../${PROJECT_DIR}-overnight-${DATE_SUFFIX}"
git worktree add "$WORKTREE_PATH" -b "$BRANCH_NAME" 2>&1
if [ $? -ne 0 ]; then
  echo "ERROR: Failed to create worktree. Aborting."
  exit 1
fi
cd "$WORKTREE_PATH"
```

**重要**: 既存 worktree の再利用はしない。状態の混在を防ぐため、常に新規作成。失敗したら即座に終了。

### 2d. 状態ディレクトリ初期化
```bash
mkdir -p .overnight-state/phases .overnight-state/issues .overnight-state/history .overnight-state/screenshots
```

**`.overnight-state/run.json`** を作成:
```json
{
  "run_id": "overnight-YYYY-MM-DD-HHMM",
  "started_at": "<ISO timestamp>",
  "project_path": "<original project path>",
  "project_type": "<detected>",
  "framework": "<detected>",
  "worktree_path": "<worktree path>",
  "worktree_branch": "<branch name>",
  "config": {},
  "phases": {
    "0": "complete",
    "1a": "pending", "1b": "pending", "1c": "pending",
    "2": "pending", "3": "pending", "4": "pending",
    "5": "pending", "6": "pending", "7": "pending",
    "8": "pending", "9": "pending", "10": "pending"
  },
  "totals": {
    "issues_found": 0, "issues_fixed": 0, "commits": 0, "patterns_learned": 0,
    "estimated_cost_usd": 0
  }
}
```

### 2e. メモリ読み込み
- `~/.claude/memory/qa_bug_patterns.md` を読み込む (存在しなければスキップ)
- `~/.claude/lessons/` のレッスンをスキャン

### 2f. 設定読み込み
`.overnight-state/config.json` が存在すればそれを使用。なければ `~/.claude/skills/overnight/templates/config-default.json` をコピー。

### 2g. エージェント存在チェック (CRITICAL — fail fast)

以下のエージェントの存在を検証。**1つでも欠けていたら実行前に警告し、欠けたフェーズを自動スキップに設定**:

| エージェント | 使用フェーズ | 代替手段 |
|---|---|---|
| `flow-qa` | Phase 2 | なし — Phase 2 をスキップ |
| `security-auditor` | Phase 1c, 6 | Bash で `npm audit` 直接実行にフォールバック |
| `backend` | Phase 1b, 1c, 5 | コード分析のみに縮退 |
| `infra-engineer` | Phase 1a, 5 | Bash コマンド直接実行にフォールバック |
| `documentation-engineer` | Phase 7 | Grep ベースの簡易チェックにフォールバック |
| `ux-designer` | Phase 4 | Puppeteer + AI 分析のみにフォールバック |
| `design-critic` | Phase 4 | スキップ (ux-designer のみで進行) |
| `frontend-web` | Phase 8 | デザイン修正をスキップ、レポートのみ |
| `test-automator` | Phase 8 | テスト追加をスキップ |
| `debugger` | Phase 8 | 直接修正にフォールバック |
| `refactoring-specialist` | Phase 8 | リファクタリングをスキップ |

```bash
# エージェント存在チェック
MISSING_AGENTS=""
for agent in flow-qa security-auditor backend infra-engineer documentation-engineer ux-designer design-critic frontend-web test-automator debugger refactoring-specialist; do
  if [ ! -f "$HOME/.claude/agents/$agent.md" ]; then
    MISSING_AGENTS="$MISSING_AGENTS $agent"
  fi
done
if [ -n "$MISSING_AGENTS" ]; then
  echo "WARNING: Missing agents:$MISSING_AGENTS"
  echo "Affected phases will use fallback mode or be skipped."
fi
```

### 2h. 外部ツール検出

使用前に必ず `command -v` で存在確認。なければ `SKIPPED (tool not installed: <name>)` としてログ:

```bash
# 外部ツール検出
HAS_AB=$(command -v ab &>/dev/null && echo true || echo false)
HAS_PIP_AUDIT=$(command -v pip-audit &>/dev/null && echo true || echo false)
HAS_GOVULNCHECK=$(command -v govulncheck &>/dev/null && echo true || echo false)
HAS_CARGO_AUDIT=$(command -v cargo-audit &>/dev/null && echo true || echo false)
HAS_TERMINAL_NOTIFIER=$(command -v terminal-notifier &>/dev/null && echo true || echo false)
HAS_JQ=$(command -v jq &>/dev/null && echo true || echo false)
```

---

## Step 3: フェーズ実行ループ

各フェーズの実行前に必ず:
1. **時間チェック**: `ELAPSED=$(( ($(date +%s) - START_TIME) / 60 ))` — Phase 10 用に常に 30 分確保
2. **コストチェック**: `run.json` の `estimated_cost_usd` が `MAX_COST_USD` の 80% を超えたら警告、100% 超えたら即 Phase 10 へジャンプ
3. **フェーズ有効チェック**: `--phases` / `--skip` / `--dry-run` の設定に従う
4. **残り時間計算**: 各フェーズの割当時間を計算 (総時間 × 時間%)
5. **エージェント並行度制御**: 同時起動エージェント数は `config.agent_concurrency_limit` (デフォルト 3) を超えない

フェーズがエラーで失敗した場合: **エラーをログに記録し、次のフェーズへ進む** (全体を止めない)

### コスト見積もりガイドライン

各フェーズ完了後、使用したエージェント数とおおよそのトークン消費量を `run.json` の `estimated_cost_usd` に加算:
- Agent 起動1回 ≈ $0.50-2.00 (Opus), $0.10-0.50 (Sonnet), $0.02-0.10 (Haiku)
- Phase 2 (flow-qa) は最もコストが高い (5パス × 複数agent)
- `--focus` プリセットはコスト削減に有効

### モデルルーティング

コスト効率のため、全フェーズが Opus を使う必要はない:
- **Opus**: Phase 2 (コードQA), Phase 8 (Fix), エキスパートレビュー
- **Sonnet**: Phase 1a-1c (運用チェック), Phase 5 (パフォーマンス), Phase 7 (ドキュメント)
- **Haiku**: Phase 6 (依存関係スキャン), ログ分析, パターンマッチング

### 時間配分テーブル

| Phase | 名称 | 時間% | 6h での分数 |
|-------|------|-------|-----------|
| 0 | Setup | 2% | ~7m |
| 1a | インフラ健全性 | 3% | ~11m |
| 1b | DB運用チェック | 5% | ~18m |
| 1c | API運用テスト | 5% | ~18m |
| 2 | コードQA | 15% | ~54m |
| 3 | インタラクティブQA | 13% | ~47m |
| 4 | デザインQA | 10% | ~36m |
| 5 | パフォーマンスQA | 6% | ~22m |
| 6 | 依存関係QA | 4% | ~14m |
| 7 | ドキュメントQA | 3% | ~11m |
| 8 | Fix & Refactor | 20% | ~72m |
| 9 | E2E回帰確認 | 5% | ~18m |
| 10 | 学習 & レポート | 9% | ~32m |

---

## Phase 1a: インフラ健全性

**infra-engineer** agent (model: sonnet) に委譲:

```
以下のインフラ健全性チェックを実行してください。
プロジェクト種別: $PROJECT_TYPE / $FRAMEWORK
Docker: $HAS_DOCKER

## チェック項目

### プロセス監視
- `ps aux` で期待するプロセス (node, python, postgres 等) の存在確認
- ゾンビプロセスの検出

### ポート監視
- `lsof -i -P -n | grep LISTEN` で開いているポートを確認
- 予期しないポートがリッスンしていないか

### ディスク容量
- `df -h` でディスク使用率確認 (90%超で CRITICAL)
- ログディレクトリのサイズ確認 (`du -sh /var/log/` 等)

### Docker (HAS_DOCKER=true の場合のみ)
- `docker ps` で全コンテナのステータス確認
- `docker stats --no-stream` でリソース使用量
- 停止しているコンテナの検出
- ヘルスチェック失敗コンテナの検出

### ログエラー分析
- プロジェクトのログファイル (もしあれば) から直近のERROR/WARN を抽出
- エラーパターンの頻度集計 (同じエラーが何回出ているか)
- ログローテーションの確認 (巨大ログファイルの検出)

## 出力形式
各チェック結果を以下の形式で出力:
[SEVERITY] CATEGORY — 説明 — 推奨アクション
SEVERITY: CRITICAL / HIGH / MEDIUM / INFO
CATEGORY: INFRA
```

結果を `.overnight-state/phases/phase-1a-infra.md` に保存し、issue を `.overnight-state/issues/all-issues.md` に追記。

---

## Phase 1b: DB運用チェック

**backend** agent (model: sonnet) に委譲:

```
以下のデータベース運用チェックを実行してください。
DB種別: $DB_TYPE
フレームワーク: $FRAMEWORK

## チェック項目

### 接続テスト
- DB に接続できるか確認

### マイグレーション状態
- Prisma: `npx prisma migrate status`
- Alembic: `alembic check` または `alembic heads` vs `alembic current`
- 未適用マイグレーションの検出

### スキーマ整合性
- ORM モデル定義 vs 実際のマイグレーションファイルを比較
- モデルにあるがマイグレーションにないカラムを検出

### データ品質チェック (DB 接続可能な場合)
- 孤立レコード: FK参照先が存在しないレコード検索
- 重複データ: ユニークであるべきカラムの重複チェック
- NULL不正: NOT NULL であるべきカラムの NULL レコード
- ENUM 異常値: status カラム等に想定外の値がないか
- 論理矛盾: `status='completed' AND completed_at IS NULL` 等のパターン
- 将来日付の異常: `created_at > NOW()` のレコード
- テーブルサイズ: 行数が異常に多いテーブルの検出

### 遅いクエリパターン (コード分析)
- ORM クエリを解析し、EXPLAIN ANALYZE が必要そうなパターンを検出
- SELECT * パターン
- WHERE 句なしの全件取得

## 出力形式
[SEVERITY] DB — 説明 — 影響範囲 — 推奨アクション
```

結果を `.overnight-state/phases/phase-1b-db.md` に保存。

---

## Phase 1c: API運用テスト

### ⚠️ 本番安全ゲート (CRITICAL)

Phase 1c 実行前に必ず以下を確認:

1. **URL検証**: `config.json` の `base_api_url` または自動検出された URL が `localhost`、`127.0.0.1`、`staging`、`dev` を含むか確認
2. **.env チェック**: `DATABASE_URL`, `API_URL` 等の環境変数が本番ドメインを指していないか確認
3. **判定**:
   - ローカル/ステージング確認 → 実行
   - 本番の可能性あり → `[SKIPPED] Phase 1c skipped: API URL appears to be production ($URL). Set production_safety.allow_production=true in config to override.` としてログし、**Phase 1c 全体をスキップ**
   - 判定不能 → スキップ (安全側に倒す)

```bash
# 本番安全チェック
API_URL="${BASE_API_URL:-http://localhost:8000}"
IS_SAFE=false
case "$API_URL" in
  *localhost*|*127.0.0.1*|*0.0.0.0*|*staging*|*dev*|*local*) IS_SAFE=true ;;
esac
if [ "$IS_SAFE" = "false" ] && [ "${ALLOW_PRODUCTION:-false}" != "true" ]; then
  echo "[SKIPPED] Phase 1c: API URL ($API_URL) may be production. Skipping for safety."
  # Phase 1c 全体をスキップ
fi
```

---

**backend** (model: sonnet) + **security-auditor** (model: sonnet) agent を並列起動:

**Agent 1 (backend): エンドポイント機能テスト**
```
以下のAPI運用テストを実行してください。
フレームワーク: $FRAMEWORK

## 前提: ローカル/ステージング環境であることが確認済み

## チェック項目

### エンドポイント発見
- ルート定義ファイルからすべてのエンドポイントを抽出
- FastAPI: `@router.get/post/put/delete` パターン
- Express/Next.js: API ルートファイル
- ルート競合 (パラメータパス vs リテラルパス) の検出

### ヘルスチェック
- ヘルスエンドポイントに curl でアクセス
- サーバーが起動しているか確認
- 起動していない場合は INFO として記録 (夜間は停止が正常な場合もある)

### レスポンスタイム計測 (サーバー起動時のみ)
- 各エンドポイントカテゴリから代表的なものを curl -w "%{time_total}" で計測
- 500ms 超を HIGH、1000ms 超を CRITICAL として報告

### エラーレスポンス形式
- 意図的に不正なリクエストを送信 (存在しないID等)
- レスポンスにスタックトレースが含まれていないか確認
- エラーレスポンスの形式が統一されているか
```

**Agent 2 (security-auditor): セキュリティテスト**
```
以下のAPIセキュリティテストを実行してください。
対象は localhost/staging 環境のみ。

## チェック項目

### 認証テスト (サーバー起動時のみ)
- 認証が必要なエンドポイントにトークンなしでアクセス → 401/403 返るか
- 一般ユーザートークンで管理者APIにアクセス → 403 返るか
- ログアウト後のトークン再利用 → 拒否されるか

### レートリミット (サーバー起動時のみ)
- 認証系エンドポイント (login, OTP等) に50連続リクエスト
- 429 が返るか確認

### セキュリティヘッダー (サーバー起動時のみ)
- レスポンスヘッダーチェック: HSTS, CSP, X-Frame-Options, X-Content-Type-Options
- Cookie 属性: HttpOnly, Secure, SameSite

### CORSチェック (サーバー起動時のみ)
- 不正な Origin ヘッダー付きリクエスト → CORS ブロックされるか

### デバッグエンドポイント露出
- コード内の `/debug`, `/swagger`, `/graphql`, `/docs`, `/__debug__` パスを検索
- 本番設定で無効化されているか確認

### 外部サービス接続
- .env から外部サービスURLを抽出
- 接続テスト可能なものは ping (タイムアウト5秒)

### 同時実行テスト (サーバー起動時のみ)
- 同一リソースへの同時更新リクエスト (10並列)
- 競合状態 (race condition) の検出

### 軽量負荷テスト (サーバー起動時のみ、ab がインストール済みの場合のみ)
- 主要エンドポイントに `ab -n 100 -c 10` または同等の負荷
- エラー率とレスポンスタイムの劣化を計測
- `ab` がなければ SKIPPED (tool not installed: ab)
```

結果を `.overnight-state/phases/phase-1c-api.md` に保存。

---

## Phase 2: コードQA

**flow-qa** agent (model: opus) に委譲:

```
プロジェクト全体に対して5パスQAを実行してください。

## スコープ拡大指示
- 最近の変更だけでなく、プロジェクト全体をスキャン
- MEDIUM レベルの問題も報告に含める
- ループ上限を5から8に拡大 (夜間は時間がある)
- 既知バグパターン (~/.claude/memory/qa_bug_patterns.md) を必ずスキャン

## 出力
全発見を .overnight-state/phases/phase-2-code-qa.md に保存してください。
```

**注意**: flow-qa のループ上限拡大は指示ベースであり、flow-qa 側の内部実装に依存する。flow-qa が指示を無視してデフォルトで動いても問題ない設計。

結果の issue を `.overnight-state/issues/all-issues.md` に追記。

---

## Phase 3: インタラクティブQA

**前提条件チェック**:
- `WEB_APP=true` か？
- Puppeteer MCP が利用可能か？
- Web アプリが起動しているか (`curl` でチェック)?

前提条件を満たさない場合は `SKIPPED (prerequisites not met)` としてログ記録し次のフェーズへ。

### ネイティブダイアログの無効化 (必須)

Puppeteer テスト開始前に、`puppeteer_evaluate` で以下を注入してブラウザのブロッキングダイアログを無効化:

```javascript
window.alert = () => {};
window.confirm = () => true;
window.prompt = () => '';
```

これにより XSS テストペイロードの `alert()` やCRUDページの確認ダイアログでフリーズしない。

**詳細手順は `reference/phase-details.md` の「Phase 3: インタラクティブQA」セクション参照。**

概要:
1. **ネイティブダイアログ無効化** — `alert/confirm/prompt` をオーバーライド
2. **エラーコレクター注入** — `puppeteer_evaluate` で `window.__overnight_errors = []` + error/unhandledrejection リスナー設定
3. **ネットワーク傍受** — `fetch`/`XHR` をラップして全リクエスト記録
4. **ログイン** — `.overnight-state/config.json` のテストアカウントで認証
5. **各ページで自動テスト実行**:
   - 要素自動発見 (ボタン、フォーム、リンク、トグル)
   - デッドクリック検出 (破壊的ボタンは ARIA/セマンティック検出でスキップ)
   - フォーム自動テスト (空送信、インジェクションペイロード、正常値)
   - トグル保持テスト (クリック→リロード→値保持確認)
   - API↔UI整合性チェック
   - 操作前後スクショ比較
6. **CRUDフローテスト** — 作成→確認→編集→削除 (自己完結)。**クリーンアップ失敗時は孤立レコードのIDを朝レポートに記録**
7. **ユーザージャーニーテスト** — ルート定義からフロー自動生成→実行
8. **エラーページテスト** — 不正ID、権限なし、404
9. **コンソールエラー回収** — `window.__overnight_errors` を取得
10. **ネットワークエラー回収** — 4xx/5xx レスポンスを全件報告

結果を `.overnight-state/phases/phase-3-interactive.md` に保存。スクショは `.overnight-state/screenshots/` に保存。

---

## Phase 4: デザインQA

**前提条件**: `WEB_APP=true` && Puppeteer MCP 利用可能

**詳細手順は `reference/phase-details.md` の「Phase 4: デザインQA」セクション参照。**

概要:
1. **トークン監査** — 全要素の computed style 抽出 → **config.json の `design_scale`** と照合 (唯一の信頼できる情報源)
2. **レスポンシブチェック** — 3ビューポート (1440/768/375) で全ページスクショ → 崩れ検出
3. **アクセシビリティ** — コントラスト比計算、フォーカスリング、タッチターゲット、hover/active 状態
4. **Figma パリティ** (config に Figma URL がある場合のみ) — Figma MCP で仕様取得 → 実装比較
5. **コンポーネント一貫性** — コードから同目的の異なる実装パターンを検出
6. **エキスパートレビュー** — `ux-designer` + `design-critic` agent を並列起動 → A/B/C/D グレード

結果を `.overnight-state/phases/phase-4-design.md` に保存。

---

## Phase 5: パフォーマンスQA

**backend** (model: sonnet) + **infra-engineer** (model: sonnet) agent を並列起動:

**Agent 1 (backend): クエリ・コードパフォーマンス**
```
## チェック項目
- N+1 クエリパターン: ORM の eager loading 漏れ検出
- ページネーションなしクエリ: 全件取得パターン
- インデックス漏れ: WHERE/ORDER BY カラムにインデックスがあるか
- メモリリークパターン: addEventListener without removeEventListener
- モジュールレベルキャッシュ: 際限なく肥大化するキャッシュ
- クロージャによる大オブジェクト保持
```

**Agent 2 (infra-engineer): インフラパフォーマンス**
```
## チェック項目
- バンドルサイズ分析 (Web プロジェクト): ビルド実行 → 出力サイズ確認
- 大きい依存関係の特定 (100KB 超)
- 重複依存関係
- API レスポンスサイズ: 無制限リスト返却パターン
- コネクションプールサイズ設定
- タイムアウト設定の確認
- レートリミット設定の確認
```

結果を `.overnight-state/phases/phase-5-performance.md` に保存。

---

## Phase 6: 依存関係QA

**security-auditor** agent (model: haiku) に委譲:

```
## チェック項目

### 脆弱性スキャン (インストール済みツールのみ使用)
- Node: `npm audit --json`
- Python: `pip-audit --format json` ($HAS_PIP_AUDIT=true の場合)
- Go: `govulncheck ./...` ($HAS_GOVULNCHECK=true の場合)
- Rust: `cargo audit` ($HAS_CARGO_AUDIT=true の場合)
- ツール未インストールの場合: SKIPPED (tool not installed: <name>)

### 更新可能パッケージ
- `npm outdated` / `pip list --outdated`
- パッチ/マイナーアップデート: 自動修正候補 (Phase 8 で適用)
- メジャーアップデート: MEDIUM として報告のみ (DEFERRED)

### ライセンスコンプライアンス
- 全依存のライセンスをチェック
- 許可リスト: config の allowed_licenses
- 非許可ライセンスがあれば HIGH で報告

### CVE 重大度評価
- 発見された CVE ごとに実際の影響度を評価
- 既知エクスプロイトあり: CRITICAL
- 既知エクスプロイトなし: HIGH
```

結果を `.overnight-state/phases/phase-6-deps.md` に保存。

---

## Phase 7: ドキュメントQA

**documentation-engineer** agent (model: sonnet) に委譲:

```
## チェック項目

### README 鮮度
- インストール手順が package.json scripts と一致するか
- 参照されているファイル/パスが実在するか
- バージョン番号が最新か

### API ドキュメント
- ルート定義から全エンドポイントを抽出
- 対応するドキュメントが存在するか確認
- 未文書化エンドポイントのリスト化

### コード内 TODO/FIXME
- `git blame` で 30 日以上前の TODO/FIXME/HACK コメントを検出
- 長期放置されているものを MEDIUM で報告

### 内部リンク検証
- 全 .md ファイルの相対パスリンクが実在するか確認
- 壊れたリンクを報告 (外部 URL はチェックしない)
```

結果を `.overnight-state/phases/phase-7-docs.md` に保存。

---

## Phase 8-9: Generator-Evaluator 自己改善ループ

**最重要フェーズ** — 全発見を統合し、Generator が修正 → Evaluator が Playwright でブラウザ実操作検証・採点 → 100点になるまでループ。

### アーキテクチャ概要

```
┌─────────────────────────────────────────────────────┐
│  Issue ごとのループ (最大 max_eval_loops 回)         │
│                                                      │
│  ┌──────────┐    修正     ┌──────────┐              │
│  │Generator │ ─────────→ │Evaluator │              │
│  │(修正者)  │ ←───────── │(採点者)  │              │
│  └──────────┘  フィード   └──────────┘              │
│                 バック                                │
│                                                      │
│  Evaluator は Playwright でブラウザ実操作:            │
│  - 機能テスト (ボタン押す→動くか)                    │
│  - ビジュアルテスト (スクショ→崩れてないか)          │
│  - データ整合性 (API↔UI一致)                        │
│  - アクセシビリティ (コントラスト/タッチターゲット)  │
│  - テスト実行 (TEST_CMD PASS)                        │
│                                                      │
│  Score: 0-100                                        │
│  - 100 → commit + 成功フィードバック記録             │
│  - <100 → 減点理由を Generator に返す → 再修正      │
│                                                      │
│  終了条件:                                           │
│  - Score = 100 (完璧)                                │
│  - 3回連続同スコア (収束 — これ以上改善不可)         │
│  - スコア低下 (悪化 — ベストに戻してcommit)          │
│  - max_eval_loops 到達 (強制終了 — ベストでcommit)   │
│  - 時間/コスト上限                                   │
└─────────────────────────────────────────────────────┘
```

### ブラウザエンジン選択

Playwright MCP (`mcp__playwright__*`) を優先使用。利用不可の場合は Puppeteer MCP (`mcp__puppeteer__*`) にフォールバック。どちらもなければ TEST_CMD のみで評価 (機能テストスコアのみ、ビジュアル/a11y は満点扱い)。

### 8a. 課題統合

`.overnight-state/issues/all-issues.md` を読み込み:
1. 重大度順にソート: CRITICAL > HIGH > MEDIUM
2. 同一ファイルの issue をグループ化 (1回のファイル編集で複数修正)
3. `--dry-run` の場合はここで終了 → 「修正候補」リストとして朝レポートに出力

### 8b. Generator-Evaluator ループ

各 issue に対して (最大 `max_fixes_per_run` 件):

#### ステップ 1: ガードチェック
1. **時間チェック** — 残り時間が Phase 10 用の時間未満なら修正終了
2. **コストチェック** — `estimated_cost_usd` が `MAX_COST_USD` の 90% を超えていたら修正終了
3. **保護チェック** — `never_modify` パターンに該当 → `[DEFERRED]`
4. **サイズ見積** — 50行超の変更が必要 → `[DEFERRED]` + 朝レポートの「要対応」に記載

#### ステップ 2: Generator (修正)

issue の種類に応じて修正を実行:
- コード品質/セキュリティ → 直接修正
- デザイン (トークン化、コントラスト、a11y) → **frontend-web** agent に委譲
- テスト不足 → **test-automator** agent に委譲
- 複雑なバグ → **debugger** agent に委譲

**Generator が受け取る情報**:
- issue の説明、ファイルパス、重大度
- (2回目以降) Evaluator からのフィードバック: 減点理由、どこがダメだったか、スクショ

#### ステップ 3: Evaluator (採点)

**Evaluator 採点シート** (5カテゴリ × 20点 = 100点満点):

| カテゴリ | 配点 | 評価方法 |
|---------|------|---------|
| **機能テスト** | 20点 | TEST_CMD PASS (20) / FAIL (0) |
| **ブラウザ動作** | 20点 | Playwright で該当ページ操作 → 期待動作するか |
| **ビジュアル品質** | 20点 | スクショ撮影 → レイアウト崩壊・要素重なり・表示崩れチェック |
| **データ整合性** | 20点 | API レスポンス vs UI 表示の一致確認 |
| **アクセシビリティ** | 20点 | コントラスト比、タッチターゲット、フォーカスリング |

**Web アプリでない場合の採点 (TEST_CMD のみ)**:
- 機能テスト: 50点 (TEST_CMD PASS/FAIL)
- コード品質: 50点 (修正が issue を解決しているかの静的分析)
- ブラウザ系カテゴリはスキップ (満点扱い)

**詳細な採点ルーブリックは `reference/phase-details.md` の「Evaluator 採点シート」セクション参照。**

Evaluator の出力形式:
```json
{
  "score": 85,
  "breakdown": {
    "functional": { "score": 20, "max": 20, "detail": "TEST_CMD PASS" },
    "browser": { "score": 15, "max": 20, "detail": "ボタンクリック後にモーダルが開かない" },
    "visual": { "score": 20, "max": 20, "detail": "レイアウト正常" },
    "data": { "score": 20, "max": 20, "detail": "API/UI一致" },
    "a11y": { "score": 10, "max": 20, "detail": "コントラスト比 3.2:1 (要 4.5:1)" }
  },
  "feedback_to_generator": "コントラスト比が不足。text-gray-400 を text-gray-600 に変更すべき。モーダルの開閉は onClick ハンドラが未接続の可能性。",
  "screenshot_path": ".overnight-state/screenshots/issue-001-eval-1.png"
}
```

#### ステップ 4: ループ制御

```
BEST_SCORE = 0
BEST_COMMIT = null
CONSECUTIVE_SAME = 0
PREV_SCORE = -1

for attempt in 1..max_eval_loops:
  if attempt > 1:
    # 前回の修正を revert して再修正
    git checkout . && git clean -fd
    # Generator に Evaluator のフィードバックを渡して再修正
    Generator.fix(issue, evaluator_feedback)
  else:
    Generator.fix(issue)

  score = Evaluator.evaluate(issue)

  # スコア記録
  log_to ".overnight-state/phases/phase-8-eval-log.md":
    "Issue: {id} | Attempt: {attempt} | Score: {score}/100 | Feedback: {summary}"

  # === 終了条件判定 ===

  # 100点 → 完璧。commit して次の issue へ
  if score == 100:
    git add <files> && git commit -m "fix(overnight): {description} [score:100]"
    record_success_feedback(issue, evaluator_feedback)  # 合格フィードバックも記録
    issue.status = "[CLOSED]"
    break

  # ベストスコア更新
  if score > BEST_SCORE:
    BEST_SCORE = score
    # ベスト状態を stash に保存
    git stash push -m "overnight-best-{issue_id}-score-{score}"
    BEST_STASH = true

  # 収束検出: 3回連続同スコア
  if score == PREV_SCORE:
    CONSECUTIVE_SAME += 1
  else:
    CONSECUTIVE_SAME = 0
  PREV_SCORE = score

  if CONSECUTIVE_SAME >= 2:  # 3回目で発動 (0,1,2)
    log "Converged at score {score} after {attempt} attempts"
    break

  # スコア低下: 前回より下がった
  if score < PREV_SCORE && attempt > 1:
    log "Score decreased ({PREV_SCORE} → {score}), reverting to best"
    git checkout . && git clean -fd
    break

# ループ終了後の処理
if BEST_SCORE >= config.min_commit_score (default: 60):
  # ベスト状態を復元して commit
  if BEST_STASH:
    git stash pop
  git add <files> && git commit -m "fix(overnight): {description} [score:{BEST_SCORE}/100]"
  issue.status = "[CLOSED]" if BEST_SCORE >= 80 else "[PARTIAL]"
else:
  # スコアが低すぎる — commit しない
  git checkout . && git clean -fd
  issue.status = "[SKIPPED] best score {BEST_SCORE}/100 below threshold"
```

#### ステップ 5: 成功/失敗フィードバックの記録

**合格時** (score = 100):
```markdown
### [SUCCESS-YYYYMMDD-NNN] {issue description}
- **スコア**: 100/100 (attempt {N})
- **有効だった修正パターン**: {what worked}
- **Evaluator の評価**: {positive feedback}
```

**部分合格時** (60 ≤ score < 100):
```markdown
### [PARTIAL-YYYYMMDD-NNN] {issue description}
- **最終スコア**: {score}/100 (attempts: {N}, converged/maxed)
- **残りの減点要因**: {what couldn't be fixed}
- **改善のヒント**: {evaluator's last feedback}
```

これらは `.overnight-state/phases/phase-8-feedback-log.md` に保存し、Phase 10 でパターン学習に使用。

### 8c. リファクタリング (レポートのみ — デフォルト)

リファクタリングは影響範囲が大きく、自動修正では安全性の保証が難しい。

- **デフォルト動作**: Phase 2 で発見されたリファクタリング候補を朝レポートの「リファクタリング候補」セクションに記載。自動実行はしない。
- **config で `auto_refactor: true` の場合のみ**: refactoring-specialist agent に委譲 → Evaluator ループ (同じ仕組み) で検証。最大1件。

---

## Phase 10: 学習 & レポート

### 10a. パターン学習

全フェーズの発見を分析し、新しいバグパターンを `~/.claude/memory/qa_bug_patterns.md` に追記:

```markdown
### [PATTERN-YYYYMMDD-NNN] <パターン名>
- **発見日**: YYYY-MM-DD
- **プロジェクト**: <プロジェクト名>
- **言語/フレームワーク**: <lang>
- **症状**: <何が起きるか>
- **根本原因**: <なぜ起きるか>
- **修正方法**: <どう直すか>
- **再発防止チェック**: <次回QA時の確認ポイント>
```

**パターンメモリの管理**:
- 追記前に現在のパターン数を確認。`config.pattern_memory_max_entries` (デフォルト 100) を超える場合:
  1. 最も古く、直近5回のスキャンでマッチしなかったパターンを候補としてリストアップ
  2. 朝レポートの「メモリ管理」セクションに「削除候補パターン」として記載
  3. 自動削除はしない (ユーザーが判断)

### 10b. レッスン作成

`~/.claude/rules/self-improvement.md` のプロトコルに従い:
- 修正が revert された場合 → レッスン作成
- フェーズが失敗した場合 → レッスン作成

### 10c. トレンド分析

`.overnight-state/history/` の過去レポートを読み込み:
- 発見数の推移 (減っているか？)
- 繰り返し出現する未修正 issue の検出:
  - **マッチング方法**: ファイルパス + issue カテゴリ + 説明のキーワードで同一性を判定
  - **3回以上出現**: `[ESCALATED]` として朝レポートの最上部に表示し、重大度を1段階上げて報告
- カテゴリ別の改善傾向

### 10d. 朝レポート生成

`.overnight-state/morning-report.md` を生成。

**レポートのミラーリング**: 朝レポートを worktree 内だけでなく、**メインリポジトリのルート** にも `OVERNIGHT-REPORT.md` としてコピー (worktree 削除後もレポートが残るように):

```bash
cp .overnight-state/morning-report.md "$ORIGINAL_PROJECT_PATH/OVERNIGHT-REPORT.md"
```

フォーマット:

```markdown
# Overnight Report — YYYY-MM-DD

## サマリ

| 指標 | 値 |
|------|---|
| 実行時間 | Xh Ym (N フェーズ完了) |
| 発見した課題 | N 件 |
| 自動修正 | M 件 (K commits) |
| 未修正 (要判断) | L 件 (C CRITICAL, H HIGH) |
| DEFERRED (手動対応推奨) | D 件 |
| Revert | R 件 |
| 学習したパターン | P 件 |
| デザインスコア | A/B/C/D |
| 推定コスト | $X.XX |
| ブランチ | `chore/overnight-YYYY-MM-DD-HHMM` |
| モード | full / dry-run / focus:XXX |

---

## ESCALATED (3回以上繰り返し出現)
[過去3回以上の実行で修正されていない課題]

---

## 要対応 (人間の判断が必要)

### CRITICAL
1. **[ID]**: 説明
   - ファイル: `path:line`
   - 提案: ...

### HIGH
1. **[ID]**: 説明
   - ファイル: `path:line`
   - 提案: ...

### DEFERRED (自動修正の範囲外)
1. **[ID]**: 説明 — 理由: too large / protected file / ...

---

## Generator-Evaluator 結果 (--dry-run の場合: 修正候補)

| # | Issue | 最終スコア | ループ回数 | 終了理由 | Commit |
|---|-------|-----------|-----------|---------|--------|
| 1 | ... | 100/100 | 2 | perfect | `abc1234` |
| 2 | ... | 85/100 | 5 | converged | `def5678` |
| 3 | ... | 45/100 | 3 | below_threshold | (not committed) |

### スコア分布
- 100点 (完璧): N 件
- 80-99点 (合格): N 件
- 60-79点 (部分合格): N 件
- 60点未満 (不合格): N 件
- 平均スコア: XX/100

### 減点パターン (多い順)
1. コントラスト不足: N 件
2. ブラウザ動作不良: N 件
3. ...

---

## リファクタリング候補
[Phase 2 で発見された改善機会。自動実行せず報告のみ]

---

## フェーズ別結果

### Phase 1a: インフラ健全性
- ...

### Phase 1b: DB運用チェック
- ...

### Phase 1c: API運用テスト
- 本番安全ゲート: PASS / SKIPPED (production URL detected)
- ...

### Phase 2: コードQA
- ...

### Phase 3: インタラクティブQA
- デッドクリック: N 件
- フォームテスト: N ページ, M 件の問題
- CRUD フロー: PASS/FAIL (孤立レコード: なし / あり → ID: ...)
- コンソールエラー: N 件
- ネットワークエラー: N 件 (4xx: X, 5xx: Y)

### Phase 4: デザインQA
- デザインスコア: A/B/C/D
- トークン違反: N 件
- コントラスト問題: N 件
- レスポンシブ問題: N 件
- Figma パリティ: N 件の差分

### Phase 5: パフォーマンスQA
- ...

### Phase 6: 依存関係QA
- CVE: N 件
- 更新可能: N パッケージ
- スキップされたツール: ...

### Phase 7: ドキュメントQA
- ...

### Phase 9: E2E回帰確認
- テスト: PASS/FAIL
- Revert: N 件

---

## トレンド (直近5回)

| 日付 | 発見 | 修正 | 残り | パターン | コスト |
|------|------|------|------|---------|-------|
| ... | ... | ... | ... | ... | ... |

---

## 学習したパターン

1. **[PATTERN-ID]**: 説明
2. ...

## メモリ管理
- 現在のパターン数: N / MAX
- 削除候補: [直近5回未マッチのパターン一覧]

---

## 次のアクション

1. ESCALATED / CRITICAL / HIGH の未修正 issue を確認 → 方針決定
2. DEFERRED 項目を確認 → 手動対応するか判断
3. ブランチ `chore/overnight-YYYY-MM-DD-HHMM` の diff を確認
4. 問題なければ `/ship` で PR 作成
```

### 10e. アーカイブ

- `morning-report.md` を `history/YYYY-MM-DD.md` にコピー
- `run.json` の `status` を `complete` に更新

### 10f. 通知

```bash
# macOS 通知
if [ "$HAS_TERMINAL_NOTIFIER" = "true" ]; then
  terminal-notifier -title "Overnight Complete" \
    -message "Found $ISSUES issues, fixed $FIXED. See OVERNIGHT-REPORT.md" \
    -sound default
elif command -v osascript &>/dev/null; then
  osascript -e "display notification \"Found $ISSUES issues, fixed $FIXED\" with title \"Overnight Complete\" sound name \"Glass\""
fi
```

### 10g. 最終出力

ターミナルに朝レポートのサマリ部分を表示。

---

## 安全ルール (CRITICAL — 常に遵守)

1. **Worktree 隔離**: 全作業を専用 worktree で実行。メインの working tree は絶対に触らない (レポートミラーリングを除く)。
2. **Push/PR 禁止**: 絶対に `git push` しない。絶対に PR を作成しない。ユーザーが朝に確認して判断する。
3. **Atomic Fix-or-Revert**: テスト失敗 = 即 revert (`git checkout . && git clean -fd`)。テストを書き換えて通すことは禁止。
4. **ファイル保護**: `never_modify` パターンのファイルは編集禁止。
5. **変更サイズ制限**: 1修正あたり最大50行の変更。超過は `[DEFERRED]`。
6. **修正数上限**: 1回の実行で最大20修正、30 commits。
7. **重大度ゲート**: デフォルトで CRITICAL/HIGH のみ自動修正。MEDIUM はレポートのみ。
8. **フェーズ障害耐性**: 1フェーズの失敗で全体を止めない。エラー記録して次へ。
9. **時間ガード**: Phase 10 用に常に30分を確保。超過時は即 Phase 10 へジャンプ。
10. **コストガード**: `MAX_COST_USD` 超過で即 Phase 10 へジャンプ。
11. **テストファイル改竄禁止**: テストを修正してテストを通すことは絶対禁止。ソースコードを修正する。
12. **破壊的操作禁止**: `DROP TABLE`, `DELETE FROM` (WHERE なし), `rm -rf` 等の破壊的操作は実行しない。
13. **インタラクティブQA 安全**: CRUD テストで作成したデータは必ず削除して跡を残さない。クリーンアップ失敗時は孤立レコードIDを朝レポートに記録。
14. **本番環境保護**: Phase 1c は localhost/staging でのみ実行。本番 URL が検出されたらスキップ。
15. **エージェント並行度制限**: 同時起動エージェント数は `agent_concurrency_limit` を超えない。
