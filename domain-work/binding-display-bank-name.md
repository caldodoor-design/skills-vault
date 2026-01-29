---
name: binding-display-bank-name
description: ユーザーが入力した金融機関名（displayBankName）をUI表示に確実に反映する。OCR誤読で空欄になってもユーザー入力を優先し、bankIdとdisplayNameの概念を分離する。
trigger: 銀行名の表示ロジック、ユーザー入力とOCR検出の優先度判定、またはbankIdとdisplayName の分離に関わる実装・修正時。
---

# S28: DisplayBankNameBinder

## 1. Purpose
ユーザーが最初に入力した金融機関名（displayBankName）を、画面上の銀行名表示へ確実に反映する。
OCR誤読で空欄/不明になっても、表示はユーザー入力を優先する。

## 2. Scope
- 対象：UI表示（左上の通帳表紙など）
- 対象：全ページ/案件
- 非対象：学習ルール検索キー（bankId）は別概念

## 3. Inputs / Outputs
### Inputs
- bankId
- userEnteredBankName (string)
- ocrDetectedBankName? (string)

### Outputs
- displayBankName (string)
- source: "USER" | "OCR" | "EMPTY"

## 4. Rules
1) userEnteredBankName があれば常に最優先（source=USER）
2) userEnteredが無い場合のみOCRを使う（source=OCR）
3) どちらも無ければ "不明"（source=EMPTY）
4) OCR値で userEntered を上書きしない

## 5. DoD
- 初期入力の銀行名が必ずUIに出る
- OCRが空でも表示が空欄にならない
- bankIdとdisplayNameの概念が分離される

## 6. Test Cases
- TC1: userEnteredあり → USER採用
- TC2: userEntered無し + OCRあり → OCR採用
- TC3: 両方無し → "不明"

## 7. Anti-Patterns
- OCRが誤読して表示が変わる
- bankIdとdisplayNameが混ざって混線する

## 8. Notes
- 製品体験として「入力した意味がある」状態を作る。
