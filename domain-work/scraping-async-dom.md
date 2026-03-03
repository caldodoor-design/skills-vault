---
name: scraping-async-dom
description: 非同期DOMの待機パターン（waitForSelector/waitForDisappearance/waitForStability）
skills: [detecting-ready-state]
---
# DOM待機パターン

**固定時間待機（setTimeout）は禁止。** MutationObserverベースの待機を使う。

## waitForSelector: 要素出現待機
```typescript
function waitForSelector(selector: string, { timeout = 10000, root = document } = {}): Promise<Element> {
  return new Promise((resolve, reject) => {
    const existing = root.querySelector(selector);
    if (existing) { resolve(existing); return; }
    const obs = new MutationObserver(() => {
      const el = root.querySelector(selector);
      if (el) { obs.disconnect(); resolve(el); }
    });
    obs.observe(root, { childList: true, subtree: true });
    setTimeout(() => { obs.disconnect(); reject(new Error(`Timeout: "${selector}" not found within ${timeout}ms`)); }, timeout);
  });
}
```

## waitForDisappearance: 要素消失待機（スピナー等）
```typescript
function waitForDisappearance(selector: string, { timeout = 10000, root = document } = {}): Promise<void> {
  return new Promise((resolve, reject) => {
    if (!root.querySelector(selector)) { resolve(); return; }
    const obs = new MutationObserver(() => {
      if (!root.querySelector(selector)) { obs.disconnect(); resolve(); }
    });
    obs.observe(root, { childList: true, subtree: true });
    setTimeout(() => { obs.disconnect(); reject(new Error(`Timeout: "${selector}" still exists after ${timeout}ms`)); }, timeout);
  });
}
```

## waitForStability: DOM安定待機
```typescript
function waitForStability({ stableTime = 500, timeout = 10000, root = document } = {}): Promise<void> {
  return new Promise((resolve, reject) => {
    let lastMutation = Date.now();
    const obs = new MutationObserver(() => { lastMutation = Date.now(); });
    obs.observe(root, { childList: true, subtree: true, attributes: true, characterData: true });
    const check = setInterval(() => {
      if (Date.now() - lastMutation >= stableTime) { obs.disconnect(); clearInterval(check); resolve(); }
    }, 100);
    setTimeout(() => { obs.disconnect(); clearInterval(check); reject(new Error(`DOM did not stabilize within ${timeout}ms`)); }, timeout);
  });
}
```

## 複合パターン
```typescript
async function waitForPageContent(contentSelector: string): Promise<Element> {
  await waitForDisappearance('.loading-spinner');
  const content = await waitForSelector(contentSelector);
  await waitForStability({ stableTime: 300 });
  return content;
}
```

## 注意点
- タイムアウトは必ず設定（無限待機禁止）
- iframe内は `iframe.contentDocument` を指定
- observerは必ずdisconnect（メモリリーク防止）
