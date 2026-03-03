---
name: simulating-keyboard-via-debugger
description: Chrome Debugger API Input.dispatchKeyEventによるRTL/LTR対応キーボードシミュレーション
---
# Debugger APIキーボードシミュレーション

## コア実装
```typescript
export async function navigateToNextPageViaDebugger(tabId: number, direction: "ltr" | "rtl"): Promise<void> {
  const keyCode = direction === "rtl" ? "ArrowLeft" : "ArrowRight";
  const vkCode = direction === "rtl" ? 37 : 39;
  const params = { key: keyCode, code: keyCode, windowsVirtualKeyCode: vkCode, nativeVirtualKeyCode: vkCode };

  await sendDebuggerCommand(tabId, "Input.dispatchKeyEvent", { type: "keyDown", ...params });
  await delay(50); // 人間のキー押下を模倣
  await sendDebuggerCommand(tabId, "Input.dispatchKeyEvent", { type: "keyUp", ...params });
}
```

## 必須要件
- **keyDown + delay(50) + keyUp** の完全シーケンス（keyDownだけだとリピート扱い）
- **windowsVirtualKeyCode必須**: 未指定だと`event.keyCode`が0になりKindleが反応しない
- manifest.jsonに`"debugger"`権限が必要

## Debugger vs Content Script
| 方法 | 用途 |
|------|------|
| Debugger API | iframe境界を越える確実なキー入力（デバッグバー表示あり） |
| Content Script | DOM操作、状態監視（警告なし、iframe内に届かない場合あり） |

ハイブリッド構成推奨: Debuggerでキー入力→CSで遷移完了検出（Ping-Pong）
