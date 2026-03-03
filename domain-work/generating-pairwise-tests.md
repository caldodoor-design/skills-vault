---
name: generating-pairwise-tests
description: PICTペアワイズテストで組合せを最小化しつつ包括的カバレッジを達成するテスト設計
trigger: 複数パラメータの組合せテスト設計時
---
# PICTテストデザイナー

## フロー
1. パラメータ・値・制約・期待結果を特定
2. PICTモデル生成: `ParameterName: Value1, Value2`
3. 制約: `IF [P1] = V THEN [P2] <> V2;`
4. テストケース生成（オンライン: pairwise.yuuniworks.com / ローカル: microsoft/pict）
5. 期待結果を決定しテストスイート化

## パラメータ設計
- 同値分割: `FileSize: Small, Medium, Large`
- 境界値: `Age: 0, 17, 18, 65, 66`
- 負値: `Amount: ~-1, 0, 100, ~999999`

## よくあるモデル
```
# Webフォーム
Name: Valid, Empty, TooLong
Email: Valid, Invalid, Empty
Password: Strong, Weak, Empty

# API
HTTPMethod: GET, POST, PUT, DELETE
Auth: Valid, Invalid, Missing
ContentType: JSON, XML, FormData
```

## ポイント
- ペアワイズで網羅テストから80-90%削減
- order 2から開始、必要に応じて上げる
- 関連パラメータはサブモデルでグループ化
- 期待結果は具体的に（「動作する」は不可）
