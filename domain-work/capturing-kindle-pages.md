---
name: capturing-kindle-pages
description: Kindle Cloud Readerの連続キャプチャ5ステップ（ローディング判定・スクショ検証・方向判断・余白トリム・終了判断）
skills: [analyzing-kindle-dom-heuristics, capturing-memory-heavy-assets, detecting-ready-state, simulating-keyboard-via-debugger, retrying-screenshot-with-escalation]
---
# Kindleページ連続キャプチャ

## Step 1: ローディング判定
```typescript
const SPINNER_SELECTORS = [".loading_spinner", "#kindleReader_spinner", "[class*='spinner']", "[class*='loading']"];

async function waitForPageReady(maxWaitMs = 5000): Promise<boolean> {
  const startTime = Date.now();
  while (Date.now() - startTime < maxWaitMs) {
    const spinning = SPINNER_SELECTORS.some(s => {
      const el = document.querySelector(s);
      return el && isVisible(el);
    });
    if (!spinning) return true;
    await delay(100);
  }
  return false;
}
```

## Step 2: スクショ検証
3点確認: コンテンツ存在（白一色でない）、UI混入なし、ローディングでない。
```typescript
function detectContentPresence(imageData: ImageData): boolean {
  const pixels = imageData.data;
  let nonWhite = 0;
  const threshold = pixels.length / 4 * 0.01; // 1%以上が非白ならOK
  for (let i = 0; i < pixels.length; i += 4) {
    if (pixels[i] < 250 || pixels[i+1] < 250 || pixels[i+2] < 250) nonWhite++;
  }
  return nonWhite > threshold;
}
```
検証失敗時は500ms待ってリトライ（retrying-screenshot-with-escalation参照）。

## Step 3: 方向判断（RTL/LTR）
優先順: CSS `dir` → ARIA属性 → ページ番号変化で判定。
- 右矢印でページ番号増加 → LTR、減少 → RTL
- 判定不能時のデフォルト: RTL（日本Kindleは漫画が多い）
- **最初に1回だけ判定、キャプチャ中は変えない**

## Step 4: 余白トリム
```typescript
function findContentBounds(pixels: Uint8ClampedArray, width: number, height: number): ContentBounds | null {
  let minX = width, minY = height, maxX = 0, maxY = 0;
  const TRIM_THRESHOLD = 250;
  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      const idx = (y * width + x) * 4;
      if (pixels[idx+3] > 10 && (pixels[idx] < TRIM_THRESHOLD || pixels[idx+1] < TRIM_THRESHOLD || pixels[idx+2] < TRIM_THRESHOLD)) {
        minX = Math.min(minX, x); maxX = Math.max(maxX, x);
        minY = Math.min(minY, y); maxY = Math.max(maxY, y);
      }
    }
  }
  // padding=10, 安全弁: areaRatio > 0.95 or < 0.1 → トリムしない
}
```

## Step 5: 最終ページ判断
1. 指定ページ数到達
2. ページ遷移失敗（ハッシュ比較で変化なし × 3回連続）
3. 奥付パターン検出（未実装）

## ルール
- スピナー消失 + 安定化待機（100-300ms）
- トリム失敗時は元画像にフォールバック
- `ImageBitmap.close()`, `canvas.width=0` でメモリ解放
- 5回連続失敗で全体中断
