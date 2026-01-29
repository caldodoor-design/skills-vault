---
name: building-prompt-context
description: 学習データをFew-shotプロンプトとしてOCR（Gemini）に提供し、認識精度とフォーマット統一を図るSkill。OCR API呼び出し時のプロンプト生成で使用する。
trigger: OCRプロンプトへの学習データ注入や、Few-shot生成ロジックを実装・修正するとき。PromptContextBuilderやgetPromptContextに関わるコード変更時。
---

# Skill: PromptContextBuilder (S3) v1.1

## 1. Purpose
OCR (Gemini) に対して、ユーザーが過去に修正・登録した学習データを Few-shot プロンプトとして提供し、認識精度の向上とフォーマットの統一を図る。

## 2. Scope
- `LearningService.getPromptContext` メソッド
- OCR API (`/api/ocr`) での呼び出し

## 3. Inputs / Outputs
- **Input**: `bankId` (string)
- **Output**: `promptText` (string) - LLMへの指示プロンプトの一部

## 4. Rules
1. **方向性**: `Client` (通帳記載) が入力で、`Description` (摘要ラベル) が出力であることを明示する。
   - `When you see "{client}", map it to description="{description}"`
2. **正規化併記**: プロンプト内で `normalized: {clientNorm}` も提示し、LLMに揺らぎ許容を教える。
3. **Context Limit**: 全件ではなく、直近または関連性の高い 20〜50件 に絞り込む（トークン節約）。
4. **Scope**: 指定された `bankId` のルールのみを展開する。

## 5. DoD
- 生成されるプロンプトが `Client -> Description` の形式になっていること。
- 指定した `bankId` のルールが含まれていること。
- AIがこのヒントに従って、未知の類似行に対しても正しい `Description` を推論できること（検証フェーズ）。

## 6. Test Cases
- Rule: `ABC` -> `振込`
- OCR Input: `ABC 12月`
- Expectation: AI suggests `description="振込"` based on the Few-shot.

## 7. Anti-Patterns
- `Description -> Client` の逆方向で教えてしまうこと（AIが混乱する）。
- 全データを無制限にプロンプトに突っ込むこと（コスト・制限オーバー）。

## 8. Notes
- 保存は無制限で行うが、プロンプト生成時は `slice(-20)` などで制限をかける。将来は Vector Search などで関連ルールを抽出するのが理想。
- **Verifier Note**: 生成プロンプト内で `client="..."` の形式ではなく、直接 `When you see "..."` と展開される実装になっているが、意図通りなのでテスト側を合わせた。
