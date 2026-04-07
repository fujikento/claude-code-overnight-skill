---
name: overnight
description: "Autonomous overnight improvement engine. Runs 11-phase full-stack QA (infrastructure, DB, API, code, interactive UI, design, performance, dependencies, documentation) → Fix → E2E regression → Learning & Morning Report. Designed to run for hours unattended."
user_invocable: true
args: "[minutes] or [--rounds N] or [--phases ...] or [--skip ...] or [--dry-run] or [--resume] or [--focus security|design|ops|performance]"
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
| `--dry-run` | 発見のみ、修正しない (Phase 8, 9 スキップ) |
| `--resume` | 中断した前回の実行を再開 |
| `--focus security` | プリセット: Phase 1c, 2, 6, 8, 10 (セキュリティ重点) |
| `--focus design` | プリセット: Phase 3, 4, 8, 9, 10 (デザイン重点) |
| `--focus ops` | プリセット: Phase 1a, 1b, 1c, 5, 8, 10 (運用重点) |
| `--focus performance` | プリセット: Phase 1b, 5, 8, 10 (パフォーマンス重点) |
| 空 | デフォルト: 360分 (6時間)、全フェーズ |

```
MAX_MINUTES = <parsed or 360>
START_TIME = $(date +%s)
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

# テストコマンド検出
TEST_CMD=""
if [ -f "package.json" ]; then
  if grep -q '"test"' package.json; then TEST_CMD="npm test"; fi
  if grep -q '"typecheck"' package.json; then TEST_CMD="$TEST_CMD && npm run typecheck"; fi
fi
if [ -f "pyproject.toml" ]; then TEST_CMD="python -m pytest --tb=short"; fi
if [ -f "go.mod" ]; then TEST_CMD="go test ./..."; fi

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
BRANCH_NAME="chore/overnight-$(date +%Y-%m-%d)"
WORKTREE_PATH="../$(basename $(pwd))-overnight"
git worktree add "$WORKTREE_PATH" -b "$BRANCH_NAME" 2>/dev/null || {
  # ブランチ既存の場合
  git worktree add "$WORKTREE_PATH" "$BRANCH_NAME" 2>/dev/null || {
    echo "Worktree already exists, reusing"
    WORKTREE_PATH=$(git worktree list | grep overnight | awk '{print $1}')
  }
}
cd "$WORKTREE_PATH"
```

### 2d. 状態ディレクトリ初期化
```bash
mkdir -p .overnight-state/phases .overnight-state/issues .overnight-state/history .overnight-state/screenshots
```

**`.overnight-state/run.json`** を作成:
```json
{
  "run_id": "overnight-YYYY-MM-DD",
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
    "issues_found": 0, "issues_fixed": 0, "commits": 0, "patterns_learned": 0
  }
}
```

### 2e. メモリ読み込み
- `~/.claude/memory/qa_bug_patterns.md` を読み込む
- `~/.claude/lessons/` のレッスンをスキャン

### 2f. 設定読み込み
`.overnight-state/config.json` が存在すればそれを使用。なければ `~/.claude/skills/overnight/templates/config-default.json` をコピー。

---

## Step 3: フェーズ実行ループ

各フェーズの実行前に必ず:
1. **時間チェック**: `ELAPSED=$(( ($(date +%s) - START_TIME) / 60 ))` — Phase 10 用に常に 30 分確保
2. **フェーズ有効チェック**: `--phases` / `--skip` / `--dry-run` の設定に従う
3. **残り時間計算**: 各フェーズの割当時間を計算 (総時間 × 時間%)

フェーズがエラーで失敗した場合: **エラーをログに記録し、次のフェーズへ進む** (全体を止めない)

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

**infra-engineer** agent に委譲:

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

**backend** agent に委譲:

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

**backend** + **security-auditor** agent を並列起動:

