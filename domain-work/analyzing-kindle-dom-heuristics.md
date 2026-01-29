---
name: analyzing-kindle-dom-heuristics
description: Kindle Cloud Readerのページ構造解析・RTL/LTR判定・漫画/テキストモード判別・UI除外のヒューリスティックを定義する。Kindleコンテンツの自動操作やキャプチャ実装時に使う。
skills: [reading-kindle-dom, detecting-ready-state]
---

# Kindle DOM Heuristics & Strategies

Kindle Cloud Reader (read.amazon.co.jp) の複雑なDOM構造を攻略し、安定してコンテンツを特定・操作するためのヒューリスティック集。

## 1. キャンバスとページ構造

Kindle Cloud Readerは、書籍の内容を `<canvas>` 要素として描画する場合と、画像 (`<img>`) または背景画像として表示する場合がある。

### 主要セレクタ戦略

iframe内のコンテンツ（Sandboxed環境など）と、メインウィンドウ内のコンテンツの両方を探索する。

```typescript
const PAGE_SELECTORS = [
  "#kindleReader_pageHost",       // 一般的なコンテナ
  ".canvas_container",            // キャンバスモード
  "[id^='page_']"                 // ページID (page_1, page_2...)
];

function findMainContentArea(): HTMLElement | null {
  // 1. iframe内の探索を優先
  const iframes = document.querySelectorAll("iframe");
  for (const iframe of Array.from(iframes)) {
    try {
      const doc = iframe.contentDocument;
      if (doc) {
        const canvas = doc.querySelector("canvas");
        if (canvas) return canvas;
      }
    } catch (e) {
      // Cross-origin制限は無視
    }
  }

  // 2. メインウィンドウ内の探索
  for (const selector of PAGE_SELECTORS) {
    const el = document.querySelector<HTMLElement>(selector);
    if (el) return el;
  }
  
  return null;
}
```

## 2. ページめくり方向 (RTL/LTR) の判定

漫画や縦書き小説（右開き）と、横書き技術書（左開き）を自動判別する。

### 判定ロジック

信頼性の高い順に判定を行う。

1. **計算済みスタイル (getComputedStyle)**: 親コンテナの direction や writing-mode を確認する。これが最も確実。
2. **ヒューリスティック**: "next" ボタンの位置やラベルを利用する。

```typescript
function detectReadingDirection(): 'ltr' | 'rtl' {
  // 1. 親コンテナのスタイルを確認（高信頼度）
  const pageHost = document.getElementById("kindleReader_pageHost");
  if (pageHost) {
    const style = getComputedStyle(pageHost);
    if (style.direction === "ltr") return "ltr";
    if (style.direction === "rtl") return "rtl";
    // 縦書き (vertical-rl) なら RTL とみなす
    if (style.writingMode && style.writingMode.includes("vertical")) return "rtl";
  }

  // 2. フォールバック: 位置による判定
  const rightArea = document.getElementById("kindleReader_pageTurnAreaRight");
  const leftArea = document.getElementById("kindleReader_pageTurnAreaLeft");
  
  const rightLabel = (rightArea?.getAttribute("aria-label") || "").toLowerCase();
  const leftLabel = (leftArea?.getAttribute("aria-label") || "").toLowerCase();
  
  if (rightLabel.includes("next")) return 'ltr';
  if (leftLabel.includes("next")) return 'rtl';
  
  // デフォルトは RTL（日本のストアでは漫画が多いため）
  return 'rtl'; 
}
```

## 3. 漫画モード vs テキストモード

| モード | 特徴 | キャプチャ戦略 |
|--------|------|----------------|
| **漫画/雑誌** | 固定レイアウト、画像ベース | スクリーンショット (captureVisibleTab) が最適 |
| **テキスト** | リフロー型、文字選択可能 | スクショ推奨、フォントレンダリング待ちが必要 |

## 4. UI除外（トリミング）

キャプチャ時にヘッダー、フッター、サイドバーが映り込まないようにする。

- **推奨: CSS opacity** - `display: none` はレイアウト再計算を引き起こすため、`opacity: 0` で見えなくする。

```css
/* 注入する一時的スタイル */
#kindleReader_header,
#kindleReader_footer,
.kindle_scrollbar {
  opacity: 0 !important;
  pointer-events: none !important;
}
```

## 5. ページ遷移完了の検知

- **スピナー**: `.loading_spinner`, `#kindleReader_spinner` の消失を待つ
- **Canvas描画**: `canvas.toDataURL()` のハッシュ値変化を監視