---
name: capturing-kindle-pages
description: Kindle Cloud Readerから複数ページを連続キャプチャする際の5ステップ（ローディング判定・スクショ検証・方向判断・余白トリム・終了判断）を定義。Kindleページ自動キャプチャ機能の実装・改善時に使用。
trigger: Kindleのページキャプチャ機能を実装・デバッグするとき、または既存のキャプチャ処理の品質改善を行うとき
skills: [analyzing-kindle-dom-heuristics, capturing-memory-heavy-assets, detecting-ready-state, simulating-keyboard-via-debugger, retrying-screenshot-with-escalation]
---

# Kindleページ連続キャプチャの5ステップ

## Purpose

Kindle Cloud Readerから書籍のページを連続してスクリーンショット取得する際の、人間の判断プロセスを自動化可能な形で定義する。「ローディング待ち→スクショ→検証→めくり→終了判断」のサイクルを確実に回すためのSkill。

---

## Step 1: ローディング判定

### 人間のプロセス

- Kindleは2状態のみ: **ぐるぐる（スピナー）** or **本（コンテンツ）**
- ぐるぐるがなければOK、シンプルに判断

### 自動化のアプローチ

```typescript
// スピナー要素の存在チェック
const SPINNER_SELECTORS = [
  ".loading_spinner",
  "#kindleReader_spinner",
  "[class*='spinner']",
  "[class*='loading']"
];

async function isPageReady(): Promise<boolean> {
  for (const selector of SPINNER_SELECTORS) {
    const spinner = document.querySelector(selector);
    if (spinner && isVisible(spinner)) {
      return false; // まだローディング中
    }
  }
  return true;
}

// ポーリングで待機
async function waitForPageReady(maxWaitMs = 5000): Promise<boolean> {
  const startTime = Date.now();
  while (Date.now() - startTime < maxWaitMs) {
    if (await isPageReady()) return true;
    await delay(100);
  }
  return false; // タイムアウト
}
```

### 実装状況: 90%

- スピナー検出は実装済み（analyzing-kindle-dom-heuristics参照）
- Canvas描画完了の検知は部分的（ハッシュ比較）

---

## Step 2: スクショ検証

### 人間のプロセス

- 目視で3点確認: **図**、**文字**、**下のページ数**
- UI/シークバーが混入してないか確認
- ローディング中だったらスクショし直し

### 自動化のアプローチ

```typescript
interface ScreenshotValidation {
  hasContent: boolean;      // 白一色でないか
  hasUIContamination: boolean; // UIが映り込んでいないか
  isLoadingState: boolean;  // スピナーが写っていないか
}

async function validateScreenshot(dataUrl: string): Promise<ScreenshotValidation> {
  const bitmap = await createImageBitmap(dataUrlToBlob(dataUrl));
  const canvas = new OffscreenCanvas(bitmap.width, bitmap.height);
  const ctx = canvas.getContext("2d")!;
  ctx.drawImage(bitmap, 0, 0);
  const imageData = ctx.getImageData(0, 0, bitmap.width, bitmap.height);

  return {
    hasContent: detectContentPresence(imageData),
    hasUIContamination: detectUIElements(imageData),
    isLoadingState: detectSpinnerInImage(imageData),
  };
}

// コンテンツ存在判定（白一色でないか）
function detectContentPresence(imageData: ImageData): boolean {
  const pixels = imageData.data;
  let nonWhitePixels = 0;
  const threshold = pixels.length / 4 * 0.01; // 1%以上が非白ならOK

  for (let i = 0; i < pixels.length; i += 4) {
    const r = pixels[i], g = pixels[i + 1], b = pixels[i + 2];
    if (r < 250 || g < 250 || b < 250) nonWhitePixels++;
  }
  return nonWhitePixels > threshold;
}
```

### リトライ戦略

```typescript
// 検証失敗時はリトライ（retrying-screenshot-with-escalation参照）
if (!validation.hasContent || validation.isLoadingState) {
  await delay(500);
  return captureAndValidate(tabId, attempt + 1);
}
```

