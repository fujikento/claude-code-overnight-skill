# Phase Details — Overnight Skill

SKILL.md から参照される各フェーズの詳細実行手順。

---

## Phase 3: インタラクティブQA — 詳細手順

### Step 3.0: 前提条件チェック

```bash
# Web アプリが起動しているか
WEB_URL=$(grep -o 'http://localhost:[0-9]*' .env .env.local next.config.* 2>/dev/null | head -1)
if [ -z "$WEB_URL" ]; then WEB_URL="http://localhost:3000"; fi

curl -s -o /dev/null -w "%{http_code}" "$WEB_URL" | grep -q "200\|301\|302"
```

起動していない場合: `SKIPPED (web app not running)` として Phase 4 へ。

### Step 3.1: エラーコレクター & ネットワーク傍受の注入

最初のページナビゲーション後に `puppeteer_evaluate` で以下を注入:

```javascript
// === エラーコレクター ===
window.__overnight_errors = [];
window.__overnight_error_count_before = 0;

window.addEventListener('error', (e) => {
  window.__overnight_errors.push({
    type: 'runtime',
    message: e.message,
    source: e.filename,
    line: e.lineno,
    col: e.colno,
    timestamp: new Date().toISOString()
  });
});

window.addEventListener('unhandledrejection', (e) => {
  window.__overnight_errors.push({
    type: 'promise',
    reason: String(e.reason),
    stack: e.reason?.stack || '',
    timestamp: new Date().toISOString()
  });
});

// === ネットワーク傍受 ===
window.__overnight_requests = [];

const originalFetch = window.fetch;
window.fetch = async function(...args) {
  const url = typeof args[0] === 'string' ? args[0] : args[0]?.url || '';
  const method = args[1]?.method || 'GET';
  const startTime = performance.now();
  
  try {
    const response = await originalFetch.apply(this, args);
    window.__overnight_requests.push({
      url, method,
      status: response.status,
      duration: Math.round(performance.now() - startTime),
      timestamp: new Date().toISOString()
    });
    return response;
  } catch (error) {
    window.__overnight_requests.push({
      url, method,
      status: 0,
      error: error.message,
      duration: Math.round(performance.now() - startTime),
      timestamp: new Date().toISOString()
    });
    throw error;
  }
};

// XHR もラップ
const originalXHROpen = XMLHttpRequest.prototype.open;
const originalXHRSend = XMLHttpRequest.prototype.send;

XMLHttpRequest.prototype.open = function(method, url) {
  this.__overnight_method = method;
  this.__overnight_url = url;
  return originalXHROpen.apply(this, arguments);
};

XMLHttpRequest.prototype.send = function() {
  const startTime = performance.now();
  this.addEventListener('loadend', () => {
    window.__overnight_requests.push({
      url: this.__overnight_url,
      method: this.__overnight_method,
      status: this.status,
      duration: Math.round(performance.now() - startTime),
      timestamp: new Date().toISOString(),
      type: 'xhr'
    });
  });
  return originalXHRSend.apply(this, arguments);
};

'interceptors_installed';
```

**重要**: ページ遷移のたびにインターセプターは消える。各ページで再注入が必要。ただし SPA (Next.js等) のクライアントサイド遷移では維持される。フルリロード時のみ再注入。

### Step 3.2: ログイン

`config.json` からテストアカウント情報を読み取り:

1. `puppeteer_navigate` → ログインページ
2. `puppeteer_fill` → メール/パスワード入力
3. `puppeteer_click` → ログインボタン
4. 2秒待機
5. `puppeteer_screenshot` → ログイン成功確認 (ダッシュボードが表示されているか)

ログイン失敗時: `SKIPPED (login failed)` として Phase 4 へ。

### Step 3.3: ページリスト構築

`config.json` にページリストがある場合はそれを使用。なければコードからルート定義を抽出:

```bash
# Next.js App Router
find app -name "page.tsx" -o -name "page.jsx" | sed 's|app||;s|/page\.[tj]sx||;s|\[.*\]|:param|g'

# Next.js Pages Router  
find pages -name "*.tsx" -o -name "*.jsx" | sed 's|pages||;s|\.[tj]sx||;s|\[.*\]|:param|g'

# React Router (grep for Route/path patterns)
grep -r 'path.*["\x27]/' --include="*.tsx" --include="*.jsx" | sed 's/.*path.*["\x27]\([^"\x27]*\).*/\1/'
```

