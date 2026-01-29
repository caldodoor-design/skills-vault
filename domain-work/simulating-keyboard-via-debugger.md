---
name: simulating-keyboard-via-debugger
description: Chrome Debugger APIのInput.dispatchKeyEventを使い、RTL/LTR対応のキーボード入力シミュレーションを行うパターン。Content Scriptからのdispatchとの使い分けを含む。
trigger: Chrome拡張からキーボードイベントをシミュレーションする必要があるとき、特にiframeやセキュリティ制約のあるページでの操作
---

# Debugger API経由のキーボード入力シミュレーション

## Purpose

Kindle Cloud Readerのような複雑なWebアプリでは、Content ScriptからのKeyboardEvent dispatchがiframe境界やイベントリスナーの制約で動作しないことがある。Chrome Debugger APIの`Input.dispatchKeyEvent`を使うことで、ブラウザレベルで確実にキー入力をシミュレーションする。

## Context

- Kindle Cloud ReaderはiFrame内にレンダリングされるため、Content Scriptからのイベントが届かない場合がある
- `document.dispatchEvent(new KeyboardEvent(...))` はフレーム境界を越えられない
- DebuggerAPIはブラウザレベルで動作するため、iframe内のリスナーにも到達する
- RTL（マンガ、右綴じ）とLTR（洋書、左綴じ）で次ページの方向キーが異なる

## Pattern

### Direction-awareキー選択

```typescript
export async function navigateToNextPageViaDebugger(
  tabId: number,
  direction: "ltr" | "rtl"
): Promise<void> {
  // RTL (マンガ): 次ページは左矢印 (keyCode 37)
  // LTR (洋書): 次ページは右矢印 (keyCode 39)
  const keyCode = direction === "rtl" ? "ArrowLeft" : "ArrowRight";
  const windowsVirtualKeyCode = direction === "rtl" ? 37 : 39;

  // keyDown イベント
  await sendDebuggerCommand(tabId, "Input.dispatchKeyEvent", {
    type: "keyDown",
    key: keyCode,
    code: keyCode,
    windowsVirtualKeyCode,
    nativeVirtualKeyCode: windowsVirtualKeyCode,
  });

  // keyDownとkeyUpの間にディレイ（実際のキー押下を模倣）
  await delay(50);

  // keyUp イベント
  await sendDebuggerCommand(tabId, "Input.dispatchKeyEvent", {
    type: "keyUp",
    key: keyCode,
    code: keyCode,
    windowsVirtualKeyCode,
    nativeVirtualKeyCode: windowsVirtualKeyCode,
  });
}
```

### なぜkeyDown + delay + keyUpの完全シーケンスが必要か

1. **keyDownだけ**: 一部のアプリはkeyDown→keyUpの完全なシーケンスを要求する（特にKindle Cloud Reader）
2. **delay(50)**: 人間のキー押下を模倣。delayなしだとkeyDown/keyUpが同一フレームで処理され、「キーが押された」と認識されない場合がある
3. **keyUpの省略**: keyDownイベントが残り続け、次回のkeyDownが「キーリピート」として扱われる可能性がある

### windowsVirtualKeyCodeの重要性

```typescript
// ★ 必須パラメータ
windowsVirtualKeyCode: direction === "rtl" ? 37 : 39,
nativeVirtualKeyCode: windowsVirtualKeyCode,
```

- Chrome DevTools ProtocolのInput.dispatchKeyEventでは、`key`や`code`だけでなく`windowsVirtualKeyCode`の指定が重要
- 一部のWebアプリは`event.keyCode`（レガシー）で判定しており、`windowsVirtualKeyCode`が設定されていないとkeyCodeが0になる
- Kindle Cloud Readerはまさにこのパターンで、`windowsVirtualKeyCode`なしだとページ遷移しない

### Content Script経由との使い分け

| 方法 | 長所 | 短所 | 用途 |
|------|------|------|------|
| **Debugger API** | iframe境界を越える、確実 | ユーザーに「デバッグ中」の警告バーが表示される | ページ遷移の実際のキー入力 |
| **Content Script** | 警告なし、権限不要 | iframe内に到達しない場合がある | DOM操作、状態監視 |

実装ではDebugger APIでキー入力→Content Scriptで遷移完了を検出（Ping-Pong）というハイブリッド構成が有効。

## Rules

- `keyDown`と`keyUp`は必ずペアで送ること
- keyDownとkeyUpの間に最低50msのdelayを入れること
- `windowsVirtualKeyCode`と`nativeVirtualKeyCode`の両方を必ず指定すること
- RTL/LTRの方向は呼び出し側で正しく判定してから渡すこと

## Anti-Patterns

- **keyDownだけ送ってkeyUpを省略**: キーリピート扱いされる、またはイベントが認識されない
- **windowsVirtualKeyCodeを省略**: event.keyCodeが0になり、レガシーなイベントハンドラが反応しない
- **delay(0)やdelayなし**: 同一フレーム内での処理により認識されない場合がある
- **Content Scriptだけでキー入力をシミュレーション**: iframe内のKindleリーダーに到達しない

## Notes

- Debugger APIの使用中はタブ上部に「<拡張名>がこのブラウザのデバッグを開始しました」の青いバーが表示される。これはChromeのセキュリティ機能であり非表示にできない
- `Input.dispatchKeyEvent`の`type`は `"keyDown"`, `"keyUp"`, `"rawKeyDown"`, `"char"` の4種類。テキスト入力には`"char"`も必要だが、矢印キーには不要
- Debugger APIを使用するにはmanifest.jsonで`"debugger"`権限が必要
