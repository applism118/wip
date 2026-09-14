# バクラク申請承認: 顧客差し戻し・修正 × AI判断トレース 調査設計

対象: 承認AI（`APPROVAL_REVIEW_RESULT_*`）と申請（`REQUESTS` / `REQUEST_DETAILS`）を Snowflake 上で紐付け、
「LayerX対応以降に顧客が差し戻し・修正した申請」について AI 側で何が起きていたかをトレースし、原因分類 → 打ち手に落とすための設計メモ。

> **前提（重要）**
> - この調査時点で Snowflake 接続と `layerone` / `layerone-dbt-snowflake` リポジトリの参照ができなかったため、
>   カラム名は **添付のテーブル一覧と一般的な命名からの推定** です。`-- 要確認` を付けた箇所は §1 の探索クエリで実際のカラム名に置き換えてください。
> - 「LayerX対応以降」の起点は `:since` パラメータ（日付）として扱っています。

---

## 0. テーブル群の役割（推定）

承認AIは「ルール検索 → カテゴリ分類 → ルール照合 → 修正提案 → 人のフィードバック」という多段パイプラインで、各段の出力が別テーブルに保存されている構造に見える。

| テーブル | 役割（推定） | 主なキー（推定） |
|---|---|---|
| `APPROVAL_REVIEW_RESULTS` | AIレビュー1回分のヘッダ。申請ごとに再申請などで複数行あり得る | `ID`, `REQUEST_ID`, `TENANT_ID`, `CREATED_AT`, 総合判定 |
| `APPROVAL_REVIEW_RESULT_RETRIEVED_RULES` | レビュー時に検索（RAG）でヒットしたルール候補 | `APPROVAL_REVIEW_RESULT_ID`, `APPROVAL_RULE_ID`, スコア/順位 |
| `APPROVAL_REVIEW_RESULT_RETRIEVE_FEEDBACKS` | 検索結果に対する人のフィードバック | `APPROVAL_REVIEW_RESULT_ID`, `APPROVAL_RULE_ID`, 良/悪 |
| `APPROVAL_REVIEW_RESULT_CLASSIFY_CATEGORIES` | 申請に付与されたルールカテゴリ分類 | `APPROVAL_REVIEW_RESULT_ID`, `APPROVAL_RULE_CATEGORY_ID` |
| `APPROVAL_REVIEW_RESULT_CLASSIFY_CATEGORY_FEEDBACKS` | 分類に対する人のフィードバック | 同上 |
| `APPROVAL_REVIEW_RESULT_CHECKED_RULES` | 実際に照合したルールと判定（OK/NG/判断不能 等）と理由 | `APPROVAL_REVIEW_RESULT_ID`, `APPROVAL_RULE_ID`, 判定, 理由テキスト, `REQUEST_DETAIL_ID`? |
| `APPROVAL_REVIEW_RESULT_CORRECTION_SUGGESTIONS` | 申請者向けの修正提案（自由文） | `APPROVAL_REVIEW_RESULT_ID`, 対象項目, 提案文 |
| `APPROVAL_REVIEW_RESULT_STRUCTURED_CORRECTION_CANDIDATES` | 構造化修正の候補（項目・修正前後の値） | `APPROVAL_REVIEW_RESULT_ID`, `REQUEST_DETAIL_ID`?, `FIELD`, `BEFORE`, `AFTER` |
| `APPROVAL_REVIEW_RESULT_STRUCTURED_CORRECTIONS` | 候補のうち採用された／確定した構造化修正 | 同上 + 採用フラグ/採用者 |
| `APPROVAL_REVIEW_RESULT_DETAIL_SPLIT_CANDIDATES` | 明細分割の候補 | `APPROVAL_REVIEW_RESULT_ID`, `REQUEST_DETAIL_ID` |
| `APPROVAL_REVIEW_RESULT_DETAIL_SPLIT_CORRECTIONS` | 実施された明細分割 | 同上 |
| `APPROVAL_REVIEW_RESULT_FEEDBACKS` | レビュー結果全体に対する人のフィードバック（差し戻し理由に最も近い信号） | `APPROVAL_REVIEW_RESULT_ID`, 評価, コメント, `USER_ID` |
| `APPROVAL_RULES` | テナントが定義した承認ルール本文 | `ID`, `TENANT_ID`, `APPROVAL_RULE_CATEGORY_ID`, 本文, 有効期間 |
| `APPROVAL_RULE_CATEGORIES` | ルールカテゴリマスタ | `ID`, 名称 |

