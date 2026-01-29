---
name: building-robust-web-automation
description: 動的Webページでの堅牢なセレクタ探索・キーボードフォールバック・リトライ・スクロール操作の戦略を定義する。DOM操作が不安定な環境での自動化実装時に使う。
skills: [detecting-ready-state, scraping-async-dom]
---

# Robust Web Automation & Interaction

動的なウェブページや、セレクタが頻繁に変わる環境（Kindle等）において、安定して要素を操作するためのテクニック。

## 戦略 A: 堅牢なセレクタ探索 (Heuristics)

IDやクラス名に頼りすぎず、以下の属性を組み合わせて目的の要素を特定する。

1. **データ属性 / ARIA属性**: `[aria-label="Next"]`, `[data-role="next-page"]`
2. **テキスト一致**: ボタン内のラベルテキストを走査する。
3. **座標ベース**: 画面の右端/左端にある「クリッカブルな要素」を特定する。
4. **フォールバック**: 第一候補が見つからない場合、代替セレクタのリストを順次試す。

## 戦略 B: キーボード・フォールバック

DOM要素のクリックが失敗、または要素が見つからない場合、物理的なキー入力（KeyEvent）をシミュレートする。

```javascript
const event = new KeyboardEvent('keydown', {
  key: 'ArrowRight',
  code: 'ArrowRight',
  keyCode: 39,
  bubbles: true
});
document.dispatchEvent(event);
```

**注意**: キーボードイベントは、対象要素またはドキュメントにフォーカスがあたっていないと反応しないことがある。事前に `element.focus()` を呼ぶか、`document.body.focus()` でフォーカスを確保すること。

### リトライ戦略 (Retry with Exponential Backoff)

要素が見つからない、またはクリックが失敗した場合は、指数バックオフでリトライする。

```typescript
async function retryWithBackoff<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  let delay = 100;
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (e) {
      if (i === maxRetries - 1) throw e;
      await new Promise(res => setTimeout(res, delay));
      delay *= 2; // 指数バックオフ
    }
  }
  throw new Error('Max retries exceeded');
}
```

## 戦略 C: タイミング制御

- **ポーリング待機**: 要素が出現するまで、一定間隔でチェックを繰り返す。
- **ページ遷移検出**: URLの変化や、ローディング・インジケーター（Spinner）の消滅をトリガーにする。
- **微調整（Safety Delay）**: ブラウザのレンダリングや内部状態の更新を待つために、100ms程度の短い `await delay(100)` を戦略的に挟む。

## 戦略 D: スクロール操作 (Virtual Lists / Lazy Loading)

仮想リスト（表示範囲外のDOMが破棄される）や無限スクロールにおいて、全ての要素を網羅するための手法。

1. **逐次スクロール**: `window.scrollBy(0, viewportHeight)` を繰り返し、各ステップで要素を収集する。
2. **scrollIntoView**: 目的の要素が分かっている場合は `el.scrollIntoView({ behavior: 'smooth', block: 'center' })` を使用。
3. **描画待ち**: スクロール後、コンテンツがレンダリングされるまで `IntersectionObserver` や一定の遅延で待機する。

### 全ページキャプチャのためのスクロール実装例

```javascript
async function scrollToBottom() {
  const distance = 100; // 1回のスクロール量
  const delay = 100;    // 待機時間
  while (window.scrollY + window.innerHeight < document.documentElement.scrollHeight) {
    window.scrollBy(0, distance);
    await new Promise(res => setTimeout(res, delay));
  }
}
```

## 推奨実装パターン

```typescript
async function smartClick(selector: string, fallbackKey?: string) {
  const element = document.querySelector(selector);
  if (element) {
    (element as HTMLElement).click();
    return true;
  }
  if (fallbackKey) {
    // Dispatch keyboard event...
    return true;
  }
  return false;
}
```
