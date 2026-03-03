---
name: detecting-ready-state
description: MutationObserverでローディング完了を検知するパターン集
skills: [scraping-async-dom]
---
# Ready State Detection

## waitForCondition（汎用）
```typescript
function waitForCondition(checkFn: () => boolean, timeout = 5000): Promise<void> {
  return new Promise((resolve, reject) => {
    if (checkFn()) { resolve(); return; }
    const timerId = setTimeout(() => { observer.disconnect(); reject(new Error('Timeout')); }, timeout);
    const observer = new MutationObserver(() => {
      if (checkFn()) { observer.disconnect(); clearTimeout(timerId); resolve(); }
    });
    observer.observe(document.body || document.documentElement, {
      childList: true, subtree: true, attributes: true, attributeFilter: ["style", "class"]
    });
  });
}
```

## スピナー消失パターン
```typescript
const SPINNER_SELECTOR = ".loader, .spinner, [data-loading]";
const SPINNER_APPEAR_WAIT = 300;
```
1. MutationObserverでスピナー出現を検知 → `seenSpinner = true`
2. スピナー消失を検知 → resolve
3. `SPINNER_APPEAR_WAIT` 内にスピナー未出現 → 既に完了と判断

## waitForElement
```typescript
function waitForElement(selector: string, timeout = 5000): Promise<Element> {
  return new Promise((resolve, reject) => {
    const existing = document.querySelector(selector);
    if (existing) { resolve(existing); return; }
    const timerId = setTimeout(() => { observer.disconnect(); reject(new Error(`Not found: ${selector}`)); }, timeout);
    const observer = new MutationObserver(() => {
      const el = document.querySelector(selector);
      if (el) { observer.disconnect(); clearTimeout(timerId); resolve(el); }
    });
    observer.observe(document.body, { childList: true, subtree: true });
  });
}
```

## ベストプラクティス
- 必ずタイムアウト設定、`observer.disconnect()` 忘れずに
- `attributeFilter` で監視対象を絞る
- クリック直後は100-300ms猶予（Safety Delay）
