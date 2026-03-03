---
name: binding-display-bank-name
description: ユーザー入力の金融機関名をUI表示に確実に反映
trigger: 銀行名表示ロジック、bankIdとdisplayNameの分離実装時
---

# S28: DisplayBankNameBinder

ユーザー入力の金融機関名（displayBankName）をUI表示に確実に反映。OCR誤読でも表示はユーザー入力を優先。

## ルール
1. userEnteredBankName があれば常に最優先（source=USER）
2. userEntered無しの場合のみOCR使用（source=OCR）
3. どちらも無ければ "不明"（source=EMPTY）
4. OCR値でuserEnteredを上書き禁止

## I/O
- Input: bankId, userEnteredBankName, ocrDetectedBankName?
- Output: displayBankName, source("USER"|"OCR"|"EMPTY")
