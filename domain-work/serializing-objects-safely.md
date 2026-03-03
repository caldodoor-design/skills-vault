---
name: serializing-objects-safely
description: 循環参照・DOMノード・巨大データに対処する安全なJSONシリアライザ
---
# 安全なオブジェクトJSON化

```typescript
const MAX_DATA_SIZE = 5000, MAX_DEPTH = 5;

export function safeSerialize(value: unknown, depth = 0, seen = new WeakSet<object>()): unknown {
  if (value === null || value === undefined) return value;
  if (typeof value === "string") return value.length > MAX_DATA_SIZE ? value.slice(0, MAX_DATA_SIZE) + `...[truncated]` : value;
  if (typeof value === "boolean" || typeof value === "number") return value;
  if (depth > MAX_DEPTH) return "[max depth exceeded]";
  if (value instanceof Error) return { name: value.name, message: value.message, stack: value.stack?.slice(0, 1000) };
  if (typeof Node !== "undefined" && value instanceof Node) return `[DOM: ${value.nodeName}]`;
  if (typeof value === "function") return `[Function: ${value.name || "anonymous"}]`;
  if (typeof value === "object") {
    if (seen.has(value)) return "[circular reference]";
    seen.add(value);
    if (Array.isArray(value)) {
      const arr = value.slice(0, 50).map(i => safeSerialize(i, depth + 1, seen));
      if (value.length > 50) arr.push(`...[${value.length - 50} more]`);
      return arr;
    }
    const result: Record<string, unknown> = {};
    let size = 0;
    for (const key of Object.keys(value).slice(0, 50)) {
      const s = safeSerialize((value as any)[key], depth + 1, seen);
      size += (JSON.stringify(s) || "").length;
      if (size > MAX_DATA_SIZE) { result["__truncated__"] = true; break; }
      result[key] = s;
    }
    return result;
  }
  return String(value);
}
```

## 防御レイヤー
| 対象 | 制限 |
|------|------|
| 文字列 | 5000文字（Base64混入防止） |
| ネスト | 5階層 |
| 循環参照 | WeakSet追跡（Setは使わない→GCに影響） |
| キー/配列 | 各50個まで |
| DOMノード | nodeNameのみ（`typeof Node !== "undefined"`でSW対応） |
| Error | name/message/stack(1000字) |