### Step 3.4: 各ページのインタラクティブテスト

各ページに対して以下を順次実行:

#### 3.4a: ナビゲーション & 基本チェック
1. `puppeteer_navigate` → ページURL
2. 2秒待機 (データロード)
3. インターセプター再注入 (フルリロードの場合)
4. `puppeteer_screenshot` → 基本スクショ
5. AI分析: 白画面? エラー表示? ローディングスタック?

#### 3.4b: 要素自動発見
```javascript
// puppeteer_evaluate
(() => {
  const getSelector = (el) => {
    if (el.id) return '#' + el.id;
    if (el.dataset.testid) return `[data-testid="${el.dataset.testid}"]`;
    const classes = [...el.classList].join('.');
    if (classes) {
      const sel = el.tagName.toLowerCase() + '.' + classes;
      if (document.querySelectorAll(sel).length === 1) return sel;
    }
    // フォールバック: nth-child パス
    const path = [];
    let current = el;
    while (current.parentElement) {
      const idx = [...current.parentElement.children].indexOf(current) + 1;
      path.unshift(`${current.tagName.toLowerCase()}:nth-child(${idx})`);
      current = current.parentElement;
      if (current.id) { path.unshift('#' + current.id); break; }
    }
    return path.join(' > ');
  };

  const buttons = [...document.querySelectorAll('button, [role="button"], input[type="submit"], input[type="button"]')]
    .filter(el => !el.disabled && el.offsetParent !== null)
    .map(el => ({
      type: 'button',
      text: (el.textContent || el.value || '').trim().slice(0, 50),
      selector: getSelector(el),
      hasOnClick: !!el.onclick || el.hasAttribute('onclick')
    }));

  const links = [...document.querySelectorAll('a[href]')]
    .filter(el => el.offsetParent !== null)
    .map(el => ({
      type: 'link',
      text: el.textContent.trim().slice(0, 50),
      href: el.href,
      selector: getSelector(el)
    }));

  const forms = [...document.querySelectorAll('form')]
    .map(form => ({
      type: 'form',
      action: form.action,
      method: form.method,
      selector: getSelector(form),
      inputs: [...form.querySelectorAll('input, textarea, select')]
        .map(input => ({
          type: input.type || input.tagName.toLowerCase(),
          name: input.name,
          required: input.required,
          selector: getSelector(input)
        }))
    }));

  const toggles = [...document.querySelectorAll('[role="switch"], input[type="checkbox"], .toggle, [data-state]')]
    .filter(el => el.offsetParent !== null)
    .map(el => ({
      type: 'toggle',
      selector: getSelector(el),
      text: (el.closest('label')?.textContent || el.getAttribute('aria-label') || '').trim().slice(0, 50),
      checked: el.checked || el.dataset.state === 'checked' || el.getAttribute('aria-checked') === 'true'
    }));

  return { buttons, links, forms, toggles,
    summary: `${buttons.length} buttons, ${links.length} links, ${forms.length} forms, ${toggles.length} toggles` };
})()
```

#### 3.4c: デッドクリック検出

各ボタンに対して (最大20個/ページ — ナビゲーション系リンクは除外):

1. エラー数とリクエスト数を記録: `window.__overnight_errors.length`, `window.__overnight_requests.length`
2. DOM のスナップショット取得: `document.body.innerHTML.length`
3. `puppeteer_click` → ボタンをクリック
4. 1秒待機
5. 変化を確認:
   ```javascript
   const domChanged = document.body.innerHTML.length !== PREV_LENGTH;
   const newErrors = window.__overnight_errors.length - PREV_ERROR_COUNT;
   const newRequests = window.__overnight_requests.length - PREV_REQUEST_COUNT;
   const modalOpened = !!document.querySelector('[role="dialog"], .modal, [data-state="open"]');
   ```
