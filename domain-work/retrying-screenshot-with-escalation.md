---
name: retrying-screenshot-with-escalation
description: スクリーンショット取得のExponential Backoffリトライと回復不可能エラー早期検出・連続失敗自動中断
---
# スクリーンショットリトライ＆エスカレーション

## 2層エスカレーション
```
[単一ページ] リトライ3回 (500ms間隔)
    ↓ 全失敗
[ループレベル] consecutiveFailures++
    ↓ 5回連続
[全体中断] finishCapture("error")

※ 回復不可能エラー → 即中断 (shouldAbort: true)
```

## 単一ページリトライ
```typescript
async function captureAndProcessPage(tabId: number): Promise<{ dataUrl: string | null; shouldAbort: boolean }> {
  for (let attempt = 0; attempt <= 3; attempt++) {
    try {
      if (attempt > 0) await delay(500);
      const raw = await captureScreenshotWithDebugger(tabId);
      return { dataUrl: await autoTrimImage(raw), shouldAbort: false };
    } catch (error) {
      const msg = error instanceof Error ? error.message : "";
      // 回復不可能: Debugger切断、タブ閉鎖、権限エラー
      if (msg.includes("Debugger is not attached") || msg.includes("No tab with given id") || msg.includes("Cannot access"))
        return { dataUrl: null, shouldAbort: true };
    }
  }
  return { dataUrl: null, shouldAbort: false }; // 連続失敗カウンタに委ねる
}
```

## 連続失敗トラッキング
- 成功時: `consecutiveFailures = 0`（リセット必須）
- 失敗時: `consecutiveFailures++`
- `MAX_CONSECUTIVE_FAILURES = 5` で全体中断
- **累積ではなく連続**（散発的失敗は許容）
