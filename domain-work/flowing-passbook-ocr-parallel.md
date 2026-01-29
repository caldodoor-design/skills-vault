---
name: flowing-passbook-ocr-parallel
description: 通帳OCRの並列処理においてZustandストアへの競合を防ぐパターン。Promise.all内で直接ストア更新せず、結果収集後に順次適用する。再解析時のprocessedAt維持ルールも含む。
trigger: OCRページ解析の並列処理、Promise.allとストア更新の組み合わせ、または再解析時のタイムスタンプ管理を実装・修正するとき。
---

# Skill 29: Passbook OCR Parallel Flow

> Created by **Agent Delta** based on `reports/sprint-ocr-optimize/verification_report_v8_9.md`

## Context
OCRによるページ解析は時間がかかるため並列処理 (Parallel Processing) が必須。
しかし、Zustandストアなどの状態管理において、非同期処理（`Promise.all`）内部での直接更新は競合（Race Condition）を引き起こすことが検証により確認された。

## The Solution (Verified)
Gammaの検証レポートに基づき、以下のパターンを標準とする。

### 1. Result Collection Pattern
処理ロジック内でストアを直接更新せず、処理結果オブジェクトを返す。

```typescript
// Good: Return Result
const processPage = async (i: number): Promise<PageResult> => {
    // ... logic ...
    return { success: true, data: ... };
};
```

```typescript
// Bad: Direct Mutation
const processPage = async (i: number) => {
    // ... logic ...
    store.update(...); // Race Condition!
};
```

### 2. Sequential Application
`Promise.all` で全ての非同期処理が完了した後、メインスレッドで**順次ループ**を用いてストアに適用する。

```typescript
const results = await Promise.all(batch);
for (const result of results) {
    if (result.success) {
        store.update(result.data); // Safe
    }
}
```

### 3. Order Guarantee
`Date.now()` は並列処理下で順序を保証しない。
必ず `startTime + index` （バッチ開始時刻＋配列インデックス）を用いて、アップロード順序を厳密に維持する。

## Anti-Patterns
- `Promise.all` の中での `set` / `dispatch` 呼び出し。
- 順序キーとしての `Date.now()` の単独使用。

## Re-analysis Strategy (Added v8.9)
**再解析（単一ページ更新）時の注意点**:
- ページ再解析時は、**既存の `processedAt` を維持する**こと。
- `processedAt` を `Date.now()` で更新すると、ソート順が変わってしまい（末尾に移動）、UXを損なう。
- **Rule**: `updatePage(id, { ...oldMeta, ...newData })` の形を守り、タイムスタンプの意図しない上書きを防ぐ。