**Agent 1 (backend): エンドポイント機能テスト**
```
以下のAPI運用テストを実行してください。
フレームワーク: $FRAMEWORK

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

### 軽量負荷テスト (サーバー起動時のみ)
- 主要エンドポイントに `ab -n 100 -c 10` または同等の負荷
- エラー率とレスポンスタイムの劣化を計測
```

結果を `.overnight-state/phases/phase-1c-api.md` に保存。

---

## Phase 2: コードQA

**flow-qa** agent に委譲 (既存の5パスQAをフル活用):

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

結果の issue を `.overnight-state/issues/all-issues.md` に追記。

---

## Phase 3: インタラクティブQA

**前提条件チェック**:
- `WEB_APP=true` か？
- Puppeteer MCP が利用可能か？
- Web アプリが起動しているか (`curl` でチェック)?

前提条件を満たさない場合は `SKIPPED (prerequisites not met)` としてログ記録し次のフェーズへ。

**詳細手順は `reference/phase-details.md` の「Phase 3: インタラクティブQA」セクション参照。**

概要:
1. **エラーコレクター注入** — `puppeteer_evaluate` で `window.__overnight_errors = []` + error/unhandledrejection リスナー設定
2. **ネットワーク傍受** — `fetch`/`XHR` をラップして全リクエスト記録
3. **ログイン** — `.overnight-state/config.json` のテストアカウントで認証
4. **各ページで自動テスト実行**:
   - 要素自動発見 (ボタン、フォーム、リンク、トグル)
   - デッドクリック検出
   - フォーム自動テスト (空送信、異常値、正常値)
   - トグル保持テスト (クリック→リロード→値保持確認)
   - API↔UI整合性チェック
   - 操作前後スクショ比較
5. **CRUDフローテスト** — 作成→確認→編集→削除 (自己完結)
6. **ユーザージャーニーテスト** — ルート定義からフロー自動生成→実行
7. **エラーページテスト** — 不正ID、権限なし、404
8. **コンソールエラー回収** — `window.__overnight_errors` を取得
9. **ネットワークエラー回収** — 4xx/5xx レスポンスを全件報告

結果を `.overnight-state/phases/phase-3-interactive.md` に保存。スクショは `.overnight-state/screenshots/` に保存。

---

## Phase 4: デザインQA

**前提条件**: `WEB_APP=true` && Puppeteer MCP 利用可能

**詳細手順は `reference/phase-details.md` の「Phase 4: デザインQA」セクション参照。**

概要:
1. **トークン監査** — 全要素の computed style 抽出 → デザインシステム照合
2. **レスポンシブチェック** — 3ビューポート (1440/768/375) で全ページスクショ → 崩れ検出
3. **アクセシビリティ** — コントラスト比計算、フォーカスリング、タッチターゲット、hover/active 状態
4. **Figma パリティ** (config に Figma URL がある場合のみ) — Figma MCP で仕様取得 → 実装比較
5. **コンポーネント一貫性** — コードから同目的の異なる実装パターンを検出
6. **エキスパートレビュー** — `ux-designer` + `design-critic` agent を並列起動 → A/B/C/D グレード

結果を `.overnight-state/phases/phase-4-design.md` に保存。

---

## Phase 5: パフォーマンスQA

**backend** + **infra-engineer** agent を並列起動:

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

**security-auditor** agent に委譲:

```
## チェック項目

### 脆弱性スキャン
- Node: `npm audit --json`
- Python: `pip-audit --format json` (利用可能な場合)
- Go: `govulncheck ./...` (利用可能な場合)
- Rust: `cargo audit` (利用可能な場合)

### 更新可能パッケージ
- `npm outdated` / `pip list --outdated`
- パッチ/マイナーアップデート: 自動修正候補 (Phase 8 で適用)
- メジャーアップデート: MEDIUM として報告のみ (DEFERRED)

### ライセンスコンプライアンス
- 全依存のライセンスをチェック
- 許可リスト: MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC
- 非許可ライセンスがあれば HIGH で報告

### CVE 重大度評価
- 発見された CVE ごとに実際の影響度を評価
- 既知エクスプロイトあり: CRITICAL
- 既知エクスプロイトなし: HIGH
```

