---
name: testing-webapp
description: PlaywrightによるWebアプリケーションのテスト・自動化スキル。Reconnaissance-Then-ActionパターンでDOM探索→アクション実行。
trigger: Webアプリのテスト、ブラウザ自動化、UIテスト、Playwright操作が必要な時
---

# Webアプリケーションテスト

ローカルWebアプリケーションのテストにはPython Playwrightスクリプトを使用する。

> **元リポジトリ**: https://github.com/anthropics/skills

## 判断ツリー: アプローチ選択

```
ユーザータスク → 静的HTMLか？
    ├─ Yes → HTMLファイルを直接読んでセレクタ特定
    │         ├─ 成功 → Playwrightスクリプトでセレクタ使用
    │         └─ 失敗 → 動的として扱う（下記）
    │
    └─ No（動的Webアプリ） → サーバーは起動済みか？
        ├─ No → scripts/with_server.py を使用
        │
        └─ Yes → Reconnaissance-Then-Action:
            1. ナビゲート＋networkidle待機
            2. スクリーンショットまたはDOM検査
            3. レンダリング状態からセレクタ特定
            4. 発見したセレクタでアクション実行
```

## サーバー管理

```bash
# 単一サーバー
python scripts/with_server.py --server "npm run dev" --port 5173 -- python your_automation.py

# 複数サーバー（バックエンド＋フロントエンド）
python scripts/with_server.py \
  --server "cd backend && python server.py" --port 3000 \
  --server "cd frontend && npm run dev" --port 5173 \
  -- python your_automation.py
```

## Playwrightスクリプトの基本構造

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto('http://localhost:5173')
    page.wait_for_load_state('networkidle')  # 重要: JS実行完了を待つ
    # ... 自動化ロジック
    browser.close()
```

## Reconnaissance-Then-Actionパターン

1. **レンダリング済みDOMを検査**:
   ```python
   page.screenshot(path='/tmp/inspect.png', full_page=True)
   content = page.content()
   page.locator('button').all()
   ```

2. **検査結果からセレクタを特定**

3. **発見したセレクタでアクションを実行**

## よくある落とし穴

- **NG**: 動的アプリで`networkidle`を待たずにDOMを検査する
- **OK**: `page.wait_for_load_state('networkidle')` の後に検査する

## ベストプラクティス

- `sync_playwright()` を使用（同期スクリプト）
- ブラウザは必ず閉じる
- セレクタは具体的に: `text=`, `role=`, CSSセレクタ, IDを使用
- 適切な待機を入れる: `page.wait_for_selector()` または `page.wait_for_timeout()`
- スクリプトはブラックボックスとして使用し、コンテキストに読み込まない
