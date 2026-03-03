---
name: testing-webapp
description: PlaywrightによるWebアプリテスト・自動化（Reconnaissance-Then-Action）
---
# Webアプリテスト（Playwright）

## アプローチ選択
- 静的HTML → ファイル直接読みでセレクタ特定
- 動的Webアプリ → サーバー未起動なら `scripts/with_server.py`、起動済みならReconnaissance-Then-Action

## サーバー管理
```bash
python scripts/with_server.py --server "npm run dev" --port 5173 -- python your_automation.py
# 複数サーバー: --server と --port を繰り返し指定
```

## Reconnaissance-Then-Action
1. ナビゲート + `page.wait_for_load_state('networkidle')`
2. スクリーンショット or DOM検査でセレクタ特定
3. 発見したセレクタでアクション実行

```python
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto('http://localhost:5173')
    page.wait_for_load_state('networkidle')
    # 自動化ロジック
    browser.close()
```

## ベストプラクティス
- `sync_playwright()` 使用、ブラウザは必ず閉じる
- セレクタ: `text=`, `role=`, CSS, ID を使用
- 待機: `wait_for_selector()` or `wait_for_timeout()`
