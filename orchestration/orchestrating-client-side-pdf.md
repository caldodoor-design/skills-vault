---
name: orchestrating-client-side-pdf
description: jsPDFによるクライアントサイドPDF生成パターン
---
# Client-Side PDF (jsPDF)

## 基本設定
```typescript
import { jsPDF } from "jspdf";
const doc = new jsPDF({ orientation: "p", unit: "mm", format: "a4", compress: true });
```

## 画像追加
```typescript
function addPageToPDF(doc: jsPDF, page: { dataUrl: string; width: number; height: number }) {
  const imgProps = doc.getImageProperties(page.dataUrl);
  const pdfWidth = 210;
  const pdfHeight = (imgProps.height * pdfWidth) / imgProps.width;
  doc.addPage([pdfWidth, pdfHeight]);
  doc.addImage(page.dataUrl, "JPEG", 0, 0, pdfWidth, pdfHeight, undefined, "FAST");
}
```

- 形式: 写真は JPEG (quality 0.75~0.85)、線画はPNGも可
- Retinaキャプチャはそのまま使うと巨大化 → 目的に合わせてリサイズ

## メモリ管理（OOM対策）
全ページオンメモリ保持は危険。バッチ処理 or Web Workerに逃がす。

## ダウンロード
```typescript
const blob = doc.output("blob");
const url = URL.createObjectURL(blob);
chrome.downloads.download({ url, filename, saveAs: true }, () => {
  setTimeout(() => URL.revokeObjectURL(url), 10000);
});
```