6. 判定:
   - DOM変化なし + リクエストなし + モーダルなし = **デッドクリック** → `[HIGH] INTERACTIVE — Dead click: "ボタンテキスト" on /page`
   - 新しいエラーが発生 = **クリックでエラー** → `[CRITICAL] INTERACTIVE — Click error: "ボタンテキスト" on /page — error message`
7. 状態リセット: ページを再ナビゲート (破壊的操作の可能性があるため)

**安全ルール**: 「削除」「delete」「remove」「キャンセル」等のテキストを含むボタンはクリックしない。confirm ダイアログが出た場合はキャンセルする。

#### 3.4d: フォーム自動テスト

各フォームに対して:

**テスト1: 空送信**
1. フォームの submit ボタンをクリック (何も入力せず)
2. バリデーションエラーが表示されるか確認
3. 表示されない場合: `[HIGH] INTERACTIVE — Form accepts empty submission on /page`

**テスト2: 異常値**
1. テキスト入力に `<script>alert('xss')</script>` を入力
2. 数値入力に `999999999999` を入力
3. メール入力に `not-an-email` を入力
4. submit → エラーハンドリングされるか確認
5. XSS が実行された場合: `[CRITICAL] SECURITY — XSS vulnerability in form on /page`

**テスト3: 正常値** (config にテストデータがある場合)
1. 適切な値を入力
2. submit
3. ネットワークリクエストを確認 → 200/201 が返るか
4. エラーレスポンスの場合: `[HIGH] INTERACTIVE — Form submission failed on /page — status code`

**安全ルール**: フォーム送信は GET/POST のみ。DELETE メソッドのフォームはテストしない。

#### 3.4e: トグル保持テスト

各トグル/スイッチに対して:

1. 現在の状態を記録 (`checked` / `data-state`)
2. `puppeteer_click` → トグルクリック
3. 1秒待機
4. ネットワークリクエストを確認: API コール発生したか?
   - なし: `[HIGH] INTERACTIVE — Toggle "label" only updates local state, no API call`
5. ページをフルリロード (`puppeteer_navigate` で同じURL)
6. トグルの状態を再確認
   - 元に戻っている: `[HIGH] INTERACTIVE — Toggle "label" state not persisted after reload`
7. 元の状態に戻す (もう一度クリック)

#### 3.4f: API↔UI 整合性チェック

ページが API からデータを取得している場合:

1. ネットワーク傍受から GET リクエストの URL を取得
2. `curl` で同じ API を直接叩いてレスポンスを取得
3. `puppeteer_evaluate` で DOM 内のテキストデータを抽出
4. 比較: API の数値 vs UI の表示が一致するか
   - 不一致: `[HIGH] INTERACTIVE — API/UI mismatch: API returns X but UI shows Y on /page`

### Step 3.5: CRUDフローテスト

config に CRUD テスト対象ページが定義されている場合:

1. **Create**: フォームに入力 → 送信 → リスト画面で新アイテム確認
2. **Read**: リスト画面でデータが表示されるか確認
3. **Update**: 作成したアイテムの編集 → 保存 → 変更反映確認
4. **Delete**: 作成したアイテムを削除 → リストから消えたか確認

**安全ルール**: テストで作成したデータは必ず削除して元の状態に戻す。

### Step 3.6: ユーザージャーニーテスト

config にユーザージャーニーが定義されている場合、各ジャーニーを実行:

```json
{
  "journeys": [
    {
      "name": "注文フロー",
      "steps": [
        { "action": "navigate", "url": "/orders" },
        { "action": "click", "selector": "button:has-text('新規注文')" },
        { "action": "fill", "selector": "input[name='item']", "value": "テスト商品" },
        { "action": "click", "selector": "button[type='submit']" },
        { "action": "assert", "check": "text_visible", "text": "注文が作成されました" }
      ]
    }
  ]
}
```

定義がない場合: ページリストから基本的なナビゲーションフローを自動生成して実行 (全ページ遷移が成功するか)。

### Step 3.7: エラーページテスト

1. 存在しないパスにアクセス: `/this-page-does-not-exist-overnight-test`
   - 適切な404ページが表示されるか (白画面でないか)
2. 存在しないIDでアクセス (パラメータルートがある場合): `/orders/999999999`
   - エラーハンドリングされるか (500にならないか)