`REQUEST_DETAILS` に紐づくのは、明細粒度で動く `CHECKED_RULES` / `STRUCTURED_CORRECTION*` / `DETAIL_SPLIT_*` のはず（`REQUEST_DETAIL_ID` 列の有無を §1 で確認）。

---

## 1. まず実カラムを確定させる探索クエリ

```sql
-- 1-1. 承認AI系テーブルの全カラム
SELECT table_schema, table_name, column_name, data_type, ordinal_position
FROM   <DB>.INFORMATION_SCHEMA.COLUMNS
WHERE  table_name ILIKE 'APPROVAL_REVIEW_RESULT%'
   OR  table_name IN ('APPROVAL_RULES', 'APPROVAL_RULE_CATEGORIES')
ORDER BY table_name, ordinal_position;

-- 1-2. 申請 ↔ AI をつなぐキー候補（REQUEST_ID / REQUEST_DETAIL_ID を持つテーブル）
SELECT table_name, column_name
FROM   <DB>.INFORMATION_SCHEMA.COLUMNS
WHERE  table_name ILIKE 'APPROVAL_REVIEW_RESULT%'
  AND  column_name ILIKE ANY ('%REQUEST_ID%', '%REQUEST_DETAIL_ID%', '%RULE_ID%', '%CATEGORY_ID%')
ORDER BY table_name, column_name;

-- 1-3. 差し戻し・修正を表すテーブル/カラムの候補
SELECT table_name, column_name, data_type
FROM   <DB>.INFORMATION_SCHEMA.COLUMNS
WHERE  table_name ILIKE 'REQUEST%'
  AND (column_name ILIKE ANY ('%STATUS%', '%REMAND%', '%REJECT%', '%RETURN%', '%ACTION%', '%EVENT%', '%HISTORY%', '%REVISION%', '%VERSION%'))
ORDER BY table_name, column_name;

-- 1-4. 判定・フィードバックの取りうる値（enum の実態把握）
SELECT <判定列>, COUNT(*) FROM APPROVAL_REVIEW_RESULT_CHECKED_RULES GROUP BY 1;   -- 要確認
SELECT <評価列>, COUNT(*) FROM APPROVAL_REVIEW_RESULT_FEEDBACKS     GROUP BY 1;   -- 要確認
```

---

## 2. 結合モデル（トレースの骨格）

```
REQUESTS ──1:N── REQUEST_DETAILS
   │
   │ REQUEST_ID
   ▼
APPROVAL_REVIEW_RESULTS (申請1件につき0..N回のAIレビュー)
   ├─ RETRIEVED_RULES ──► APPROVAL_RULES ──► APPROVAL_RULE_CATEGORIES
   ├─ CLASSIFY_CATEGORIES ─────────────────► APPROVAL_RULE_CATEGORIES
   ├─ CHECKED_RULES ──────► APPROVAL_RULES   (REQUEST_DETAIL_ID があれば明細粒度)
   ├─ CORRECTION_SUGGESTIONS
   ├─ STRUCTURED_CORRECTION_CANDIDATES ─► STRUCTURED_CORRECTIONS
   ├─ DETAIL_SPLIT_CANDIDATES ──────────► DETAIL_SPLIT_CORRECTIONS
   └─ FEEDBACKS / RETRIEVE_FEEDBACKS / CLASSIFY_CATEGORY_FEEDBACKS
   │
   ▼
差し戻し・修正イベント（REQUEST の履歴/承認ステップ系テーブル）
```

**時系列の対応付けが肝**: 申請は差し戻し → 修正 → 再申請で `APPROVAL_REVIEW_RESULTS` が複数行になる。
「その差し戻しの直前に人が見ていたAI判定」を取るには、`review.CREATED_AT <= remand.EVENT_AT` で最新1件を選ぶ（§3 Q2）。

---

## 3. クエリ雛形

### Q1. LayerX対応以降の顧客差し戻し・修正一覧（ベース）

差し戻し・修正の実体テーブルは §1-3 で確定させる。以下は「申請履歴テーブルに action 種別がある」想定。

