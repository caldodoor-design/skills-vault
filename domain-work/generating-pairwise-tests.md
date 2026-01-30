---
name: generating-pairwise-tests
description: PICT（ペアワイズ独立組合せテスト）を用いて体系的にテストケースを設計する。複数入力パラメータを持つ機能のテスト設計時に使用。
trigger: テストケース設計、組合せテスト、テストマトリクス作成、パラメータ組合せの網羅
---

# PICTテストデザイナー

要件やコードからシステムを分析し、テストパラメータを特定、制約付きPICTモデルを生成し、ペアワイズテストケースを作成する。

## 使用タイミング

- 複数入力パラメータを持つ機能のテストケース設計
- 多数の組合せを持つ構成のテストスイート作成
- 最小テストケースで包括的カバレッジが必要な場合
- 要件分析からのテストシナリオ特定
- APIエンドポイント、Webフォーム、システム構成のテストマトリクス構築

## ワークフロー

### 1. 要件またはコードの分析

以下を特定する:
- パラメータ: 入力変数、設定オプション、環境要因
- 値: 各パラメータの取り得る値（同値分割を使用）
- 制約: ビジネスルール、技術的制限、パラメータ間の依存関係
- 期待される結果: 各組合せで何が起こるべきか

### 2. PICTモデルの生成

パラメータ定義と制約を記述:
ParameterName: Value1, Value2, Value3
IF [Parameter1] = Value THEN [Parameter2] <> OtherValue;

### 3. テストケースの生成

モデルをファイルに保存し、以下でテストケース作成:
- https://pairwise.yuuniworks.com/
- https://pairwise.teremokgames.com/
- PICTローカルインストール（公式リポジトリ参照）

### 4. 期待される出力の決定

ビジネス要件、コードロジック、有効/無効の組合せに基づいて期待結果を決定。

### 5. テストスイートのフォーマット

PICTモデル + Markdownテーブル + 期待出力を提供。

## ベストプラクティス

### パラメータ特定
- 説明的な名前: AuthMethod, UserRole, PaymentType
- 同値分割の適用: FileSize: Small, Medium, Large
- 境界値の含有: Age: 0, 17, 18, 65, 66
- エラーテスト用の負値: Amount: ~-1, 0, 100, ~999999

### 制約の記述
- 理由の文書化
- シンプルから段階的に追加
- 過剰/不足な制約を避ける

### 期待結果の定義
- 具体的に（「ログイン成功、ダッシュボードにリダイレクト」）
- 曖昧にしない（「動作する」「エラー」は不可）

### スケーラビリティ
- サブモデルで関連パラメータをグループ化
- 無関係な機能は別テストスイートに分離
- order 2（ペアワイズ）から開始
- ペアワイズテストは網羅テストから80-90%のケース削減

## 一般的なパターン

### Webフォームテスト
Name: Valid, Empty, TooLong
Email: Valid, Invalid, Empty
Password: Strong, Weak, Empty
Terms: Accepted, NotAccepted

### APIエンドポイントテスト
HTTPMethod: GET, POST, PUT, DELETE
Authentication: Valid, Invalid, Missing
ContentType: JSON, XML, FormData
PayloadSize: Empty, Small, Large

### 構成テスト
Environment: Dev, Staging, Production
CacheEnabled: True, False
LogLevel: Debug, Info, Error
Database: SQLite, PostgreSQL, MySQL

## トラブルシューティング

- テストケース未生成: 制約の過度な制限、構文確認、パラメータ名一致確認
- テストケース過多: order 2確認、サブモデル分割検討
- 無効な組合せ: 不足制約の追加、制約ロジック確認

## 元リポジトリ

- https://github.com/omkamal/pypict
- PICT公式: https://github.com/microsoft/pict
- pypict: https://github.com/kmaehashi/pypict
- オンラインツール: https://pairwise.yuuniworks.com/