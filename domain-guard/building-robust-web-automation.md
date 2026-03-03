---
name: building-robust-web-automation
description: 動的Webページでの堅牢なセレクタ探索・フォールバック・リトライ戦略
skills: [detecting-ready-state, scraping-async-dom]
---
# Robust Web Automation

## セレクタ探索（優先順）
1. データ属性/ARIA: `[aria-label="Next"]`, `[data-role="next-page"]`
2. テキスト一致: ボタン内ラベル走査
3. 座標ベース: 画面端のクリッカブル要素
4. フォールバック: 代替セレクタリストを順次試行

## キーボードフォールバック
DOM要素クリック失敗時、KeyEventをシミュレート。事前に `element.focus()` 必須。

```javascript
document.dispatchEvent(new KeyboardEvent('keydown', {
  key: 'ArrowRight', code: 'ArrowRight', keyCode: 39, bubbles: true
}));
```

## リトライ（指数バックオフ）

```typescript
async function retryWithBackoff<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  let delay = 100;
  for (let i = 0; i < maxRetries; i++) {
    try { return await fn(); }
    catch (e) {
      if (i === maxRetries - 1) throw e;
      await new Promise(res => setTimeout(res, delay));
      delay *= 2;
    }
  }
  throw new Error('Max retries exceeded');
}
```

## タイミング制御
- ポーリング待機: 要素出現まで一定間隔チェック
- ページ遷移検出: URL変化 or スピナー消滅をトリガー
- Safety Delay: `await delay(100)` でレンダリング待ち

## スクロール（仮想リスト/無限スクロール）
`window.scrollBy(0, viewportHeight)` を繰り返し、各ステップで要素収集。スクロール後は `IntersectionObserver` or 遅延で描画待ち。
