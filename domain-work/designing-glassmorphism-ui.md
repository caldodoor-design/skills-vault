---
name: designing-glassmorphism-ui
description: すりガラス風UIの実装パターン
trigger: Glassmorphismデザインの実装時
---

# Glassmorphism UI System

## CSS変数
```css
:root {
  --glass-bg: rgba(255, 255, 255, 0.15);
  --glass-border: rgba(255, 255, 255, 0.2);
  --glass-shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.37);
  --bg-gradient: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}
```

## コンテナ
```css
.container {
  background: var(--glass-bg);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border: 1px solid var(--glass-border);
  box-shadow: var(--glass-shadow);
  border-radius: 16px;
}
```

## ポイント
- ホバー時に`translateY`+`box-shadow`変化で浮遊感
- pulse/blinkアニメーションで「生きている」感覚
- `backdrop-filter`未対応ブラウザ用に背景色フォールバック必須