3. コンソールエラーなしで処理されるか確認

### Step 3.8: 結果回収

全ページテスト完了後:

```javascript
// puppeteer_evaluate
({
  errors: window.__overnight_errors,
  requests: window.__overnight_requests.filter(r => r.status >= 400),
  totalRequests: window.__overnight_requests.length
})
```

結果を `.overnight-state/phases/phase-3-interactive.md` に保存。

---

## Phase 4: デザインQA — 詳細手順

### Step 4.1: トークン監査

各ページで `puppeteer_evaluate`:

```javascript
(() => {
  const DESIGN_SCALE = {
    spacing: [0, 1, 2, 4, 6, 8, 10, 12, 14, 16, 20, 24, 28, 32, 36, 40, 44, 48, 56, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 384],
    fontSize: [10, 11, 12, 13, 14, 15, 16, 18, 20, 24, 30, 36, 48, 60, 72, 96, 128],
    borderRadius: [0, 2, 4, 6, 8, 12, 16, 9999]
  };

  const violations = [];
  const colorMap = {};
  const fontSizeMap = {};
  const spacingMap = {};

  const parsePixels = (val) => {
    if (!val || val === 'auto' || val === 'normal') return null;
    const match = val.match(/^([\d.]+)px$/);
    return match ? parseFloat(match[1]) : null;
  };

  const isOnScale = (value, scale) => {
    if (value === null) return true;
    return scale.some(s => Math.abs(s - value) < 0.5);
  };

  const elements = document.querySelectorAll('*');
  for (const el of elements) {
    if (!el.offsetParent && el.tagName !== 'BODY' && el.tagName !== 'HTML') continue;
    const cs = getComputedStyle(el);

    // 色の収集
    const color = cs.color;
    const bgColor = cs.backgroundColor;
    if (color && color !== 'rgba(0, 0, 0, 0)') colorMap[color] = (colorMap[color] || 0) + 1;
    if (bgColor && bgColor !== 'rgba(0, 0, 0, 0)') colorMap[bgColor] = (colorMap[bgColor] || 0) + 1;

    // フォントサイズ
    const fs = parsePixels(cs.fontSize);
    if (fs) fontSizeMap[fs] = (fontSizeMap[fs] || 0) + 1;

    // スペーシング (padding/margin)
    for (const prop of ['paddingTop', 'paddingRight', 'paddingBottom', 'paddingLeft', 'marginTop', 'marginRight', 'marginBottom', 'marginLeft']) {
      const val = parsePixels(cs[prop]);
      if (val && val > 0) {
        spacingMap[val] = (spacingMap[val] || 0) + 1;
        if (!isOnScale(val, DESIGN_SCALE.spacing)) {
          violations.push({
            type: 'spacing_off_scale',
            element: el.tagName + (el.className ? '.' + [...el.classList].join('.') : ''),
            property: prop,
            value: val,
            nearest: DESIGN_SCALE.spacing.reduce((prev, curr) =>
              Math.abs(curr - val) < Math.abs(prev - val) ? curr : prev
            )
          });
        }
      }
    }

    // border-radius
    const br = parsePixels(cs.borderRadius);
    if (br && br > 0 && !isOnScale(br, DESIGN_SCALE.borderRadius)) {
      violations.push({
        type: 'border_radius_off_scale',
        element: el.tagName + (el.className ? '.' + [...el.classList].join('.') : ''),
        value: br
      });
    }
  }

  return {
    uniqueColors: Object.keys(colorMap).length,
    topColors: Object.entries(colorMap).sort((a,b) => b[1]-a[1]).slice(0, 20),
    fontSizes: Object.keys(fontSizeMap).sort((a,b) => a-b),
    spacingViolations: violations.filter(v => v.type === 'spacing_off_scale').length,
    radiusViolations: violations.filter(v => v.type === 'border_radius_off_scale').length,
    violations: violations.slice(0, 50)
  };
})()
```

**問題判定**:
- ユニーク色が30種超: `[MEDIUM] DESIGN — Too many unique colors (N), design system may not be enforced`
- スペーシング違反が20件超: `[MEDIUM] DESIGN — N spacing values off design scale`
- `var(--` を使わず直接色値を使っている箇所 → Grep でコード内から検出

