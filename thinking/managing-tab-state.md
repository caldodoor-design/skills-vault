---
name: managing-tab-state
description: Chrome拡張でのタブID取得・有効性検証・特殊ページ判定パターン
skills: [handling-chrome-extension-messaging]
---
# Tab State Management

## タブID取得（Popup開始時）
```typescript
async function getTargetTabId(sender: chrome.runtime.MessageSender): Promise<number | null> {
  if (sender.tab?.id) return sender.tab.id;
  const tabs = await chrome.tabs.query({ active: true, currentWindow: true });
  return tabs[0]?.id ?? null;
}
```

## タブ有効性検証
```typescript
async function isTabValid(tabId: number): Promise<boolean> {
  try { const tab = await chrome.tabs.get(tabId); return tab !== undefined; }
  catch { return false; }
}
```
長時間ループでは各ステップでisTabValid確認。

## 状態ストレージ
```typescript
interface TaskState { isRunning: boolean; tabId: number | null; }
const STORAGE_KEY = "taskState";
async function getState(): Promise<TaskState> {
  const result = await chrome.storage.local.get(STORAGE_KEY);
  return result[STORAGE_KEY] ?? { isRunning: false, tabId: null };
}
```

## 注意点
- `currentWindow: true` はPopupが別ウィンドウだと意図しないタブを取得する可能性
- **特殊ページ**（`chrome://*`, `chrome-extension://*`, `devtools://*`）ではContent Script不動作
- `chrome.tabs.TAB_ID_NONE` (-1) を有効なIDとして扱わない

```typescript
function isRestrictedUrl(url: string | undefined): boolean {
  if (!url) return true;
  return /^(chrome|chrome-extension|devtools|edge|about):/.test(url);
}
```
