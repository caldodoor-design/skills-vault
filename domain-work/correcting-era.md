---
name: correcting-era
description: 短い年号を令和優先で西暦変換するSRPルール
trigger: 年号判定ロジック（fixEraRollover等）の実装・修正時
---

# S30: Era Correction — Strong Reiwa Priority (SRP)

短い年号（"7","07"等）→ 物理的に可能なら無条件で令和。

```typescript
const currentReiwaYear = currentYear - 2018; // 2026 -> R8
if (yearPart > 0 && yearPart <= currentReiwaYear + 1) {
    return 2018 + yearPart; // 強制的に令和
}
```

## Fallback
SRP範囲外（例: `30`）は従来ロジック:
1. 未来チェック: 現在+1年より未来にはならない
2. 距離チェック: 直前の行（lastWesternYear）に近い年代を採用

## 注意
- SRP基準を緩めない（弱めない）こと
