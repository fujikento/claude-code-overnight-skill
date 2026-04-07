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

### Step 3.1: ネイティブダイアログ無効化 & エラーコレクター & ネットワーク傍受の注入

最初のページナビゲーション後に `puppeteer_evaluate` で以下を注入。**ダイアログ無効化が最初** (XSS テストや confirm ダイアログでフリーズ防止):

```javascript
// === ネイティブダイアログ無効化 (最初に実行) ===
window.alert = () => {};
window.confirm = () => true;
window.prompt = () => '';

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

**安全ルール — セマンティック破壊検出**: 以下のいずれかに該当するボタンはクリックしない:
- テキストが `config.destructive_detection.keywords` に一致 (多言語対応)
- `aria-label` が `config.destructive_detection.aria_patterns` に一致
- `data-destructive`, `data-danger`, `data-action="delete"` 等の data 属性
- `.btn-danger`, `.btn-destructive` 等の CSS クラス
- `confirm()` はオーバーライド済みだが、セマンティック検出で事前にスキップする方が安全。

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

**クリーンアップ失敗時の対応**: ネットワーク断やセッション切れで削除が失敗した場合:
1. 作成したレコードの ID/名前を記録
2. 再度削除を試みる (最大2回リトライ)
3. それでも失敗した場合: `[INFO] CRUD cleanup failed — orphaned record ID: <id> on <page>` として朝レポートに記録
4. 自動的にデータを壊す操作 (直接SQLなど) は絶対に実行しない

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

**重要**: デザインスケールの値は `config.json` の `phases.4.design_scale` から読み込む (唯一の信頼できる情報源)。JS ペイロード内にハードコードしない。

```javascript
// DESIGN_SCALE は config.json の phases.4.design_scale から注入される
// 例: puppeteer_evaluate 呼び出し前に config を読んで値を埋め込む
(() => {
  // config.json から注入 (SKILL.md が実行前に置換)
  const DESIGN_SCALE = __DESIGN_SCALE_FROM_CONFIG__;

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

## Phase 8-9: Evaluator 採点シート — 詳細手順

Phase 8-9 は統合された Generator-Evaluator ループ。旧 Phase 9 (回帰確認) は Evaluator の一部として吸収。

### ブラウザエンジン選択

```
# 優先順位: Playwright MCP → Puppeteer MCP → TEST_CMD のみ
BROWSER_ENGINE="none"
if Playwright MCP tools available (mcp__playwright__*):
  BROWSER_ENGINE="playwright"
elif Puppeteer MCP tools available (mcp__puppeteer__*):
  BROWSER_ENGINE="puppeteer"
```

ツール名マッピング:
| 操作 | Playwright MCP | Puppeteer MCP |
|------|---------------|---------------|
| ページ遷移 | `mcp__playwright__browser_navigate` | `mcp__puppeteer__puppeteer_navigate` |
| スクリーンショット | `mcp__playwright__browser_take_screenshot` | `mcp__puppeteer__puppeteer_screenshot` |
| クリック | `mcp__playwright__browser_click` | `mcp__puppeteer__puppeteer_click` |
| テキスト入力 | `mcp__playwright__browser_type` | `mcp__puppeteer__puppeteer_fill` |
| JS 実行 | `mcp__playwright__browser_evaluate` | `mcp__puppeteer__puppeteer_evaluate` |
| ホバー | `mcp__playwright__browser_hover` | `mcp__puppeteer__puppeteer_hover` |
| セレクト | `mcp__playwright__browser_select_option` | `mcp__puppeteer__puppeteer_select` |

### Evaluator 採点ルーブリック (100点満点)

#### カテゴリ 1: 機能テスト (20点)

```
TEST_CMD を実行:
- 全テスト PASS → 20点
- 一部 FAIL だが修正対象外のテスト → 15点
- 修正に関連するテスト FAIL → 0点
- TEST_CMD が存在しない → 型チェック (tsc --noEmit) で代替、PASS=20, FAIL=0
```

#### カテゴリ 2: ブラウザ動作テスト (20点)

修正した issue に関連するページで実操作検証:

```
Step 1: 該当ページにナビゲート
Step 2: ネイティブダイアログ無効化 + エラーコレクター注入
Step 3: issue の種類に応じたテスト実行:

[デッドクリック修正の場合]
  - 修正されたボタンをクリック
  - DOM 変化 or ネットワークリクエスト or モーダル表示を確認
  - 動作あり → 20点、動作なし → 0点

[フォーム修正の場合]
  - フォームに正常値を入力 → submit
  - 200/201 レスポンス → 20点
  - バリデーションエラー適切表示 → 15点 (修正が不完全だが改善)
  - エラーハンドリングなし → 0点

[トグル修正の場合]
  - トグルクリック → API コール確認 → リロード → 値保持確認
  - 完全動作 → 20点
  - API コールあるが保持されない → 10点
  - API コールなし → 0点

[API エラー修正の場合]
  - curl で API を直接叩く
  - 200 返却 + 正常レスポンス → 20点
  - 200 だが不正なレスポンス → 10点
  - 4xx/5xx → 0点

[コードのみの修正 (UIなし) の場合]
  - 修正箇所を静的解析
  - issue が解決されていると判断 → 20点
  - 部分的に解決 → 10点
  - 解決されていない → 0点
```

#### カテゴリ 3: ビジュアル品質 (20点)

修正前後のスクショを比較:

```
Step 1: 修正前のスクショ (Phase 3 で撮影済み) を参照
Step 2: 修正後のページをスクショ撮影
Step 3: AI で比較分析:

チェック項目:
- レイアウト崩壊なし → +5点
- 要素の重なりなし → +5点
- テキスト切れなし → +5点
- 修正による意図しない視覚的変化なし → +5点

減点:
- 新しいレイアウト崩壊 → -10点
- 新しい要素重なり → -10点
- 既存要素が消えた → -15点
- 白画面 / エラー表示 → -20点 (0点に)
```

**Web アプリでない場合**: このカテゴリは満点 (20点) 扱い。

#### カテゴリ 4: データ整合性 (20点)

```
Step 1: 修正後のページで表示されるデータを抽出 (browser_evaluate)
Step 2: 同じデータを API 経由で取得 (curl)
Step 3: 比較:

- 数値が完全一致 → 10点
- テキストが完全一致 → 5点
- リスト件数が一致 → 5点

減点:
- 数値不一致 → -10点
- データ欠損 (API にあるが UI に表示されない) → -5点
- 余分なデータ (UI に表示されるが API にない) → -5点
```

**Web アプリでない / API がない場合**: 満点 (20点) 扱い。

#### カテゴリ 5: アクセシビリティ (20点)

```
Step 1: 修正後のページで a11y チェック実行 (browser_evaluate で JS 注入):

チェック項目:
- テキストコントラスト比 ≥ 4.5:1 (通常) / ≥ 3:1 (大文字) → +8点
- タッチターゲット ≥ 44px → +4点
- フォーカスリング可視 → +4点
- aria-label / role 適切 → +4点

減点:
- コントラスト比 < 4.5:1 → -4点/箇所 (最大-8)
- タッチターゲット < 44px → -2点/箇所 (最大-4)
- フォーカスリングなし → -4点
- aria 不適切 → -2点/箇所 (最大-4)

修正前より a11y スコアが悪化した場合: さらに -5点のペナルティ
```

**Web アプリでない場合**: 満点 (20点) 扱い。

### Evaluator のフィードバック生成

スコアに関わらず、Evaluator は Generator に以下のフィードバックを返す:

**合格 (100点) の場合も必ずフィードバック**:
```
{
  "score": 100,
  "feedback_to_generator": "完璧。修正パターンが効果的だった: [具体的に何が良かったか]",
  "learnings": "このパターンの修正には [手法] が有効。次回も同様のアプローチを推奨。"
}
```

**不合格 (<100点) の場合**:
```
{
  "score": 75,
  "feedback_to_generator": "機能は動作するがコントラスト比が不足 (3.2:1, 要4.5:1)。text-gray-400 → text-gray-600 に変更すべき。",
  "specific_failures": [
    { "category": "a11y", "detail": "contrast ratio 3.2:1 on .btn-primary text", "suggestion": "darken text color" }
  ],
  "screenshot_path": ".overnight-state/screenshots/issue-001-eval-2.png"
}
```

### 収束判定の詳細

```
ループ履歴の例:

Attempt 1: Score 60  → Generator に FB → 再修正
Attempt 2: Score 80  → Generator に FB → 再修正
Attempt 3: Score 85  → Generator に FB → 再修正
Attempt 4: Score 85  → 同スコア1回目
Attempt 5: Score 85  → 同スコア2回目 → 収束判定: STOP
→ Score 85 でコミット ([PARTIAL] ステータス)

別のパターン:
Attempt 1: Score 70  → Generator に FB → 再修正
Attempt 2: Score 95  → Generator に FB → 再修正
Attempt 3: Score 90  → スコア低下! → Attempt 2 のベスト (95) に戻す → STOP
→ Score 95 でコミット ([CLOSED] ステータス)

最良パターン:
Attempt 1: Score 80  → Generator に FB → 再修正
Attempt 2: Score 100 → 完璧! → commit → STOP
→ Score 100 で commit + 成功パターン記録
```

### Evaluator の実行効率

各 issue の評価で全ページをテストするのではなく、**影響範囲のみテスト**:

```
修正ファイル → 影響ページのマッピング:
- components/Button.tsx → Button を使う全ページ (grep で特定)
- app/orders/page.tsx → /orders のみ
- styles/globals.css → 全ページ
- lib/api/orders.ts → /orders, /dashboard (API を使うページ)

最小テストセット = 修正ファイルから影響を受けるページのみ
```
