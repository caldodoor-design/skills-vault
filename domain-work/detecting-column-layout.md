---
name: detecting-column-layout
description: 通帳ページの列境界推定→columnMap生成。ヘッダーアンカー優先、フォールバックでVirtual Grid。
---
# ColumnLayoutDetector

## I/O
- **入力**: page(width, height, tokens[{text, bbox, confidence}]), options(preferHeaderAnchors, fallbackVirtualGrid)
- **出力**: columnMap(date/withdraw/deposit/description/balance各XRange, 0〜1正規化), confidence(0-1), status(OK|WARNING)

## ルール
1. ヘッダー/アンカー優先で列推定（安定）
2. アンカー無し→Virtual Gridで仮置き+WARNING
3. columnMapは必ず0〜1正規化
4. 境界曖昧→広めに取りWARNING
5. 同じ入力→同じcolumnMap（deterministic）

## 既知の制限
- Auto Detectionは日本標準通帳のハードコードVirtual Gridに依存
- 特殊レイアウト→S24手動補正（Manual Profile）が必須