```sql
WITH remands AS (
  SELECT
    h.REQUEST_ID,
    r.TENANT_ID,
    h.ACTION_TYPE,            -- 要確認: 'REMAND' / 'RETURN' / 'EDIT' 等
    h.ACTED_BY_USER_ID,       -- 要確認
    h.COMMENT,                -- 要確認: 差し戻し理由テキスト
    h.CREATED_AT AS EVENT_AT
  FROM REQUEST_HISTORIES h    -- 要確認: 実テーブル名
  JOIN REQUESTS r ON r.ID = h.REQUEST_ID
  WHERE h.ACTION_TYPE IN ('REMAND', 'EDIT')      -- 要確認
    AND h.CREATED_AT >= :since
    AND r.DELETED_AT IS NULL
    AND r.TENANT_ID <> :layerx_tenant_id          -- 「顧客による」= 自社テナント除外
)
SELECT * FROM remands ORDER BY EVENT_AT DESC;
```

### Q2. 差し戻し直前のAIレビュー結果を1:1で付ける

```sql
WITH remands AS ( /* Q1 */ ),
review_before_remand AS (
  SELECT
    rm.*,
    rr.ID           AS REVIEW_RESULT_ID,
    rr.CREATED_AT   AS REVIEWED_AT,
    rr.<総合判定列>  AS AI_VERDICT,          -- 要確認: approve / needs_fix / unknown 等
    ROW_NUMBER() OVER (PARTITION BY rm.REQUEST_ID, rm.EVENT_AT
                       ORDER BY rr.CREATED_AT DESC) AS rn
  FROM remands rm
  LEFT JOIN APPROVAL_REVIEW_RESULTS rr
    ON  rr.REQUEST_ID = rm.REQUEST_ID
    AND rr.CREATED_AT <= rm.EVENT_AT
)
SELECT * FROM review_before_remand WHERE rn = 1 OR REVIEW_RESULT_ID IS NULL;
```

`REVIEW_RESULT_ID IS NULL` の行 = **AIレビューが走っていない差し戻し**（対象外 or 実行漏れ）。まずここで母数を分ける。

### Q3. その時AIが「検索した／照合した」ルールと判定（ルールトレース）

```sql
WITH base AS ( /* Q2 の rn = 1 */ )
SELECT
  b.REQUEST_ID, b.EVENT_AT, b.AI_VERDICT, b.COMMENT AS REMAND_COMMENT,
  ar.ID            AS RULE_ID,
  arc.NAME         AS RULE_CATEGORY,           -- 要確認
  ar.<本文列>       AS RULE_TEXT,               -- 要確認
  rr.<順位/スコア列> AS RETRIEVE_RANK,            -- 要確認
  cr.<判定列>       AS CHECK_RESULT,            -- 要確認: OK / NG / UNKNOWN
  cr.<理由列>       AS CHECK_REASON,            -- 要確認
  cr.REQUEST_DETAIL_ID                          -- 要確認: 明細粒度なら
FROM base b
LEFT JOIN APPROVAL_REVIEW_RESULT_RETRIEVED_RULES rr
       ON rr.APPROVAL_REVIEW_RESULT_ID = b.REVIEW_RESULT_ID
LEFT JOIN APPROVAL_REVIEW_RESULT_CHECKED_RULES cr
       ON cr.APPROVAL_REVIEW_RESULT_ID = b.REVIEW_RESULT_ID
      AND cr.APPROVAL_RULE_ID = rr.APPROVAL_RULE_ID
LEFT JOIN APPROVAL_RULES ar            ON ar.ID = COALESCE(cr.APPROVAL_RULE_ID, rr.APPROVAL_RULE_ID)
LEFT JOIN APPROVAL_RULE_CATEGORIES arc ON arc.ID = ar.APPROVAL_RULE_CATEGORY_ID
ORDER BY b.EVENT_AT DESC, RETRIEVE_RANK;
```

見るポイント:
- 検索されたが照合されなかった（`rr` あり `cr` なし）→ 絞り込み段の取りこぼし
- 照合して OK 判定だが人が差し戻した → 判定ミス（`CHECK_REASON` を読む）
- そもそも該当ルールが `APPROVAL_RULES` に存在しない → ルール未整備（AIの問題ではない）

### Q4. AIの修正提案 vs 実際の修正（明細粒度）

