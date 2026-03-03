---
name: navigating-pages-with-session-id
description: UUIDセッションIDで非同期ループの陳腐化検出、Ping-Pong状態遷移と二段タイムアウトで安全なページ遷移管理
---
# Session IDパターンによるページ遷移管理

## 問題
停止→再開始で古いループと新しいループが並行動作するrace condition。

## Session ID検証
```typescript
const sessionId = `${Date.now()}-${Math.random().toString(36).substring(2, 9)}`;
await setState({ isCapturing: true, sessionId });

async function captureLoop(sessionId: string): Promise<void> {
  // ★ 全てのawaitの後にsessionIdを再検証
  if (getState().sessionId !== sessionId) return; // staleループは静かに終了
  // ... 重い非同期処理 ...
  if (!getState().isCapturing || getState().sessionId !== sessionId) return;
  // ... delay後にも再検証 ...
}
```

## Ping-Pong状態遷移
```
SW → TURN_PAGE → CS（キー入力→スピナー検出→消失検出）→ PAGE_TURNED → SW
```
- `awaitingPageTurn`フラグで重複PAGE_TURNEDを除外
- `expectedPage`で順序整合性を検証

## 二段タイムアウト
| 段階 | タイムアウト | 意味 |
|------|------------|------|
| TURN_PAGE応答 | 30秒 | CS がメッセージに無応答 |
| PAGE_TURNED待機 | 60秒 | ページ描画が完了しない |

タイムアウトcallback内でもsessionId + isCapturingを検証。

## ルール
- 全awaitポイント後にsessionId再検証（1つでも欠けると余分な1ステップ実行）
- staleループではfinishCapture()を呼ばない（新セッションを巻き込む）
- clearTimeout忘れ禁止（正常完了後にタイムアウト発火する）
- MAX_CONSECUTIVE_FAILURES=5で連続失敗時に強制終了
