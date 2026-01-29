---
name: scraping-async-dom
description: 非同期DOMの待機パターン（waitForSelector/waitForStability等）を定義する。動的ページからデータ取得する実装時に使う。
skills: [detecting-ready-state]
---

# DOM待機スクレイピング ベストプラクティス

## 目的

非同期で読み込まれるDOM要素を安定して取得するための待機パターンを定義する。

---

## 禁止パターン

### ❌ 固定時間待機 (setTimeout)

```typescript
// 禁止: 固定時間待機
await new Promise(resolve => setTimeout(resolve, 2000));
const element = document.querySelector('.target');
```

**理由:**
- ネットワーク速度やデバイス性能により必要な待機時間が変動する
- 短すぎると要素が見つからない
- 長すぎると無駄な待機時間が発生する
- 環境依存でテストが不安定になる

---

## 推奨パターン

### ✅ waitForSelector: セレクタ出現待機

指定したセレクタの要素が出現するまで待機する。

```typescript
function waitForSelector(
  selector: string,
  options: {
    timeout?: number;
    root?: Element | Document;
  } = {}
): Promise<Element> {
  const { timeout = 10000, root = document } = options;

  return new Promise((resolve, reject) => {
    // 既に存在する場合は即座に返す
    const existing = root.querySelector(selector);
    if (existing) {
      resolve(existing);
      return;
    }

    const observer = new MutationObserver((mutations, obs) => {
      const element = root.querySelector(selector);
      if (element) {
        obs.disconnect();
        resolve(element);
      }
    });

    observer.observe(root, {
      childList: true,
      subtree: true,
    });

    // タイムアウト処理
    setTimeout(() => {
      observer.disconnect();
      reject(new Error(`Timeout: selector "${selector}" not found within ${timeout}ms`));
    }, timeout);
  });
}
```

### ✅ waitForDisappearance: 要素消失待機

指定したセレクタの要素が消えるまで待機する（ローディングスピナー等）。

```typescript
function waitForDisappearance(
  selector: string,
  options: {
    timeout?: number;
    root?: Element | Document;
  } = {}
): Promise<void> {
  const { timeout = 10000, root = document } = options;

  return new Promise((resolve, reject) => {
    // 既に存在しない場合は即座に返す
    const existing = root.querySelector(selector);
    if (!existing) {
      resolve();
      return;
    }

    const observer = new MutationObserver((mutations, obs) => {
      const element = root.querySelector(selector);
      if (!element) {
        obs.disconnect();
        resolve();
      }
    });

    observer.observe(root, {
      childList: true,
      subtree: true,
    });

    // タイムアウト処理
    setTimeout(() => {
      observer.disconnect();
      reject(new Error(`Timeout: selector "${selector}" still exists after ${timeout}ms`));
    }, timeout);
  });
}
```

### ✅ waitForStability: DOM安定待機

DOM変化が止まるまで待機する。ページ全体の読み込み完了判定に使用。

```typescript
function waitForStability(
  options: {
    stableTime?: number;  // 変化がない状態が続く時間
    timeout?: number;
    root?: Element | Document;
  } = {}
): Promise<void> {
  const { stableTime = 500, timeout = 10000, root = document } = options;

  return new Promise((resolve, reject) => {
    let lastMutationTime = Date.now();
    let checkInterval: ReturnType<typeof setInterval>;

    const observer = new MutationObserver(() => {
      lastMutationTime = Date.now();
    });

    observer.observe(root, {
      childList: true,
      subtree: true,
      attributes: true,
      characterData: true,
    });

    // 定期的に安定性をチェック
    checkInterval = setInterval(() => {
      const elapsed = Date.now() - lastMutationTime;
      if (elapsed >= stableTime) {
        observer.disconnect();
        clearInterval(checkInterval);
        resolve();
      }
    }, 100);

    // タイムアウト処理
    setTimeout(() => {
      observer.disconnect();
      clearInterval(checkInterval);
      reject(new Error(`Timeout: DOM did not stabilize within ${timeout}ms`));
    }, timeout);
  });
}
```

---

## 複合パターン

### ページ遷移後のコンテンツ待機

```typescript
async function waitForPageContent(contentSelector: string): Promise<Element> {
  // 1. ローディングスピナーが消えるまで待機
  await waitForDisappearance('.loading-spinner');

  // 2. コンテンツが出現するまで待機
  const content = await waitForSelector(contentSelector);

  // 3. DOMが安定するまで待機
  await waitForStability({ stableTime: 300 });

  return content;
}
```

### リトライ付きセレクタ待機

```typescript
async function waitForSelectorWithRetry(
  selector: string,
  maxRetries: number = 3,
  retryDelay: number = 1000
): Promise<Element> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await waitForSelector(selector, { timeout: 5000 });
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(r => setTimeout(r, retryDelay));
    }
  }
  throw new Error('Unreachable');
}
```

---

## 使用時の注意点

1. **タイムアウトは必ず設定する**: 無限待機はアプリケーションをハングさせる
2. **適切なrootを指定する**: iframe内の要素は `iframe.contentDocument` を指定
3. **observerは必ずdisconnectする**: メモリリーク防止
4. **エラーハンドリングを忘れない**: タイムアウト時の適切な処理を実装する