結果を `.overnight-state/phases/phase-6-deps.md` に保存。

---

## Phase 7: ドキュメントQA

**documentation-engineer** agent に委譲:

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

## Phase 8: Fix & Refactor

**最重要フェーズ** — 全発見を統合して優先度順に修正。

### 8a. 課題統合
`.overnight-state/issues/all-issues.md` を読み込み:
1. 重大度順にソート: CRITICAL > HIGH > MEDIUM
2. 同一ファイルの issue をグループ化 (1回のファイル編集で複数修正)
3. `--dry-run` の場合はここで終了

### 8b. 修正ループ

`.overnight-state/config.json` の設定に従う:
- `auto_fix_severities`: デフォルト `["CRITICAL", "HIGH"]`
- `max_fixes_per_run`: デフォルト `20`
- `max_lines_changed_per_fix`: デフォルト `50`
- `never_modify`: `["*.lock", "*.env", "*.env.*", "migrations/", "*.migration.*", ".github/"]`

各 issue に対して (最大 `max_fixes_per_run` 件):

1. **時間チェック** — 残り時間が Phase 9+10 用の時間未満なら修正終了
2. **保護チェック** — `never_modify` パターンに該当 → `[DEFERRED]`
3. **サイズ見積** — 50行超の変更が必要 → `[DEFERRED] (too large for auto-fix)`
4. **ファイル読み込み** — 対象ファイルの内容を Read
5. **修正実行** — issue の種類に応じて:
   - コード品質/セキュリティ → 直接修正
   - デザイン (トークン化、コントラスト、a11y) → **frontend-web** agent に委譲
   - テスト不足 → **test-automator** agent に委譲
   - 複雑なバグ → **debugger** agent に委譲
6. **テスト実行** — `TEST_CMD` を実行
7. **結果判定**:
   - テスト PASS → `git add <specific files> && git commit -m "fix(overnight): <description>"`
   - テスト FAIL → `git checkout . && git clean -fd` → `[SKIPPED] fix caused test failure`
8. **issue 更新** — `[CLOSED]` + commit hash、または `[SKIPPED]` + 理由

### 8c. リファクタリング (最大1件)

修正完了後、Phase 2 で発見された最もインパクトの大きいリファクタリング機会を1件実施:
- **refactoring-specialist** agent に委譲
- テスト → commit or revert (同じプロトコル)

---

## Phase 9: E2E回帰確認

Phase 8 で修正を行った場合のみ実行 (修正0件ならスキップ)。

**前提条件**: `WEB_APP=true` && Puppeteer MCP 利用可能

1. 全ページにナビゲート → スクショ撮影
2. コンソールエラーチェック
3. Phase 3 で発見・修正された issue のページを重点確認
4. 修正前後の比較 → 改善されたか、悪化していないか確認

修正が UI を壊している場合:
- 該当 commit を `git revert` → issue を `[REVERTED]` に更新
- 朝レポートに「回帰テストで問題発見、revert 済み」と記録

結果を `.overnight-state/phases/phase-9-regression.md` に保存。

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

### 10b. レッスン作成

`~/.claude/rules/self-improvement.md` のプロトコルに従い:
- 修正が revert された場合 → レッスン作成
- フェーズが失敗した場合 → レッスン作成

### 10c. トレンド分析

`.overnight-state/history/` の過去レポートを読み込み:
- 発見数の推移 (減っているか？)
- 繰り返し出現する未修正 issue (3回以上出現 → CRITICAL に昇格)
- カテゴリ別の改善傾向

### 10d. 朝レポート生成

`.overnight-state/morning-report.md` を生成 (フォーマットは以下):

