---
name: detecting-column-layout
description: 通帳ページごとに列の境界（支払/預入/摘要/残高など）を推定し、OCR結果を正しい列に割り当てるためのcolumnMapを生成する。銀行差や撮影歪みがあっても安定した列検出を行う。
trigger: OCR結果のトークンを列（セル）に割り当てる前段階で、ページの列レイアウトを検出・推定する必要があるとき。
---

# S23: ColumnLayoutDetector

> Updated by **Agent Delta** based on `reports/sprint-008/verification_report.md`
> Validated: 2026-01-20 (Sprint 8 Verification)

## 1. Purpose
通帳ページごとに「列の境界（支払/預入/摘要/残高など）」を推定し、
OCR結果を列に正しく割り当てられるようにする。
銀行差や撮影歪みにより列がズレても、正規化されたcolumnMapを返す。

## 2. Scope
- 対象：1ページ（画像 or OCR bbox情報）
- 出力：columnMap（0〜1正規化のX範囲）+ 推定信頼度
- 非対象：金額の方向判定、残高整合（S5/S6が担当）

## 3. Inputs / Outputs
### Inputs
- page:
  - width, height
  - tokens: {text, bbox:{x0,y0,x1,y1}, confidence}[]
  - optional: headerCandidates（あるなら）
- options:
  - preferHeaderAnchors: boolean
  - fallbackVirtualGrid: boolean

### Outputs
- result:
  - status: "OK" | "WARNING"
  - columnMap:
    - dateXRange
    - withdrawXRange
    - depositXRange
    - descriptionXRange
    - balanceXRange
    - miscXRange (optional)
  - confidence: number (0-1)
  - reasons: string[]

## 4. Rules
1) 可能ならヘッダー/アンカー優先で列推定（安定）
2) アンカーが無い/弱い場合は Virtual Grid で仮置きし WARNING
3) columnMapは必ず0〜1に正規化（描画/再利用のため）
4) 2列の境界が曖昧な場合は広めに取り、CHECK側でWARNINGを付ける

## 5. DoD
- columnMapが返る（必ず）
- ヘッダー無しでも fallbackVirtualGrid で動く（ただしWARNING）
- 同じ入力なら同じcolumnMap（deterministic）

## 6. Test Cases
- TC1: ヘッダーあり → OKでcolumnMap
- TC2: ヘッダーなし → WARNINGでVirtualGrid
- TC3: bboxが歪んでも列レンジが崩れない（許容）
- TC4: 同一入力で同一結果（決定論）

## 7. Anti-Patterns
- 返さない（null）で後段が詰む
- 推定がランダムで毎回変わる
- confidence無しでブラックボックス化

## 8. Notes
- 目的は「列の安定」。賢さより安全に倒す。
- **Known Issue (Sprint 8 Verified)**:
  - "Auto Detection" は現在、一般的な通帳に合わせたハードコードされた "Japanese Standard Virtual Grid" に依存している。
  - 特殊なレイアウト（海外通帳や特殊帳票）の場合、精度が落ちるため、その場合は `S24` による手動補正（Manual Profile）が必須となる。
