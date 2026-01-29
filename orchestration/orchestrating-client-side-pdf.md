---
name: orchestrating-client-side-pdf
description: jsPDFを使ったクライアントサイドPDF生成の実装パターンとメモリ管理戦略を定義する。ブラウザ内で画像ベースPDFを生成する実装時に使う。
skills: []
---

# Client-Side PDF Orchestration (jspdf)

ブラウザ側（Client-Side）だけで、大量の画像を含む高品質なPDFを生成するための技術選定と実装パターン。サーバーサイド処理を行わないため、プライバシー保護とコスト削減に有利だが、メモリ管理が最大の課題となる。

## 1. ライブラリ選定と設定

**推奨: `jspdf` (v2.5.1以上)**

軽量かつ日本語フォントの埋め込みなどが不要な「画像ベースPDF」作成においてはデファクトスタンダード。

```typescript
import { jsPDF } from "jspdf";

// A4縦 (Portrait), 単位mm
// ※ 画像サイズに合わせて動的に変更する場合は後述
const doc = new jsPDF({
  orientation: "p",
  unit: "mm",
  format: "a4",
  compress: true // 必須: ファイルサイズ削減
});
```

## 2. 画像の追加 (`addImage`) の最適解

`addImage` メソッドは強力だが、パラメータ選定を誤るとファイルサイズが肥大化するか、画質が劣化する。

### 推奨パラメータ

```typescript
interface PageData {
  dataUrl: string; // Base64 (image/jpeg推奨)
  width: number;
  height: number;
}

function addPageToPDF(doc: jsPDF, page: PageData) {
  const imgProps = doc.getImageProperties(page.dataUrl);
  
  // A4サイズ (210mm x 297mm) にフィットさせる計算
  const pdfWidth = 210;
  const pdfHeight = (imgProps.height * pdfWidth) / imgProps.width;
  
  // ページを追加 (最初のページは addPage 不要な場合があるため注意)
  // ここでは常に新規ページを追加するパターン
  doc.addPage([pdfWidth, pdfHeight]); 
  
  // 画像を描画
  // "FAST" = 圧縮処理をスキップして高速化（画質には影響小）
  // "MEDIUM" / "SLOW" = 圧縮率向上
  doc.addImage(
    page.dataUrl, 
    "JPEG", 
    0, 0, pdfWidth, pdfHeight, 
    undefined, 
    "FAST" 
  );
}
```

### 画質 vs サイズのトレードオフ

- **形式**: `PNG` は可逆圧縮で巨大になるため、写真は `JPEG` (品質0.75〜0.85) 推奨。漫画などの線画は `PNG` の方が綺麗な場合があるが、Kindleキャプチャの場合は `JPEG` で十分高品質。
- **解像度**: Retinaディスプレイ(2x, 3x) でキャプチャした画像をそのまま使うとPDFが巨大化する。目的（閲覧用なら画面サイズ、印刷用なら高解像度）に合わせて事前リサイズを検討する。

## 3. メモリ管理 (OOM対策)

Chrome拡張機能のService WorkerやContent Scriptにはメモリ制限がある。数百ページの画像を `jsPDF` インスタンスに突っ込むとクラッシュする。

### 戦略: バッチ処理とBlob化

全ページをオンメモリで保持せず、一定枚数ごとに処理するか、Base64文字列を早めに解放する。しかし `jsPDF` は内部で全データを保持して `save()` 時に結合するため、長編の場合は限界がある。

**Web Workerの使用 (推奨)**
メインスレッドをブロックしないためにも、PDF生成は Web Worker (または本来ならOffscreen Canvas等) に逃がすのが理想。

## 4. ダウンロード処理

作成したPDFをユーザーに保存させる。

```typescript
function savePDF(doc: jsPDF, filename: string) {
  // ブラウザのメモリ制限を回避するため、Blobとして出力
  const blob = doc.output("blob");
  const url = URL.createObjectURL(blob);
  
  chrome.downloads.download({
    url: url,
    filename: filename,
    saveAs: true // ユーザーに保存場所を選ばせる
  }, () => {
    // ダウンロード開始後、URLを解放
    setTimeout(() => URL.revokeObjectURL(url), 10000);
  });
}
```

## 5. チェックリスト

- [ ] 画像形式は JPEG (quality 0.8) を基本とするか
- [ ] `compress: true` は有効か
- [ ] アスペクト比を維持してリサイズしているか
- [ ] `chrome.downloads` API の権限はあるか