### Step 4.2: レスポンシブチェック

3つのビューポートで全ページをスクショ:

```
Desktop: puppeteer_navigate with launchOptions: { args: ['--window-size=1440,900'] }
Tablet:  puppeteer_evaluate → window.resizeTo(768, 1024) (or re-navigate with viewport)
Mobile:  puppeteer_evaluate → window.resizeTo(375, 812)
```

実際にはスクショのサイズパラメータで制御:
- `puppeteer_screenshot` → `width: 1440, height: 900` (Desktop)
- `puppeteer_screenshot` → `width: 768, height: 1024` (Tablet)
- `puppeteer_screenshot` → `width: 375, height: 812` (Mobile)

各スクショをAI分析:
- 横スクロール発生していないか
- テキスト切れ
- レイアウト崩壊
- 要素の重なり

```javascript
// 横スクロール検出
puppeteer_evaluate:
document.documentElement.scrollWidth > document.documentElement.clientWidth
```

### Step 4.3: アクセシビリティチェック

```javascript
(() => {
  const issues = [];

  // コントラスト比計算
  const luminance = (r, g, b) => {
    const [rs, gs, bs] = [r, g, b].map(c => {
      c = c / 255;
      return c <= 0.03928 ? c / 12.92 : Math.pow((c + 0.055) / 1.055, 2.4);
    });
    return 0.2126 * rs + 0.7152 * gs + 0.0722 * bs;
  };

  const parseColor = (color) => {
    const match = color.match(/rgba?\((\d+),\s*(\d+),\s*(\d+)/);
    if (match) return [parseInt(match[1]), parseInt(match[2]), parseInt(match[3])];
    return null;
  };

  const contrastRatio = (fg, bg) => {
    const l1 = luminance(...fg);
    const l2 = luminance(...bg);
    const lighter = Math.max(l1, l2);
    const darker = Math.min(l1, l2);
    return (lighter + 0.05) / (darker + 0.05);
  };

  const getEffectiveBg = (el) => {
    let current = el;
    while (current) {
      const bg = getComputedStyle(current).backgroundColor;
      const parsed = parseColor(bg);
      if (parsed && (parsed[0] !== 0 || parsed[1] !== 0 || parsed[2] !== 0 || bg.includes('rgba(0, 0, 0, 0)') === false)) {
        return parsed;
      }
      current = current.parentElement;
    }
    return [255, 255, 255]; // デフォルト白
  };

  // テキスト要素のコントラストチェック
  const textElements = document.querySelectorAll('p, span, h1, h2, h3, h4, h5, h6, a, button, label, td, th, li, dt, dd');
  for (const el of textElements) {
    if (!el.offsetParent && el.tagName !== 'BODY') continue;
    if (!el.textContent.trim()) continue;

    const cs = getComputedStyle(el);
    const fg = parseColor(cs.color);
    const bg = getEffectiveBg(el);
    if (!fg || !bg) continue;

    const ratio = contrastRatio(fg, bg);
    const fontSize = parseFloat(cs.fontSize);
    const isBold = parseInt(cs.fontWeight) >= 700;
    const isLargeText = fontSize >= 24 || (fontSize >= 18.66 && isBold);
    const threshold = isLargeText ? 3 : 4.5;

    if (ratio < threshold) {
      issues.push({
        type: 'contrast',
        element: el.tagName,
        text: el.textContent.trim().slice(0, 30),
        ratio: Math.round(ratio * 100) / 100,
        required: threshold,
        fg: `rgb(${fg.join(',')})`,
        bg: `rgb(${bg.join(',')})`
      });
    }
  }

  // タッチターゲットチェック (44x44px)
  const interactive = document.querySelectorAll('button, a, input, select, textarea, [role="button"], [role="checkbox"], [role="switch"]');
  for (const el of interactive) {
    if (!el.offsetParent) continue;
    const rect = el.getBoundingClientRect();
    if (rect.width < 44 || rect.height < 44) {
      issues.push({
        type: 'touch_target',
        element: el.tagName,
        text: (el.textContent || el.value || '').trim().slice(0, 30),
        width: Math.round(rect.width),
        height: Math.round(rect.height)
      });
    }
  }

  // フォーカスリングチェック (サンプリング)
  const focusable = [...document.querySelectorAll('button, a[href], input, select, textarea')].slice(0, 10);
  const focusIssues = [];
  for (const el of focusable) {
    el.focus();
    const cs = getComputedStyle(el);
    const hasOutline = cs.outlineStyle !== 'none' && cs.outlineWidth !== '0px';
    const hasRing = cs.boxShadow !== 'none';
    if (!hasOutline && !hasRing) {
      focusIssues.push(el.tagName + ': ' + (el.textContent || '').trim().slice(0, 20));
    }
  }
  if (focusIssues.length > 0) {
    issues.push({ type: 'focus_ring', elements: focusIssues });
  }

  return { total: issues.length, issues: issues.slice(0, 50) };
})()
```

