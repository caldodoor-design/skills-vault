# Evaluation Criteria (Definition of Done)

Smart Passbook Engine の品質ゲート。
Sprintごとの「勝ち負け」をここで判定する。

---

## ✅ 機能ゲート（必須）
1. ERROR 行が 0 件（Export Gate）
2. WARNING 行がある場合は明示承認が必要（Human-in-the-Loop）
3. IGNORE 行はエクスポート対象外

---

## ✅ Sprint 3 主要KPI（銀行テンプレ）
- 列安定性：date / withdraw / deposit / balance が想定列に入っている体感がある
- 崩壊耐性：ヘッダ検出失敗時でも Virtual Grid + WARNING で復旧する
- 未登録銀行：default profile で落ちずに動く

---

## ✅ 安全性
- “自動修正”は禁止が基本。Strict Auto-Confirm 条件以外は WARNING で止める。
- bankId未設定・未登録でもクラッシュしない（default fallback）

---

## ✅ パフォーマンス
- 1ページ解析〜表示：15秒以内（目標）
- リトライ：最大3回指数バックオフ

---

## ✅ 完了条件（Sprint共通）
- verification_report.md が作成されている
- 主要E2EがPASS
- 次Sprintに入る前に「Known Issues」が整理されている