### 実装状況: 70%

- コンテンツ存在チェック: 実装済み（capturing-memory-heavy-assets）
- UI混入検出: 未実装（トリミングで回避中）
- スピナー画像認識: 未実装（DOM検出で代替）

---

## Step 3: 方向判断（RTL/LTR）

### 人間のプロセス

1. 本の種類であたりをつける（漫画→RTL、洋書→LTR）
2. 実際にめくってみる
3. ページ数が増えた→正解、減った→逆
4. ページ数がないページもある→数ページ試す

### 自動化のアプローチ

```typescript
type ReadingDirection = "ltr" | "rtl";

interface DirectionDetectionResult {
  direction: ReadingDirection;
  confidence: "high" | "medium" | "low";
  method: "css" | "aria" | "page_number" | "heuristic";
}

async function detectReadingDirection(tabId: number): Promise<DirectionDetectionResult> {
  // 1. CSSスタイルから判定（高信頼）
  const cssDirection = await detectFromCSS();
  if (cssDirection) {
    return { direction: cssDirection, confidence: "high", method: "css" };
  }

  // 2. ARIA属性から判定（高信頼）
  const ariaDirection = await detectFromARIA();
  if (ariaDirection) {
    return { direction: ariaDirection, confidence: "high", method: "aria" };
  }

  // 3. ページ番号変化で判定（中信頼）
  const pageDirection = await detectByPageNavigation(tabId);
  if (pageDirection) {
    return { direction: pageDirection, confidence: "medium", method: "page_number" };
  }

  // 4. デフォルト: 日本のKindleでは漫画が多いのでRTL
  return { direction: "rtl", confidence: "low", method: "heuristic" };
}

// ページ番号変化による判定
async function detectByPageNavigation(tabId: number): Promise<ReadingDirection | null> {
  const beforePage = await getCurrentPageNumber();
  if (beforePage === null) return null;

  // 右矢印で試す
  await navigateToNextPageViaDebugger(tabId, "ltr");
  await delay(300);
  const afterPage = await getCurrentPageNumber();

  if (afterPage === null) return null;

  if (afterPage > beforePage) {
    // 右矢印で進んだ = LTR
    return "ltr";
  } else if (afterPage < beforePage) {
    // 右矢印で戻った = RTL（次ページは左矢印）
    await navigateToNextPageViaDebugger(tabId, "rtl"); // 元に戻す
    await delay(300);
    return "rtl";
  }

  return null; // 判定不能
}
```

### 実装状況: 80%

- CSS/ARIA判定: 実装済み（analyzing-kindle-dom-heuristics）
- ページ番号追跡: 部分実装
- 自動試行による確認: 未実装

---

## Step 4: 余白トリム

### 人間のプロセス

- 上下左右で文字/画像がどこまであるか認識
- その枠をコンテンツ領域として切り出す

### 自動化のアプローチ

```typescript
// capturing-memory-heavy-assets で詳細実装済み
interface ContentBounds {
  x: number;
  y: number;
  width: number;
  height: number;
}

function findContentBounds(
  pixels: Uint8ClampedArray,
  width: number,
  height: number
): ContentBounds | null {
  let minX = width, minY = height, maxX = 0, maxY = 0;
  let hasContent = false;
  const TRIM_THRESHOLD = 250; // ほぼ白を許容

  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      const idx = (y * width + x) * 4;
      const r = pixels[idx], g = pixels[idx + 1];
      const b = pixels[idx + 2], a = pixels[idx + 3];

      // 白でも透明でもないピクセル = コンテンツ
      const isContent = a > 10 &&
        (r < TRIM_THRESHOLD || g < TRIM_THRESHOLD || b < TRIM_THRESHOLD);

      if (isContent) {
        hasContent = true;
        minX = Math.min(minX, x);
        maxX = Math.max(maxX, x);
        minY = Math.min(minY, y);
        maxY = Math.max(maxY, y);
      }
    }
  }

  if (!hasContent) return null;

  // パディング追加
  const padding = 10;
  return {
    x: Math.max(0, minX - padding),
    y: Math.max(0, minY - padding),
    width: Math.min(width, maxX - minX + 1 + padding * 2),
    height: Math.min(height, maxY - minY + 1 + padding * 2),
  };
}
```

