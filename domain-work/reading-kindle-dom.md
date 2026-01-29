---
name: reading-kindle-dom
description: Kindle Cloud Readerの主要DOM要素（Canvas/ナビゲーション/ローディング等）のセレクタを一元管理する。Kindleページの要素取得やセレクタ参照時に使う。
skills: [analyzing-kindle-dom-heuristics]
---

# Kindle Cloud Reader DOM セレクタ辞書

## 目的

Kindle Cloud Reader固有のDOM構造に対応するセレクタを一元管理する。

---

## セレクタ定義

### CANVAS_SELECTOR (Content Target)

**用途:** 書籍コンテンツが描画されるCanvas要素

**セレクタ:**
```
iframe[id^='column_']
```

**Inner Image:**
```
img
```

**備考:**
- コンテンツは `#kr-renderer` 内の複数のIframe (`column_0_frame_0` 等) に分割されて描画される
- 各Iframe内の `img` タグが実体

---

### NEXT_BUTTON_SELECTOR (Navigation)

**用途:** 次のページへ進むボタン/クリック領域

**セレクタ:**
```
#kr-chevron-right
```

**Container:**
```
.kr-chevron-container-right
```

**備考:**
- コンテナ全体がクリック可能な場合があるが、ID指定の方が確実

---

### PREV_BUTTON_SELECTOR

**用途:** 前のページへ戻るボタン/クリック領域

**セレクタ:**
```
#kr-chevron-left
```

**備考:**
- NEXT_BUTTON_SELECTORと対称構造と推定

---

### LOADING_SPINNER_SELECTOR (State Detection)

**用途:** ページ読み込み中に表示されるローディングインジケータ

**セレクタ:**
```
.loader
```

**備考:**
- 画面全体を覆うオーバーレイ
- `display: none` で非表示になる挙動を監視すること

---

### IFRAME_SELECTOR

**用途:** コンテンツが埋め込まれるiframe

**セレクタ:**
```
iframe[id^='column_']
```

**備考:**
- `#kr-renderer` 内に配置される
- 複数のiframeに分割描画される構造

---

### PAGE_NUMBER_SELECTOR

**用途:** 現在のページ番号/位置表示要素

**セレクタ:**
```
（未調査）
```

**備考:**
-

---

### BOOK_TITLE_SELECTOR

**用途:** 現在開いている書籍のタイトル表示要素

**セレクタ:**
```
（未調査）
```

**備考:**
-

---

## 注意事項

- Kindle Cloud ReaderのDOM構造はアップデートで変更される可能性がある
- セレクタが動作しなくなった場合は再調査が必要
- クラス名に動的ハッシュが含まれる場合は属性セレクタ等を検討する
