---
name: applying-column-template
description: BankProfile比率でページ垂直分割しトークンを列に割り当て
trigger: 列テンプレート適用・列境界判定の実装・修正時
---

# S8: ColumnTemplateApplier

BankProfile（S7）の比率に基づきページを垂直分割し、OCRトークンを列に割り当て。

## ルール
1. **Strict Banding**: 列範囲（X座標）に中心点が含まれるトークンをその列に割り当て
2. **Straddle Warning**: 列境界跨ぎ（Overlap < 70%）は必ずWARNING
3. **Boundary Confidence**: 列境界付近でConfidence < 0.2もWARNING
4. はみ出しトークンも警告

## I/O
- Input: Token[], ColumnRatios
- Output: TransactionRow[], ValidationWarning[]

## 注意
- 境界線上の文字を勝手に割り振ってWARNINGを出さないのはアンチパターン（金額ミスの原因）
