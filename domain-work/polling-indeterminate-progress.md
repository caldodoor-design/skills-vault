---
name: polling-indeterminate-progress
description: 総数不明の進捗表示を不確定アニメーションから確定進捗バーに動的切り替えし、画像取得の二重実行をフラグで防止するUIパターン。
trigger: 進捗の総数が後から判明するUI、またはポーリングベースの状態監視を実装するとき。
---

# 不確定→確定進捗UIの切り替え

## Purpose

キャプチャ処理の総ページ数（maxPages）はユーザーが指定しない場合は不明。総数不明でも「処理中」を伝え、総数が判明した時点で正確な進捗率に切り替える。また、完了時の画像取得が二重に実行されないよう防御する。

## Context

Chrome拡張のPopup UIから、Background Service Workerのキャプチャ状態を1秒間隔でポーリングする。状態には以下の情報が含まれる:
- `isCapturing`: キャプチャ中か
- `capturedCount`: キャプチャ済みページ数
- `maxPages`: 総ページ数（undefined = 不明）
- `status`: "idle" | "capturing" | "complete" | "error"

## Pattern

### 1. ポーリングの開始と停止

```typescript
let pollIntervalId: ReturnType<typeof setInterval> | null = null;

function startPolling(): void {
  stopPolling(); // 既存のインターバルをクリア
  pollIntervalId = setInterval(pollStatus, 1000);
}

function stopPolling(): void {
  if (pollIntervalId !== null) {
    clearInterval(pollIntervalId);
    pollIntervalId = null;
  }
}

async function pollStatus(): Promise<void> {
  try {
    const response = await sendMessage<void, CaptureState>({ type: "GET_CAPTURE_STATE" });
    if (response.success && response.data) {
      updateUI(response.data);
    }
  } catch {
    // ポーリングエラーは無視（次回で回復する）
  }
}
```

**なぜ1秒間隔か**: ページキャプチャは1ページ1-3秒かかる。1秒ポーリングならUIが適度に更新され、かつBackgroundへの負荷が小さい。

### 2. 不確定→確定の進捗バー切り替え

```typescript
if (progressBar) {
  if (total && total > 0) {
    // 確定進捗: 正確なパーセンテージ
    const progress = Math.min((capturedCount / total) * 100, 100);
    progressBar.style.width = `${progress}%`;
    progressBar.style.animation = "none";
  } else if (isCapturing) {
    // 不確定進捗: pulseアニメーション
    progressBar.style.width = "30%";
    progressBar.style.animation = "pulse 1.5s infinite";
  } else if (capturedImages.length > 0) {
    // 完了: 100%
    progressBar.style.width = "100%";
    progressBar.style.animation = "none";
  } else {
    // 待機中: 0%
    progressBar.style.width = "0%";
    progressBar.style.animation = "none";
  }
}
```

**状態遷移図:**
```
[0% / 静止]  →  [30% / pulse]  →  [N% / 静止]  →  [100% / 静止]
  待機中        不確定進捗        確定進捗          完了
               (total不明)      (total判明)
```

### 3. ステータステキストの連動

```typescript
if (statusText) {
  if (status === "error") {
    statusText.textContent = "Error occurred";
  } else if (isCapturing) {
    statusText.textContent = total
      ? `Capturing page ${capturedCount} of ${total}...`
      : "Capturing pages...";  // 総数不明時は数値を出さない
  } else if (status === "complete" || capturedImages.length > 0) {
    statusText.textContent = "Capture complete!";
  } else {
    statusText.textContent = "Ready";
  }
}
```

### 4. Double-Fetch Prevention（二重取得防止）

```typescript
let isFetchingImages = false;
let imagesFetched = false;

// ポーリング停止条件
if (status === "complete" || status === "error" || (!isCapturing && status === "idle" && hasCaptureStarted)) {
  stopPolling();

  // 画像取得（3つのフラグで二重実行を防止）
  const shouldFetch = (status === "complete" || status === "error") &&
    !imagesFetched &&      // まだ取得していない
    !isFetchingImages &&   // 取得中でもない
    capturedImages.length === 0; // ローカルにもない

  if (shouldFetch) {
    fetchCapturedImages();
  }
}

async function fetchCapturedImages(): Promise<void> {
  if (isFetchingImages || imagesFetched) {
    return; // ガード（関数レベルでも二重防止）
  }
  isFetchingImages = true;

  try {
    const response = await sendMessage({ type: "GET_CAPTURED_PAGES" });
    if (response.success && response.data?.pages) {
      capturedImages = response.data.pages.map((p) => p.dataUrl);
      imagesFetched = true;
    }
  } finally {
    isFetchingImages = false;
  }
}
```

**なぜ3つのフラグが必要か:**
- `imagesFetched`: 取得完了を記録。完了後の再取得を防ぐ。
- `isFetchingImages`: 取得中を記録。ポーリングの最後のイベントが2回`updateUI`を呼ぶレースコンディションを防ぐ。
- `capturedImages.length === 0`: stateにインラインで画像が含まれている場合は取得不要。

## Rules

- ポーリング開始時に必ず`stopPolling()`を先に呼ぶ。二重インターバルを防止する。
- ポーリングエラーは無視する。次回のポーリングで自動回復するため。
- `animation = "none"`を明示的にセットする。CSSの`animation`プロパティは一度セットすると自動では消えない。
- 画像取得は呼び出し箇所と関数内部の両方でガードする（二重防御）。

## Anti-Patterns

- WebSocketやlong pollingで状態を監視する（Chrome拡張のService Worker→Popup通信には過剰。Service Workerがいつ停止するか不定）
- `setInterval`の戻り値をチェックせずに`clearInterval(null)`する（動作はするがコードの意図が不明確）
- 不確定進捗でwidthを0%やランダム値にする（ユーザーに「壊れている」と思わせる）
- 画像取得のフラグを1つだけにする（レースコンディションで二重取得が発生する）
- `hasCaptureStarted`なしにidle状態でポーリングを止める（初期表示でもidleなので、開始前にポーリングが止まってしまう）

## Notes

- `pulse`アニメーションはCSS側で定義。`@keyframes pulse { 0%, 100% { opacity: 0.6 } 50% { opacity: 1 } }`のような不確定プログレスバーの標準的なパターン。
- `Math.min(..., 100)`で100%を超えないようガード。`capturedCount > total`になるエッジケース（maxPages推定ミス等）に対応。
- ポーリング間隔1000msはUI更新頻度とバッテリー消費のバランス。Popup表示中のみ動作するため、バッテリー影響は軽微。
