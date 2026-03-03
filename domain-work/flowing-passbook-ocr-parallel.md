---
name: flowing-passbook-ocr-parallel
description: 通帳OCR並列処理でのZustand競合防止パターン
trigger: OCR並列処理とストア更新の組み合わせ実装時
---

# S29: Passbook OCR Parallel Flow

## パターン: Result Collection → Sequential Application

```typescript
// Good: 結果を返すだけ
const processPage = async (i: number): Promise<PageResult> => {
    return { success: true, data: ... };
};

// Promise.all後に順次適用
const results = await Promise.all(batch);
for (const result of results) {
    if (result.success) store.update(result.data); // Safe
}
```

## ルール
1. `Promise.all`内でストア直接更新禁止（Race Condition）
2. 順序キーは`startTime + index`（`Date.now()`単独禁止）
3. 再解析時は既存の`processedAt`を維持（ソート順変更防止）