### 安全弁

```typescript
const areaRatio = (bounds.width * bounds.height) / (originalWidth * originalHeight);
if (areaRatio > 0.95) return originalDataUrl; // 5%未満のトリムは無意味
if (areaRatio < 0.1) return originalDataUrl;  // 極端に小さい = 検出ミス
```

### 実装状況: 100%

- ピクセルレベル検出: 実装済み
- 安全弁（極端なトリム防止）: 実装済み
- メモリ管理: 実装済み（capturing-memory-heavy-assets）

---

## Step 5: 最終ページ判断

### 人間のプロセス

- 指定ページ数あり → その枚数で終了
- 指定なし → めくれなくなったら終了
- 内容でも判断（奥付＝出版日などが出たら終わり）

### 自動化のアプローチ

```typescript
interface CaptureEndCondition {
  type: "page_count" | "navigation_failed" | "content_pattern";
  reached: boolean;
  reason?: string;
}

async function checkEndCondition(
  state: CaptureState,
  config: CaptureConfig
): Promise<CaptureEndCondition> {
  // 1. 指定ページ数に到達
  if (config.targetPageCount && state.currentPage >= config.targetPageCount) {
    return { type: "page_count", reached: true, reason: `${config.targetPageCount}ページ完了` };
  }

  // 2. ページ遷移失敗（めくれなくなった）
  const pageChanged = await didPageChange(state.lastPageHash);
  if (!pageChanged && state.navigationAttempts >= 3) {
    return { type: "navigation_failed", reached: true, reason: "ページ遷移不可" };
  }

  // 3. 奥付パターン検出（将来実装）
  // const isColophon = await detectColophonPattern(currentImage);
  // if (isColophon) {
  //   return { type: "content_pattern", reached: true, reason: "奥付検出" };
  // }

  return { type: "page_count", reached: false };
}

// ページ変化の検出（ハッシュ比較）
async function didPageChange(lastHash: string | null): Promise<boolean> {
  const currentHash = await getCanvasHash();
  return currentHash !== lastHash;
}
```

### 実装状況: 60%

- 指定ページ数: 実装済み
- 遷移失敗検出: 実装済み（ハッシュ比較）
- 奥付パターン認識: 未実装

---

## Rules

1. **ローディング完了を必ず待つ**: スピナー消失 + 安定化時間（100-300ms）
2. **方向は最初に1回だけ判定**: キャプチャ中に方向を変えない
3. **トリムは元画像を保持**: トリム失敗時は元画像にフォールバック
4. **メモリを適切に解放**: ImageBitmap.close()、canvas.width=0（capturing-memory-heavy-assets参照）
5. **連続失敗で中断**: 5回連続失敗で全体を中断（retrying-screenshot-with-escalation参照）

---

## Anti-Patterns

- **スピナー非表示を待たずにスクショ**: 白画像やローディング画面がキャプチャされる
- **方向判定をキャプチャごとに行う**: 無駄なオーバーヘッド、状態の不整合
- **トリムで元画像を破棄**: 検出ミス時にリカバリ不能
- **ページ番号だけで終了判断**: ページ番号のないページで誤終了
- **無限リトライ**: 回復不可能なエラーでリソースを浪費

---

## Related Skills

| Skill | 役割 |
|-------|------|
| analyzing-kindle-dom-heuristics | DOM構造解析、セレクタ戦略 |
| capturing-memory-heavy-assets | トリミング、メモリ管理 |
| detecting-ready-state | ローディング完了検知 |
| simulating-keyboard-via-debugger | ページめくり操作 |
| retrying-screenshot-with-escalation | リトライ・エスカレーション |
