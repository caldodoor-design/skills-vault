---
name: capturing-memory-heavy-assets
description: OffscreenCanvasを用いた大容量画像のメインスレッド非ブロック処理と、段階的メモリ解放・CSP回避・ピクセルレベルコンテンツ検出のパターン。
trigger: Service WorkerやWorkerでスクリーンショットや画像を処理するとき、またはCanvas操作でメモリリークを防ぎたいとき
---

# OffscreenCanvasによる大容量画像処理

## Purpose

Chrome拡張のService Workerにはwindowオブジェクトがなく、通常のCanvasやdocument.createElementが使えない。OffscreenCanvasを使ってピクセル操作を行い、大容量画像（スクリーンショット）のトリミング処理を実現する。同時に、メモリリーク防止とCSP制約回避を行う。

## Context

- Service Worker環境（DOMなし、window.documentなし）
- 連続スクリーンショットキャプチャで数十〜数百枚の画像を処理
- 各画像はフルスクリーンPNG（数MB）
- メモリを適切に解放しないとService Workerがクラッシュする
- Chrome拡張のCSPでfetch("data:...")が制限される場合がある

## Pattern

### 全体構造（try-finally による確実なクリーンアップ）

```typescript
async function autoTrimImage(dataUrl: string): Promise<string> {
  // 変数を外側で宣言（finallyで確実に解放するため）
  let bitmap: ImageBitmap | null = null;
  let canvas: OffscreenCanvas | null = null;
  let trimmedCanvas: OffscreenCanvas | null = null;

  try {
    // === Phase 1: デコード ===
    const blob = dataUrlToBlob(dataUrl); // fetch不使用（CSP回避）
    bitmap = await createImageBitmap(blob);

    // === Phase 2: ピクセル解析 ===
    canvas = new OffscreenCanvas(bitmap.width, bitmap.height);
    let ctx = canvas.getContext("2d");
    ctx.drawImage(bitmap, 0, 0);
    const imageData = ctx.getImageData(0, 0, bitmap.width, bitmap.height);
    let pixels = imageData.data;
    const bounds = findContentBounds(pixels, bitmap.width, bitmap.height);

    // 早期メモリ解放：解析完了後すぐに不要なデータをnull化
    pixels = null;
    ctx = null;

    if (!bounds) return dataUrl;

    // === Phase 3: トリミング ===
    trimmedCanvas = new OffscreenCanvas(bounds.width, bounds.height);
    let trimmedCtx = trimmedCanvas.getContext("2d");
    trimmedCtx.drawImage(bitmap, bounds.x, bounds.y, bounds.width, bounds.height,
                         0, 0, bounds.width, bounds.height);

    // bitmap解放：drawImage完了後すぐに
    bitmap.close();
    bitmap = null;

    const trimmedBlob = await trimmedCanvas.convertToBlob({ type: "image/png" });
    const trimmedDataUrl = await blobToDataUrl(trimmedBlob);
    trimmedCtx = null;

    return trimmedDataUrl;
  } catch (error) {
    console.warn("[Background] Auto-trim failed, using original:", error);
    return dataUrl; // 失敗時は元画像をそのまま返す（処理を止めない）
  } finally {
    // === 確実なクリーンアップ ===
    if (bitmap) {
      bitmap.close(); // ImageBitmap専用の解放メソッド
      bitmap = null;
    }
    if (canvas) {
      canvas.width = 0;  // バッキングストアを解放
      canvas.height = 0;
      canvas = null;
    }
    if (trimmedCanvas) {
      trimmedCanvas.width = 0;
      trimmedCanvas.height = 0;
      trimmedCanvas = null;
    }
  }
}
```

### CSP回避：fetchの代わりにatobでdataUrl→Blob変換

```typescript
function dataUrlToBlob(dataUrl: string): Blob {
  const [header, base64Data] = dataUrl.split(",");
  const mimeMatch = header.match(/:(.*?);/);
  const mimeType = mimeMatch ? mimeMatch[1] : "image/png";

  const binaryString = atob(base64Data);
  const bytes = new Uint8Array(binaryString.length);
  for (let i = 0; i < binaryString.length; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }
  return new Blob([bytes], { type: mimeType });
}
```

**なぜfetchを避けるか**: Chrome拡張のService WorkerではCSPにより`fetch("data:image/png;base64,...")`が失敗する場合がある。`atob`+`Uint8Array`は純粋なJavaScript処理でCSPの影響を受けない。

### ピクセルレベルコンテンツ検出（白地トリミング）

```typescript
function findContentBounds(
  pixels: Uint8ClampedArray, width: number, height: number
): BoundingBox | null {
  let minX = width, minY = height, maxX = 0, maxY = 0;
  let hasContent = false;

  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      const idx = (y * width + x) * 4;
      const r = pixels[idx], g = pixels[idx + 1];
      const b = pixels[idx + 2], a = pixels[idx + 3];

      // 白でも透明でもないピクセルをコンテンツとみなす
      const isContent = a > 10 &&
        (r < TRIM_THRESHOLD || g < TRIM_THRESHOLD || b < TRIM_THRESHOLD);

      if (isContent) {
        hasContent = true;
        if (x < minX) minX = x;
        if (x > maxX) maxX = x;
        if (y < minY) minY = y;
        if (y > maxY) maxY = y;
      }
    }
  }
  if (!hasContent) return null;

  // パディング追加（コンテンツが端ギリギリにならないように）
  const padding = 10;
  return {
    x: Math.max(0, minX - padding),
    y: Math.max(0, minY - padding),
    width: Math.min(width, maxX - minX + 1 + padding * 2),
    height: Math.min(height, maxY - minY + 1 + padding * 2),
  };
}
```

### トリミング判定の安全弁

```typescript
const areaRatio = trimmedArea / originalArea;
if (areaRatio > 0.95) return dataUrl;       // 5%未満のトリムは無意味
if (areaRatio < MIN_CONTENT_RATIO) return dataUrl; // 極端に小さい→検出ミスの可能性
```

## Rules

- `ImageBitmap`は必ず`.close()`で明示的に解放すること（GCに頼らない）
- `OffscreenCanvas`はwidth/heightを0にセットしてバッキングストアを解放すること
- try-finallyで変数をnull化し、正常系・異常系両方でクリーンアップすること
- `getImageData()`の返すUint8ClampedArrayは使用後すぐにnull化すること（巨大配列）
- トリミング処理が失敗しても元画像をそのまま返すこと（キャプチャを止めない）

## Anti-Patterns

- **fetch("data:...")でdataUrlをBlobに変換**: CSPでブロックされる可能性がある
- **bitmap.close()を忘れる**: ImageBitmapはGCで回収されにくく、メモリリークする
- **canvas変数をnull化するだけでwidth/heightを0にしない**: バッキングストアが残る
- **ピクセル検出でアルファチャネルを無視**: 透明ピクセルをコンテンツとして扱ってしまう
- **トリミング面積の妥当性チェックを省略**: 検出ミスで画像が極端に小さくなる

## Notes

- `OffscreenCanvas`はService Worker, Web Worker, メインスレッド全てで使用可能
- `createImageBitmap()`はBlobからデコードする際にメインスレッドをブロックしない
- TRIM_THRESHOLD=250は「ほぼ白」を許容する値。Kindleの背景色に合わせて調整
- 連続キャプチャでは毎回のメモリ解放が累積して大きな差になる
