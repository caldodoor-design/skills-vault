---
name: polling-indeterminate-progress
description: 総数不明→確定の進捗バー動的切り替えと画像二重取得防止パターン
---
# 不確定→確定進捗UI

## 状態遷移
```
[0% 静止] → [30% pulse] → [N% 静止] → [100% 静止]
 待機中      不確定(total不明)  確定(total判明)   完了
```

## ポーリング（1秒間隔）
- 開始時に必ず`stopPolling()`→`setInterval(pollStatus, 1000)`
- エラーは無視（次回で回復）
- 停止条件: `status === "complete" | "error"` or `!isCapturing && idle && hasCaptureStarted`

## 進捗バー切り替え
```typescript
if (total && total > 0) {
  progressBar.style.width = `${Math.min((count / total) * 100, 100)}%`;
  progressBar.style.animation = "none";
} else if (isCapturing) {
  progressBar.style.width = "30%";
  progressBar.style.animation = "pulse 1.5s infinite";
}
```
`animation = "none"` は明示的にセット（自動では消えない）。

## 二重取得防止（3フラグ）
```typescript
const shouldFetch = (status === "complete" || status === "error")
  && !imagesFetched      // まだ取得していない
  && !isFetchingImages   // 取得中でもない
  && capturedImages.length === 0; // ローカルにもない
```
関数レベルでもガード（二重防御）。
