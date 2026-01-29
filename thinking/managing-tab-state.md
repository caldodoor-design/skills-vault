---
name: managing-tab-state
description: Chrome拡張でのタブID取得・有効性検証・特殊ページ判定・ストレージ管理のパターンを定義する。タブ操作を伴う拡張機能の状態管理実装時に使う。
skills: [handling-chrome-extension-messaging]
---

# Tab State Management for Chrome Extensions

Chrome拡張機能でターゲットタブのID管理・有効性検証を行い、タブが閉じられた際のエラー処理を確実に行うためのガイドライン。

## 課題

- ユーザーが対象タブを閉じたり、別のウィンドウへ移動した場合、`chrome.tabs.sendMessage` が失敗する。
- Popup経由で操作を開始した場合、`sender.tab` が存在しないため、ターゲットタブIDを別途取得する必要がある。
- `chrome.tabs.get()` はタブが存在しない場合に例外を投げる（Promiseがreject）。

## ターゲットタブIDの取得

### Popupからの開始時
Popupには `sender.tab` がないため、アクティブタブを明示的に取得する。

```typescript
async function getTargetTabId(
  sender: chrome.runtime.MessageSender
): Promise<number | null> {
  // Content Scriptからのメッセージなら sender.tab.id を使用
  if (sender.tab?.id) {
    return sender.tab.id;
  }

  // Popupからの場合はアクティブタブを取得
  const tabs = await chrome.tabs.query({ active: true, currentWindow: true });
  if (tabs.length > 0 && tabs[0].id) {
    return tabs[0].id;
  }

  console.warn("[SW] Could not determine target tab ID");
  return null;
}
```

## タブ有効性の検証

保存したタブIDが依然として有効かどうかを確認する関数。

```typescript
async function isTabValid(tabId: number): Promise<boolean> {
  try {
    const tab = await chrome.tabs.get(tabId);
    return tab !== undefined;
  } catch {
    // タブが存在しない場合、chrome.tabs.get は例外を投げる
    return false;
  }
}
```

## 使用例: ループ処理での活用

長時間実行するタスク（キャプチャループ等）では、各ステップでタブの有効性を確認する。

```typescript
async function processLoop(tabId: number): Promise<void> {
  while (state.isRunning) {
    // タブがまだ存在するか確認
    if (!(await isTabValid(tabId))) {
      console.error("[SW] Target tab was closed");
      await finishTask("error", "Tab was closed");
      return;
    }

    // ... 処理 ...
  }
}
```

## 状態ストレージでの管理

`chrome.storage.local` に `tabId` を保存し、Popup再オープン時やService Worker復帰時に参照できるようにする。

```typescript
interface TaskState {
  isRunning: boolean;
  tabId: number | null;
  // ... 他のプロパティ
}

const STORAGE_KEY = "taskState";

async function getState(): Promise<TaskState> {
  const result = await chrome.storage.local.get(STORAGE_KEY);
  return result[STORAGE_KEY] ?? { isRunning: false, tabId: null };
}

async function setState(partial: Partial<TaskState>): Promise<void> {
  const current = await getState();
  await chrome.storage.local.set({ [STORAGE_KEY]: { ...current, ...partial } });
}
```

## 注意点

- **複数ウィンドウ**: `currentWindow: true` は呼び出し元のウィンドウを基準にするため、Popupが別ウィンドウにある場合に意図しないタブを取得する可能性がある。
- **Incognito**: シークレットウィンドウのタブは通常ウィンドウからアクセスできない場合がある。`manifest.json` の `incognito` 設定を確認すること。
- **特殊ページ (Critical)**: 以下のURLパターンでは Content Script が動作しないため、`sendMessage` は失敗する：
    - `chrome://*` (設定、拡張機能管理など)
    - `chrome-extension://*` (他の拡張機能のページ)
    - `devtools://*` (開発者ツール)
    - `edge://*`, `about:*` (ブラウザ固有ページ)
    これらのタブが対象の場合は、事前に `tab.url` をチェックして除外すること。
- **TAB_ID_NONE**: `chrome.tabs.TAB_ID_NONE` (値: `-1`) は、タブが存在しないコンテキスト（ポップアップ、拡張ページ、devtools内など）から送信された場合に返される。この値を有効なタブIDとして扱わないこと。

```typescript
// 特殊ページかどうかを判定
function isRestrictedUrl(url: string | undefined): boolean {
  if (!url) return true;
  return /^(chrome|chrome-extension|devtools|edge|about):/.test(url);
}
```

