---
name: designing-glassmorphism-ui
description: すりガラス風（Glassmorphism）UIのCSS変数・backdrop-filter・アニメーションの実装パターンを定義する。モダンでプレミアム感のあるUI構築時に使う。
skills: []
---

# Modern Glassmorphism UI System

モダンでプレミアム感のある「すりガラス（Glassmorphism）」風のインターフェースをVanilla CSSで構築するためのデザインシステム。

## 基本コンセプト

- **半透明とぼかし**: `backdrop-filter: blur()` を使用。
- **階層化**: 背景グラデーション、すりガラス、境界光。
- **鮮快なカラー**: HSLベースの調和のとれたパレット。

## デザインスタック (CSS変数例)

```css
:root {
  --glass-bg: rgba(255, 255, 255, 0.15);
  --glass-border: rgba(255, 255, 255, 0.2);
  --glass-shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.37);
  --bg-gradient: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}
```

## 実装のポイント

### 1. コンテナの定義
背景を透過させつつ、ぼかしを加えることで文字の可読性を確保する。

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

### 2. インタラクティブ要素
ホバー時に `translateY` や `box-shadow` を変化させ、「浮遊感」を演出する。

### 3. ステータス・インジケーター
脈動するアニメーション（pulse）や点滅（blink）を組み合わせ、アプリが「生きている」感覚を与える。

```css
@keyframes pulse {
  0% { transform: scale(0.95); opacity: 0.7; }
  50% { transform: scale(1); opacity: 1; }
  100% { transform: scale(0.95); opacity: 0.7; }
}
```

## 注意点

- `backdrop-filter` は古いブラウザで未対応の場合があるため、背景色のみのフォールバックを用意すること。
- 背景のグラデーションは、コントラストが強すぎないものを選ぶと安っぽさが消える。
