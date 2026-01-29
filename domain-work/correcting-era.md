---
name: correcting-era
description: 通帳OCRにおける短い年号（例：「7」「07」）を令和優先で正しく西暦に変換するヒューリスティック。Strong Reiwa Priority（SRP）ルールにより、物理的に可能な令和年は無条件で令和とみなす。
trigger: 年号判定ロジック（fixEraRollover等）の実装・修正時、またはOCRデータの年号クレンジング処理に関わるとき。
---

# Skill 30: Era Correction Strategy

> Created by **Agent Delta** based on `reports/sprint-ocr-optimize/verification_report_v8_9.md`

## Context
通帳OCRにおいて、「7」や「07」のような短い年号（Short Year）が出現する。
これを単純な「現在年との距離」だけで判定すると、平成7年(1995)と誤認されるリスクがある。
Gammaの検証により、以下のヒューリスティックが有効であることが確認された。

## The Solution (Verified)

### Strong Reiwa Priority (SRP)
現在進行系の記帳データにおいて、平成一桁（30年以上前）のデータが処理される確率は極めて低い。
したがって、「物理的に可能な令和の年」であれば、無条件で令和とみなす。

**Logic:**
```typescript
const currentReiwaYear = currentYear - 2018;
// e.g. 2026 -> R8

if (yearPart > 0 && yearPart <= currentReiwaYear + 1) {
    // 範囲内なら強制的に令和 (2018 + yearPart)
    return 2018 + yearPart;
}
```

### Fallback Logic
SRPの範囲外（例: `30`）の場合は、従来通り以下の優先度で判定する。
1. **未来チェック**: 現在+1年より未来にはならない。
2. **距離チェック**: 直前の行（`lastWesternYear`）に近い年代を採用。

## Usage Guidelines
- OCR処理後のデータクレンジング工程（`fixEraRollover`等）では必ずこのロジックを使用すること。
- 年号判定ロジックを変更する際は、必ず本スキルを参照し、SRP基準を緩めない（弱めない）こと。
