---
name: detecting-test-framework
description: プロジェクトのテストフレームワークと設定を自動検出する。TDD開始前の環境把握時に使う。
model: sonnet
---

# テストFW自動検出

## 手順
1. package.json / requirements.txt / Cargo.toml 等を確認
2. テスト関連パッケージを特定（jest, vitest, pytest, etc.）
3. 既存テストファイルのパターンを検出
4. テスト設定ファイル（jest.config, vitest.config等）を確認

## 出力
- 検出されたFW名
- 設定ファイルパス
- テストファイル命名規則
- テスト実行コマンド
