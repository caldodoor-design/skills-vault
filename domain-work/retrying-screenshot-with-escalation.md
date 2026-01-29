---
name: retrying-screenshot-with-escalation
description: スクリーンショット取得失敗時のExponential Backoffリトライと、回復不可能エラーの早期検出・連続失敗による自動中断戦略。
trigger: スクリーンショット取得・外部API呼び出しなど、一時的失敗と恒久的失敗を区別する必要があるリトライ処理を実装するとき。
---

# スクリーンショット取得のリトライとエスカレーション戦略

## Purpose

スクリーンショット取得は、タブ状態・Debugger接続・レンダリングタイミングなど複数の要因で失敗しうる。一時的な失敗にはリトライで対応し、回復不可能な失敗には即座に中断することで、無駄な待機を防ぎつつ信頼性を確保する。

## Context

Chrome拡張のBackground Service Workerからchrome.debugger APIでスクリーンショットを取得する場面。以下の3種類の失敗が発生する:
1. **一時的失敗**: レンダリング未完了、タイミングずれ → リトライで回復可能
2. **回復不可能な失敗**: Debugger切断、タブ閉鎖 → リトライしても無駄
3. **慢性的な失敗**: 原因不明だが連続で失敗 → 一定回数で全体を中断

## Pattern

### 1. 単一ページのリトライ（Exponential Backoff）

```typescript
const SCREENSHOT_RETRY_COUNT = 3;
const SCREENSHOT_RETRY_DELAY_MS = 500;

async function captureAndProcessPage(tabId: number): Promise<{ dataUrl: string | null; shouldAbort: boolean }> {
  let lastError: string = "Unknown error";

  for (let attempt = 0; attempt <= SCREENSHOT_RETRY_COUNT; attempt++) {
    try {
      if (attempt > 0) {
        console.log(`[Background] Screenshot retry ${attempt}/${SCREENSHOT_RETRY_COUNT}...`);
        await delay(SCREENSHOT_RETRY_DELAY_MS);
      }

      const rawDataUrl = await captureScreenshotWithDebugger(tabId);
      const processedDataUrl = await autoTrimImage(rawDataUrl);
      return { dataUrl: processedDataUrl, shouldAbort: false };
    } catch (error) {
      lastError = error instanceof Error ? error.message : "Unknown error";
      console.error(`[Background] Capture attempt ${attempt + 1} failed:`, lastError);

      // 回復不可能エラーの早期判定
      if (
        lastError.includes("Debugger is not attached") ||
        lastError.includes("No tab with given id") ||
        lastError.includes("Cannot access")
      ) {
        console.error("[Background] Unrecoverable error, aborting capture");
        return { dataUrl: null, shouldAbort: true };
      }
    }
  }

  // 全リトライ失敗 → shouldAbort: false（連続失敗カウンタに委ねる）
  return { dataUrl: null, shouldAbort: false };
}
```

**設計判断**: `shouldAbort`フラグで呼び出し元に判断を委ねる。回復不可能なら`true`で即中断、リトライ切れなら`false`で連続失敗カウンタに任せる。

### 2. 連続失敗トラッキング（エスカレーション）

```typescript
const MAX_CONSECUTIVE_FAILURES = 5;

// captureLoop内
if (captureResult.dataUrl) {
  await setState({
    capturedImages: [...latestState.capturedImages, captureResult.dataUrl],
    currentPage: latestState.currentPage + 1,
    consecutiveFailures: 0, // 成功時にリセット
  });
} else {
  const newFailureCount = latestState.consecutiveFailures + 1;

  if (newFailureCount >= MAX_CONSECUTIVE_FAILURES) {
    await finishCapture("error", `Capture aborted after ${MAX_CONSECUTIVE_FAILURES} consecutive failures`);
    return;
  }

  await setState({ consecutiveFailures: newFailureCount });
}
```

**なぜ連続失敗か**: 散発的な失敗（例: 100ページ中3ページ失敗）は許容したい。しかし5回連続で失敗するなら、環境自体に問題がある可能性が高い。

### 3. 2層のエスカレーション構造

```
[単一ページ]  リトライ3回 (500ms間隔)
    ↓ 全失敗
[ループレベル] consecutiveFailures++
    ↓ 5回連続
[全体中断]    finishCapture("error", ...)

※ 回復不可能エラー → ループレベルを飛ばして即中断 (shouldAbort: true)
```

## Rules

- リトライ対象は一時的失敗のみ。回復不可能エラーはエラーメッセージの文字列マッチで判定し、即座にループを抜ける。
- 成功時に`consecutiveFailures`を必ず0にリセットする。散発的失敗を不当にカウントしない。
- `shouldAbort`フラグで単一ページレベルとループレベルの責務を分離する。

## Anti-Patterns

- 全エラーを同一視してリトライする（Debugger切断後に3回待つのは無駄）
- 累積失敗数でカウントする（100ページ中5ページ失敗は正常動作の範囲内）
- リトライ内で状態を変更する（リトライは副作用なしで完結すべき）
- `shouldAbort`を使わず例外で制御フローを管理する（呼び出し元が適切に判断できなくなる）

## Notes

- `SCREENSHOT_RETRY_DELAY_MS = 500`は固定遅延。現実装ではExponential Backoffではなく等間隔リトライだが、Chrome Debugger APIの応答特性上これで十分。
- 回復不可能エラーの判定文字列はChrome APIのエラーメッセージに依存する。Chromeバージョンアップで文字列が変わる可能性がある。
- `consecutiveFailures`はstateに永続化されるため、ループの各イテレーション間で正確に追跡できる。