```sql
WITH base AS ( /* Q2 の rn = 1 */ )
SELECT
  b.REQUEST_ID, b.EVENT_AT,
  rd.ID                 AS REQUEST_DETAIL_ID,
  cs.<提案文列>          AS SUGGESTION_TEXT,        -- 要確認
  scc.<項目列>           AS FIELD,                  -- 要確認
  scc.<修正前列>          AS AI_BEFORE,
  scc.<修正後列>          AS AI_AFTER,
  sc.ID IS NOT NULL      AS AI_CORRECTION_ADOPTED,  -- 候補が確定に昇格したか
  dsc.ID IS NOT NULL     AS DETAIL_SPLIT_SUGGESTED,
  dscr.ID IS NOT NULL    AS DETAIL_SPLIT_APPLIED
FROM base b
JOIN REQUEST_DETAILS rd ON rd.REQUEST_ID = b.REQUEST_ID AND rd.DELETED_AT IS NULL
LEFT JOIN APPROVAL_REVIEW_RESULT_CORRECTION_SUGGESTIONS cs
       ON cs.APPROVAL_REVIEW_RESULT_ID = b.REVIEW_RESULT_ID
LEFT JOIN APPROVAL_REVIEW_RESULT_STRUCTURED_CORRECTION_CANDIDATES scc
       ON scc.APPROVAL_REVIEW_RESULT_ID = b.REVIEW_RESULT_ID AND scc.REQUEST_DETAIL_ID = rd.ID
LEFT JOIN APPROVAL_REVIEW_RESULT_STRUCTURED_CORRECTIONS sc
       ON sc.<候補ID列> = scc.ID                                                -- 要確認
LEFT JOIN APPROVAL_REVIEW_RESULT_DETAIL_SPLIT_CANDIDATES dsc
       ON dsc.APPROVAL_REVIEW_RESULT_ID = b.REVIEW_RESULT_ID AND dsc.REQUEST_DETAIL_ID = rd.ID
LEFT JOIN APPROVAL_REVIEW_RESULT_DETAIL_SPLIT_CORRECTIONS dscr
       ON dscr.<候補ID列> = dsc.ID;                                              -- 要確認
```

「AIは正しく修正を提案していたのに申請者が反映せず差し戻された」= 提案の見せ方/導線の問題として切り分けられる。

### Q5. 人のフィードバックを差し戻しに突き合わせる

```sql
SELECT
  b.REQUEST_ID, b.EVENT_AT, b.AI_VERDICT,
  f.<評価列>  AS FEEDBACK_RATING,        -- 要確認
  f.<本文列>  AS FEEDBACK_COMMENT,
  rf.APPROVAL_RULE_ID AS RETRIEVE_FB_RULE_ID, rf.<評価列> AS RETRIEVE_FB,
  cf.APPROVAL_RULE_CATEGORY_ID AS CATEGORY_FB_ID, cf.<評価列> AS CATEGORY_FB
FROM base b
LEFT JOIN APPROVAL_REVIEW_RESULT_FEEDBACKS                   f  ON f.APPROVAL_REVIEW_RESULT_ID  = b.REVIEW_RESULT_ID
LEFT JOIN APPROVAL_REVIEW_RESULT_RETRIEVE_FEEDBACKS          rf ON rf.APPROVAL_REVIEW_RESULT_ID = b.REVIEW_RESULT_ID
LEFT JOIN APPROVAL_REVIEW_RESULT_CLASSIFY_CATEGORY_FEEDBACKS cf ON cf.APPROVAL_REVIEW_RESULT_ID = b.REVIEW_RESULT_ID;
```

### Q6. 集計: 原因バケット別の件数

```sql
WITH joined AS ( /* Q2 + Q3 を request 粒度に集約 */
  SELECT
    b.REQUEST_ID, b.TENANT_ID, b.EVENT_AT, b.AI_VERDICT, b.REVIEW_RESULT_ID,
    COUNT(rr.APPROVAL_RULE_ID)                                        AS N_RETRIEVED,
    COUNT(cr.APPROVAL_RULE_ID)                                        AS N_CHECKED,
    COUNT_IF(cr.<判定列> = 'NG')                                       AS N_NG,      -- 要確認
    COUNT_IF(cr.<判定列> = 'UNKNOWN')                                  AS N_UNKNOWN  -- 要確認
  FROM base b
  LEFT JOIN APPROVAL_REVIEW_RESULT_RETRIEVED_RULES rr ON rr.APPROVAL_REVIEW_RESULT_ID = b.REVIEW_RESULT_ID
  LEFT JOIN APPROVAL_REVIEW_RESULT_CHECKED_RULES   cr ON cr.APPROVAL_REVIEW_RESULT_ID = b.REVIEW_RESULT_ID
  GROUP BY 1,2,3,4,5
)
SELECT
  CASE
    WHEN REVIEW_RESULT_ID IS NULL           THEN 'A. AI未実行'
    WHEN N_NG > 0                            THEN 'B. AIも指摘していた（人は指摘を見て差し戻し）'
    WHEN N_UNKNOWN > 0                       THEN 'C. AIが判断不能だった'
    WHEN N_CHECKED = 0 AND N_RETRIEVED > 0   THEN 'D. 検索はしたが照合対象に残らなかった'
    WHEN N_RETRIEVED = 0                     THEN 'E. 該当ルールを検索できず/ルール未定義'
    ELSE                                          'F. AIは全てOK判定 → 見逃し'
  END AS CAUSE_BUCKET,
  COUNT(*) AS N_REMANDS,
  COUNT(DISTINCT TENANT_ID) AS N_TENANTS
FROM joined
GROUP BY 1 ORDER BY N_REMANDS DESC;
```