```markdown
# 🌅 Overnight Report — YYYY-MM-DD

## サマリ

| 指標 | 値 |
|------|---|
| 実行時間 | Xh Ym (N フェーズ完了) |
| 発見した課題 | N 件 |
| 自動修正 | M 件 (K commits) |
| 未修正 (要判断) | L 件 (C CRITICAL, H HIGH) |
| Revert | R 件 |
| 学習したパターン | P 件 |
| デザインスコア | A/B/C/D |
| ブランチ | `chore/overnight-YYYY-MM-DD` |

---

## ⚠️ 要対応 (人間の判断が必要)

### CRITICAL
1. **[ID]**: 説明
   - ファイル: `path:line`
   - 提案: ...

### HIGH
1. **[ID]**: 説明
   - ファイル: `path:line`
   - 提案: ...

---

## ✅ 自動修正済み

| # | Issue | 修正内容 | Commit |
|---|-------|---------|--------|
| 1 | ... | ... | `abc1234` |

---

## 📊 フェーズ別結果

### Phase 1a: インフラ健全性
- ...

### Phase 1b: DB運用チェック
- ...

### Phase 1c: API運用テスト
- ...

### Phase 2: コードQA
- ...

### Phase 3: インタラクティブQA
- デッドクリック: N 件
- フォームテスト: N ページ, M 件の問題
- CRUD フロー: PASS/FAIL
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

### Phase 7: ドキュメントQA
- ...

### Phase 9: E2E回帰確認
- テスト: PASS/FAIL
- Revert: N 件

---

## 📈 トレンド (直近5回)

| 日付 | 発見 | 修正 | 残り | パターン |
|------|------|------|------|---------|
| ... | ... | ... | ... | ... |

---

## 🧠 学習したパターン

1. **[PATTERN-ID]**: 説明
2. ...

---

## 次のアクション

1. CRITICAL/HIGH の未修正 issue を確認 → 方針決定
2. ブランチ `chore/overnight-YYYY-MM-DD` の diff を確認
3. 問題なければ `/ship` で PR 作成
```

### 10e. アーカイブ

- `morning-report.md` を `history/YYYY-MM-DD.md` にコピー
- `run.json` の `status` を `complete` に更新

### 10f. 通知

```bash
# macOS 通知
if command -v terminal-notifier &>/dev/null; then
  terminal-notifier -title "🌅 Overnight Complete" \
    -message "Found $ISSUES issues, fixed $FIXED. See morning-report.md" \
    -sound default
elif command -v osascript &>/dev/null; then
  osascript -e "display notification \"Found $ISSUES issues, fixed $FIXED\" with title \"Overnight Complete\" sound name \"Glass\""
fi
```

### 10g. 最終出力

ターミナルに朝レポートのサマリ部分を表示。

---

## 安全ルール (CRITICAL — 常に遵守)

1. **Worktree 隔離**: 全作業を専用 worktree で実行。メインの working tree は絶対に触らない。
2. **Push/PR 禁止**: 絶対に `git push` しない。絶対に PR を作成しない。ユーザーが朝に確認して判断する。
3. **Atomic Fix-or-Revert**: テスト失敗 = 即 revert (`git checkout . && git clean -fd`)。テストを書き換えて通すことは禁止。
4. **ファイル保護**: `never_modify` パターンのファイルは編集禁止。
5. **変更サイズ制限**: 1修正あたり最大50行の変更。超過は `[DEFERRED]`。
6. **修正数上限**: 1回の実行で最大20修正、30 commits。
7. **重大度ゲート**: デフォルトで CRITICAL/HIGH のみ自動修正。MEDIUM はレポートのみ。
8. **フェーズ障害耐性**: 1フェーズの失敗で全体を止めない。エラー記録して次へ。
9. **時間ガード**: Phase 10 用に常に30分を確保。超過時は即 Phase 10 へジャンプ。
10. **テストファイル改竄禁止**: テストを修正してテストを通すことは絶対禁止。ソースコードを修正する。
11. **破壊的操作禁止**: `DROP TABLE`, `DELETE FROM` (WHERE なし), `rm -rf` 等の破壊的操作は実行しない。
12. **インタラクティブQA 安全**: CRUD テストで作成したデータは必ず削除して跡を残さない。本番データは変更しない。
