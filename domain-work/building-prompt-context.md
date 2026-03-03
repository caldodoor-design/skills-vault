---
name: building-prompt-context
description: 学習データをFew-shotプロンプトとしてOCR(Gemini)に提供
trigger: OCRプロンプトへの学習データ注入・Few-shot生成ロジック実装時
---

# S3: PromptContextBuilder

学習データをFew-shotプロンプトとしてOCR（Gemini）に提供し、認識精度向上とフォーマット統一を図る。

## ルール
1. Client(通帳記載)が入力、Description(摘要ラベル)が出力: `When you see "{client}", map it to description="{description}"`
2. normalized: {clientNorm} も併記し揺らぎ許容を教える
3. Context Limit: 直近20〜50件に絞る（トークン節約）
4. 指定bankIdのルールのみ展開

## I/O
- Input: bankId
- Output: promptText (LLMへの指示プロンプト)

## 注意
- Description→Clientの逆方向で教えると混乱する
- 保存は無制限、プロンプト生成時は`slice(-20)`等で制限