### Q7. ルール・カテゴリ別の「見逃し率」

```sql
-- 照合したルール単位で、OK判定なのに後で差し戻された割合
SELECT
  arc.NAME AS RULE_CATEGORY, ar.ID AS RULE_ID,
  COUNT(*)                                          AS N_CHECKED,
  COUNT_IF(cr.<判定列> = 'OK' AND rm.REQUEST_ID IS NOT NULL) AS N_OK_BUT_REMANDED,
  N_OK_BUT_REMANDED / NULLIF(N_CHECKED, 0)          AS MISS_RATE
FROM APPROVAL_REVIEW_RESULT_CHECKED_RULES cr
JOIN APPROVAL_REVIEW_RESULTS rr ON rr.ID = cr.APPROVAL_REVIEW_RESULT_ID
JOIN APPROVAL_RULES ar          ON ar.ID = cr.APPROVAL_RULE_ID
JOIN APPROVAL_RULE_CATEGORIES arc ON arc.ID = ar.APPROVAL_RULE_CATEGORY_ID
LEFT JOIN remands rm ON rm.REQUEST_ID = rr.REQUEST_ID AND rm.EVENT_AT > rr.CREATED_AT
WHERE rr.CREATED_AT >= :since
GROUP BY 1,2 ORDER BY MISS_RATE DESC, N_CHECKED DESC;
```

---

## 4. 原因分類 → 打ち手の対応表

| バケット | データ上の見え方 | 想定原因 | 打ち手 |
|---|---|---|---|
| A. AI未実行 | `REVIEW_RESULT_ID IS NULL` | 対象外テナント/フォーム、実行エラー | 実行条件・エラーログの確認。AI以前の問題として母数から除く |
| B. AIも指摘していた | `N_NG > 0` | AIは機能。申請者が指摘を無視/気づかず提出 | 提出前ブロック・修正提案の表示強化（Q4で「提案あり×未反映」を数える） |
| C. 判断不能 | `N_UNKNOWN > 0` | 添付・明細の情報不足、ルール文の曖昧さ | `CHECK_REASON` を読みルール文の改善提案 or 入力必須項目の追加 |
| D. 検索→照合で脱落 | `N_RETRIEVED>0, N_CHECKED=0` | カテゴリ分類の誤り、絞り込み閾値 | `CLASSIFY_CATEGORY_FEEDBACKS` と突合し分類モデル/閾値を調整 |
| E. 検索できず | `N_RETRIEVED = 0` | ルール未登録、検索クエリ表現の乖離 | 差し戻しコメントからルール化候補を抽出して顧客に提案（ルール整備支援） |
| F. 全てOK→見逃し | `N_CHECKED>0, N_NG=0` | 判定ロジックの誤り | `CHECK_REASON` と差し戻しコメントを並べて誤判定パターンを分類 → プロンプト/ルール改善 |

差し戻しコメント（Q1 `COMMENT`）は全バケット共通の一次情報なので、バケット × コメントの一覧をまず人が読む。
その上で B/F は AI 改善、C/E は顧客側ルール整備支援、A/D は運用/設定の問題、と担当を分ける。

---

## 5. 次にやること

1. §1 の探索クエリを実行し、`-- 要確認` のカラム名を確定する（特に 差し戻しイベントの実テーブル、`CHECKED_RULES` の判定 enum、`REQUEST_DETAIL_ID` の有無）。
2. Q1 → Q2 の順で母数を確認（差し戻し件数、うち AI レビューありの件数）。
3. Q6 のバケット分布を出し、大きいバケットから Q3/Q4/Q5 で個票を読む。
4. 実カラム名が分かれば、このメモの雛形を dbt モデル（`layerone-dbt-snowflake`）化して定点観測に乗せる。
