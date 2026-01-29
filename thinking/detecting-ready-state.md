---
name: detecting-ready-state
description: MutationObserverを使ったローディング完了検知（スピナー消失・要素出現）のパターンを定義する。SPAや動的ページで準備完了状態を判定する実装時に使う。
skills: [scraping-async-dom]
---

# Ready State Detection with MutationObserver

動的なWebページにおいて、ローディング完了（準備完了状態）を確実に検知するためのパターン集。

## 課題

- SPAやAjax読み込みでは `DOMContentLoaded` や `load` イベントだけでは不十分。
- ローディングスピナーやプレースホルダーが消えるタイミングを検知する必要がある。
- 単純な `setTimeout` は不安定で、速すぎると未完了、遅すぎると無駄な待機。

## MutationObserver パターン

### 基本構造

```typescript
function waitForCondition(
  checkFn: () => boolean,
  timeout: number = 5000
): Promise<void> {
  return new Promise((resolve, reject) => {
    // 即時チェック
    if (checkFn()) {
      resolve();
      return;
    }

    const timerId = setTimeout(() => {
      observer.disconnect();
      reject(new Error(`Timeout: condition not met within ${timeout}ms`));
    }, timeout);

    const observer = new MutationObserver(() => {
      if (checkFn()) {
        observer.disconnect();
        clearTimeout(timerId);
        resolve();
      }
    });

    const target = document.body || document.documentElement;
    if (!target) {
      reject(new Error("No observable root element available"));
      return;
    }

    observer.observe(target, {
      childList: true,
      subtree: true,
      attributes: true,
      attributeFilter: ["style", "class"],
    });
  });
}
```

## スピナー消失パターン

ローディングスピナーが表示され、消えたら完了とみなすパターン。

```typescript
const SPINNER_SELECTOR = ".loader, .spinner, [data-loading]";
const SPINNER_APPEAR_WAIT = 300; // msスピナーが現れるのを待つ

function waitForSpinnerToDisappear(timeout: number = 5000): Promise<void> {
  return new Promise((resolve, reject) => {
    let seenSpinner = false;
    let appearTimerId: number | null = null;

    const timerId = window.setTimeout(() => {
      observer.disconnect();
      if (appearTimerId) clearTimeout(appearTimerId);
      reject(new Error("Timeout: spinner did not disappear"));
    }, timeout);

    const finish = () => {
      observer.disconnect();
      clearTimeout(timerId);
      if (appearTimerId) clearTimeout(appearTimerId);
      resolve();
    };

    const isSpinnerVisible = (): boolean => {
      const spinner = document.querySelector(SPINNER_SELECTOR);
      if (!spinner) return false;
      const style = getComputedStyle(spinner);
      return style.display !== "none" && style.visibility !== "hidden";
    };

    const observer = new MutationObserver(() => {
      if (isSpinnerVisible()) {
        seenSpinner = true;
        if (appearTimerId) {
          clearTimeout(appearTimerId);
          appearTimerId = null;
        }
      } else if (seenSpinner) {
        // スピナーが見えて、今は消えた = 完了
        finish();
      }
    });

    const target = document.body || document.documentElement;
    if (!target) {
      reject(new Error("No observable root element available"));
      return;
    }

    observer.observe(target, {
      childList: true,
      subtree: true,
      attributes: true,
      attributeFilter: ["style", "class"],
    });

    // 初期チェック
    if (isSpinnerVisible()) {
      seenSpinner = true;
    } else {
      // スピナーがまだ出ていない場合、少し待ってから判断
      appearTimerId = window.setTimeout(() => {
        if (!seenSpinner) {
          // スピナーが現れなかった = 既に完了状態
          finish();
        }
      }, SPINNER_APPEAR_WAIT);
    }
  });
}
```

## 要素出現パターン

特定の要素が出現したら準備完了とみなすパターン。

```typescript
function waitForElement(
  selector: string,
  timeout: number = 5000
): Promise<Element> {
  return new Promise((resolve, reject) => {
    const existing = document.querySelector(selector);
    if (existing) {
      resolve(existing);
      return;
    }

    const timerId = setTimeout(() => {
      observer.disconnect();
      reject(new Error(`Element not found: ${selector}`));
    }, timeout);

    const observer = new MutationObserver(() => {
      const el = document.querySelector(selector);
      if (el) {
        observer.disconnect();
        clearTimeout(timerId);
        resolve(el);
      }
    });

    observer.observe(document.body, { childList: true, subtree: true });
  });
}
```

## ベストプラクティス

- **必ずタイムアウトを設定**: 無限待機を避ける。
- **observer.disconnect() を忘れない**: メモリリークとパフォーマンス劣化の原因になる。
- **attributeFilter を使う**: 監視対象を絞ることでパフォーマンス向上。
- **Safety Delay**: クリック直後などは、スピナーが現れるまで100〜300msの猶予を設ける。
