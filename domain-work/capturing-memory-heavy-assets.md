---
name: capturing-memory-heavy-assets
description: OffscreenCanvasによる大容量画像処理。段階的メモリ解放・CSP回避・ピクセルレベルコンテンツ検出。
---
# OffscreenCanvasによる大容量画像処理

Service Worker環境（DOMなし）でOffscreenCanvasを使い、スクリーンショットのトリミングとメモリ管理を行う。

## 全体構造（try-finally）
```typescript
async function autoTrimImage(dataUrl: string): Promise<string> {
  let bitmap: ImageBitmap | null = null;
  let canvas: OffscreenCanvas | null = null;
  let trimmedCanvas: OffscreenCanvas | null = null;
  try {
    const blob = dataUrlToBlob(dataUrl); // fetch不使用（CSP回避）
    bitmap = await createImageBitmap(blob);
    canvas = new OffscreenCanvas(bitmap.width, bitmap.height);
    const ctx = canvas.getContext("2d")!;
    ctx.drawImage(bitmap, 0, 0);
    const imageData = ctx.getImageData(0, 0, bitmap.width, bitmap.height);
    const bounds = findContentBounds(imageData.data, bitmap.width, bitmap.height);
    if (!bounds) return dataUrl;
    // トリミング・Blob変換・dataUrl返却...
  } catch { return dataUrl; } // 失敗時は元画像をそのまま返す
  finally {
    if (bitmap) { bitmap.close(); bitmap = null; }
    if (canvas) { canvas.width = 0; canvas.height = 0; canvas = null; }
    if (trimmedCanvas) { trimmedCanvas.width = 0; trimmedCanvas.height = 0; trimmedCanvas = null; }
  }
}
```

## CSP回避: dataUrl→Blob変換
```typescript
function dataUrlToBlob(dataUrl: string): Blob {
  const [header, base64] = dataUrl.split(",");
  const mime = header.match(/:(.*?);/)?.[1] ?? "image/png";
  const bin = atob(base64);
  const bytes = new Uint8Array(bin.length);
  for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i);
  return new Blob([bytes], { type: mime });
}
```
`fetch("data:...")` はCSPでブロックされる。`atob`+`Uint8Array`はCSP無関係。

## コンテンツ検出
- TRIM_THRESHOLD=250（ほぼ白を許容）
- `alpha > 10 && (r < 250 || g < 250 || b < 250)` → コンテンツ
- padding=10px追加
- 安全弁: areaRatio > 0.95（トリム不要）or < 0.1（検出ミス）→ 元画像返却

## メモリ解放ルール
- `ImageBitmap.close()` 必須（GCに頼らない）
- `OffscreenCanvas.width=0, height=0` でバッキングストア解放
- `getImageData()`のUint8ClampedArrayは使用後すぐにnull化