### Step 4.4: Figma パリティチェック

config に `figma_file_key` と `figma_node_ids` がある場合のみ:

1. `mcp__claude_ai_Figma__get_design_context` で Figma デザイン仕様取得
2. `mcp__claude_ai_Figma__get_screenshot` で Figma スクショ取得
3. Puppeteer で実装のスクショ取得
4. AI で視覚比較: 色、スペーシング、フォント、レイアウト構造の差異を報告

設定がない場合: `SKIPPED (no Figma config)` — 他のデザインチェックは実行する。

### Step 4.5: コンポーネント一貫性チェック

コードベースを Grep で分析:

```bash
# ボタンの実装パターン
grep -r "className.*btn\|<Button\|<button" --include="*.tsx" --include="*.jsx" | \
  sed 's/.*className="\([^"]*\)".*/\1/' | sort | uniq -c | sort -rn

# モーダル/ダイアログ
grep -r "Dialog\|Modal\|modal\|dialog" --include="*.tsx" --include="*.jsx" | wc -l

# テーブル
grep -r "<Table\|<table\|DataTable\|DataGrid" --include="*.tsx" --include="*.jsx" | wc -l
```

同じ目的に3種類以上の実装パターン → `[MEDIUM] DESIGN — Inconsistent component usage`

### Step 4.6: エキスパートデザインレビュー

`ux-designer` + `design-critic` を並列起動。各ページのスクショを渡して:

**ux-designer へ:**
```
以下のページスクショのデザインレビューをしてください。

## レビュー観点
1. 色の一貫性 (デザインシステムに沿っているか)
2. タイポグラフィ (フォントサイズ・ウェイトの一貫性)
3. スペーシング (余白のリズム)
4. コンポーネント再利用度
5. アクセシビリティ
6. マイクロインタラクション

## 出力
A/B/C/D グレードと具体的な改善ポイント (最大5件)
```

**design-critic へ:**
```
デザインディレクターとして厳格に評価してください。

## 評価基準
1. ビジュアルバランス
2. 情報の階層構造
3. 余白の使い方
4. 色のコントラストと調和
5. 全体的なプロフェッショナリズム

## 出力
10点満点のスコアと、最も改善インパクトの高い指摘3件
```

---

## Phase 9: E2E回帰確認 — 詳細手順

Phase 8 で修正を行った場合のみ実行。

### Step 9.1: 基本回帰

修正されたファイルに関連するページを特定:
- `.tsx` / `.jsx` ファイル → そのコンポーネントが使われるページ
- API ルートファイル → そのAPIを呼ぶページ
- CSS / スタイルファイル → 全ページ

### Step 9.2: ページ再テスト

対象ページに対して:
1. `puppeteer_navigate` → ページ
2. エラーコレクター注入
3. `puppeteer_screenshot` → スクショ
4. コンソールエラーチェック
5. Phase 3 で発見した問題が修正されているか確認
6. 新しい問題が発生していないか確認

### Step 9.3: 回帰検出時の対応

修正が新しい問題を引き起こした場合:
1. 該当コミットを特定
2. `git revert <commit-hash> --no-edit`
3. issue を `[REVERTED]` に更新
4. 朝レポートに記録: "Phase 9 で回帰検出、commit X を revert"
