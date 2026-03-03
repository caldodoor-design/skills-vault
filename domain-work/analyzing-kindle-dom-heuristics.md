---
name: analyzing-kindle-dom-heuristics
description: Kindle Cloud Readerのページ構造解析・RTL/LTR判定・漫画/テキスト判別・UI除外ヒューリスティック
skills: [reading-kindle-dom, detecting-ready-state]
---
# Kindle DOM Heuristics

## 1. コンテンツ探索
```typescript
const PAGE_SELECTORS = ["#kindleReader_pageHost", ".canvas_container", "[id^='page_']"];
// iframe内を優先探索 → メインウィンドウ内探索
```

## 2. RTL/LTR判定（優先順）
```typescript
function detectReadingDirection(): 'ltr' | 'rtl' {
  // 1. getComputedStyle: direction, writing-mode (vertical-rl → RTL)
  // 2. ページめくりエリアのaria-label ("next"の位置)
  // 3. デフォルト: RTL（日本ストアは漫画が多い）
}
```

## 3. 漫画 vs テキスト
| モード | 特徴 | キャプチャ |
|--------|------|-----------|
| 漫画/雑誌 | 固定レイアウト、画像ベース | captureVisibleTab |
| テキスト | リフロー型、文字選択可能 | スクショ推奨、フォントレンダリング待ち必要 |

## 4. UI除外
```css
#kindleReader_header, #kindleReader_footer, .kindle_scrollbar {
  opacity: 0 !important; /* display:noneはレイアウト再計算を引き起こす */
  pointer-events: none !important;
}
```

## 5. 遷移完了検知
- スピナー: `.loading_spinner`, `#kindleReader_spinner` の消失
- Canvas: `canvas.toDataURL()` のハッシュ値変化監視
