---
name: serializing-objects-safely
description: 任意のJavaScriptオブジェクトを、循環参照・DOMノード・巨大データに対処しつつ安全にJSON化する再帰シリアライザ。
trigger: ログ出力やテレメトリ送信で、構造が不明な任意オブジェクトをJSON文字列に変換する必要があるとき。
---

# 任意オブジェクトの安全なJSON化

## Purpose

`JSON.stringify`は循環参照で例外を投げ、DOMノードで無限展開し、巨大オブジェクトでメモリを食い潰す。ロガーやテレメトリなど「何が来るかわからない」入力を安全にシリアライズする汎用関数が必要。

## Context

Background Service Worker / Content Script / Popup など、あらゆるコンテキストのログ出力で使用される。特にChrome拡張では:
- Content ScriptがDOMノードを誤ってログに渡す可能性
- Errorオブジェクトは`JSON.stringify`で空オブジェクト`{}`になる
- chrome API のレスポンスが循環参照を含む場合がある

## Pattern

```typescript
const MAX_DATA_SIZE = 5000;
const MAX_DEPTH = 5;

export function safeSerialize(
  value: unknown,
  depth: number = 0,
  seen: WeakSet<object> = new WeakSet()
): unknown {
  // === プリミティブ型: そのまま返す（文字列のみサイズ制限） ===
  if (value === null || value === undefined) return value;
  if (typeof value === "boolean" || typeof value === "number" || typeof value === "string") {
    if (typeof value === "string" && value.length > MAX_DATA_SIZE) {
      return value.slice(0, MAX_DATA_SIZE) + `...[truncated ${value.length - MAX_DATA_SIZE} chars]`;
    }
    return value;
  }

  // === 深さ制限 ===
  if (depth > MAX_DEPTH) {
    return "[max depth exceeded]";
  }

  // === Errorオブジェクト: 必要なプロパティだけ抽出 ===
  if (value instanceof Error) {
    return {
      name: value.name,
      message: value.message,
      stack: value.stack?.slice(0, 1000),
    };
  }

  // === DOMノード: nodeNameだけ出力 ===
  if (typeof Node !== "undefined" && value instanceof Node) {
    return `[DOM: ${value.nodeName}]`;
  }

  // === 関数: 名前だけ出力 ===
  if (typeof value === "function") {
    return `[Function: ${value.name || "anonymous"}]`;
  }

  // === オブジェクト・配列 ===
  if (typeof value === "object") {
    // 循環参照チェック（WeakSetで追跡）
    if (seen.has(value)) {
      return "[circular reference]";
    }
    seen.add(value);

    // 配列: 先頭50要素まで
    if (Array.isArray(value)) {
      const arr = value.slice(0, 50).map((item) => safeSerialize(item, depth + 1, seen));
      if (value.length > 50) {
        arr.push(`...[${value.length - 50} more items]`);
      }
      return arr;
    }

    // オブジェクト: 先頭50キーまで + サイズ制限
    const result: Record<string, unknown> = {};
    const keys = Object.keys(value);
    let serializedSize = 0;

    for (const key of keys.slice(0, 50)) {
      const serialized = safeSerialize((value as Record<string, unknown>)[key], depth + 1, seen);
      const serializedStr = JSON.stringify(serialized) || "";
      serializedSize += serializedStr.length;

      if (serializedSize > MAX_DATA_SIZE) {
        result[key] = "[truncated due to size]";
        result["__truncated__"] = true;
        break;
      }
      result[key] = serialized;
    }

    if (keys.length > 50) {
      result["__moreKeys__"] = keys.length - 50;
    }

    return result;
  }

  // === その他: 文字列化 ===
  return String(value);
}
```

### 防御レイヤーの一覧

| レイヤー | 対象 | 制限値 | 理由 |
|----------|------|--------|------|
| 文字列サイズ | string | 5000文字 | Base64画像データ等の混入防止 |
| 再帰深度 | ネスト | 5階層 | スタックオーバーフロー防止 |
| 循環参照 | object | WeakSet | TypeError防止 |
| キー数 | object | 50個 | ログ肥大化防止 |
| 配列要素数 | array | 50個 | ログ肥大化防止 |
| 累積サイズ | object全体 | 5000文字 | メモリ過消費防止 |
| DOMノード | Node | nodeNameのみ | 無限展開防止 |
| Error | Error | name/message/stack | 空オブジェクト化防止 |

## Rules

- `WeakSet`を使うこと。`Set`だとオブジェクトがGCされなくなる。
- `seen`は呼び出し元から渡す設計にし、1回のシリアライズ呼び出し全体で共有する。
- Error の`stack`は1000文字で切る。フルスタックトレースはログに不要。
- DOMノードは`typeof Node !== "undefined"`で存在チェックする（Service Workerには`Node`がない）。
- 戻り値は`unknown`型。`JSON.stringify`可能な値を返す設計だが、型安全のため`unknown`で返す。

## Anti-Patterns

- `JSON.stringify`に`replacer`引数だけで対処する（循環参照は検出できるが、深さ制限やサイズ制限ができない）
- `try { JSON.stringify(v) } catch { return "[error]" }`（情報が完全に失われる）
- `seen`に`Set`を使う（オブジェクト参照がGCされない）
- DOMノードの存在チェックなしに`instanceof Node`する（Service Workerでは`Node`未定義でReferenceError）

## Notes

- この関数は「何が来ても壊れない」ことが最優先。情報の完全性よりも安全性を重視する設計。
- `safeSerialize`の戻り値をさらに`JSON.stringify`してログに出力する想定。戻り値自体はプレーンオブジェクト。
- 累積サイズチェックは`JSON.stringify(serialized)`で行うため、各キーの処理にオーバーヘッドがある。パフォーマンスクリティカルなパスでは使用を避ける。
