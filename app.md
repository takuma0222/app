# チャンバ状態チェック要件（v3）

## 1. 前提整理

### 1.1 システム間の情報分布

| 情報 | App | MES（予約） | MES（着工指示） |
|------|-----|-------------|-----------------|
| 装置ID | ○ | ○ | ○ |
| カードNo | ○ | ○ | ○ |
| 今回レシピID | × | **○** | **○** |
| 今回使用ポート（搬送レシピ） | × | × | **○** |
| 前回レシピID | **○** | × | **○** |
| 前回使用ポート | **○** | × | **○** |
| 前回完了時刻 | **○** | **○** | **○** |
| 装置の現在処理中ポート | **○** | × | × |
| レシピの想定処理時間 | **○**（マスタ） | × | × |

- App は装置ごとに **前回レシピID・前回使用ポート・前回完了時刻** を保持している。
- MES の着工指示にも **前回レシピID・前回使用ポート** が含まれる（App保持値と突合可能）。
- MES は **今回レシピID** を保持しているが、予約IFには含まれない。
- App→MES への予約は **装置 + カードNo** のみ（既存仕様）。
- MES の予約には搬送レシピ（使用ポート情報）が無い。
- **完了時刻は App・MES 双方が保持** しており、タイマー起点として使用する。

### 1.2 要件一覧

| # | 要件 | 概要 |
|---|------|------|
| R1 | 時間制約付き特定レシピ判定 | 特定レシピ群の前回完了から一定時間以内に次の着工を行わなければ NG |
| R2 | 介在レシピ許容 | 対象レシピ間に別レシピが挟まっても OK |
| R3 | ポート競合待ち制約 | 処理中ポートと今回使用ポートが異なる場合、処理完了を待たなければならない装置がある |
| R4 | 残時間 vs 処理時間チェック | 時間制約の残り時間が今回レシピの処理時間を下回る場合、着工を拒否する |

---

## 2. 設計方針

### 2.1 タイマー起点 = 完了時刻

- App・MES 双方が完了時刻を保持しているため、タイマー起点は **前回完了時刻** とする。
- 着工時刻ではなく完了時刻を使うことで、処理時間のばらつきに左右されない安定した判定が可能。

### 2.2 判定に使う情報ソース

| 判定 | 情報ソース | 取得タイミング |
|------|-----------|--------------|
| 時間制約（R1） | App保持の `last_complete_at` + 着工指示の今回レシピID | 着工指示受信時 |
| ポート競合（R3） | App保持の処理中ポート + 着工指示の今回使用ポート | 着工指示受信時 |
| 残時間チェック（R4） | App保持の `last_complete_at` + App マスタのレシピ想定処理時間 | 着工指示受信時 |

**MES IF の追加変更は不要。** 既存の着工指示 IF で十分。

### 2.3 判定粒度 ― 装置単位 vs ポート単位

| モード | 判定キー | ユースケース |
|--------|---------|-------------|
| 装置単位 | `equipment_id` + `recipe_group_id` | 装置全体で共有するプロセス条件の維持 |
| ポート単位 | `equipment_id` + `port_id` + `recipe_group_id` | チャンバ/ポートごとにコンディションが異なる場合 |

設定マスタの `scope` カラムで切り替える。

---

## 3. 業務ルール

### 3.1 判定タイミング

| フェーズ | 実施内容 | 判定可否 |
|---------|---------|---------|
| 予約登録（App→MES） | 装置 + カードNo を送信 | **最終判定不可**（今回使用ポート不明） |
| ↳ 参考チェック | App が前回状態を参照し、残時間が少ない場合に **警告ログ** を出力 | 参考のみ |
| 着工指示受信（MES→App） | 今回レシピ + ポート + 前回レシピ + 前回ポートを取得 | **最終判定実施** |

### 3.2 判定ロジック（全体フロー）

```
着工指示受信(equipment_id, card_no, recipe_id, port_ids[],
             prev_recipe_id, prev_port_ids[])
  │
  │ ── [判定1] ポート競合チェック (R3) ──
  │
  ├─ 装置がポート競合待ち対象か？
  │   └─ YES → 現在処理中ポートを取得
  │       ├─ 処理中ポートなし → PASS
  │       ├─ port_ids[] と処理中ポートが同一 → PASS
  │       └─ port_ids[] と処理中ポートが異なる → WAIT
  │           （処理完了まで着工を保留）
  │
  │ ── [判定2] 時間制約チェック (R1) ──
  │
  ├─ recipe_id → recipe_group_id を引き当て
  │   └─ 対象外 → ALLOW（制約なし）
  │
  ├─ scope に応じた判定キーで last_complete_at を取得
  │   └─ 存在しない（初回） → ALLOW
  │
  ├─ elapsed_sec = now - last_complete_at
  │   ├─ elapsed_sec <= max_interval_sec → PASS（→判定3へ）
  │   └─ elapsed_sec >  max_interval_sec → REJECT
  │
  │ ── [判定3] 残時間 vs 処理時間チェック (R4) ──
  │
  ├─ remaining_sec = max_interval_sec - elapsed_sec
  │   （＝次回の同グループレシピ着工までに許される残り時間）
  ├─ recipe_duration_sec = レシピの想定処理時間（App マスタ）
  │   ├─ remaining_sec >= recipe_duration_sec → ALLOW
  │   └─ remaining_sec <  recipe_duration_sec → REJECT
  │       （今回処理が完了する頃には時間制約を超過するため）
  │
  └─ ALLOW → last_complete_at は処理完了時に更新
     REJECT → ログ出力 + オペレータ通知
     WAIT   → 処理完了イベントを待って再判定
```

### 3.3 許可/拒否/待機ルール

| 条件 | 結果 | 理由コード |
|------|------|-----------|
| 対象グループ外のレシピ | ALLOW | − |
| 対象グループ初回着工（前回実績なし） | ALLOW | − |
| ポート競合あり（処理中ポート ≠ 今回ポート） | **WAIT** | `PORT_CONFLICT_WAIT` |
| 経過時間 ≤ 閾値、かつ残時間 ≥ 処理時間 | ALLOW | − |
| 経過時間 > 閾値 | **REJECT** | `TIME_WINDOW_EXCEEDED` |
| 経過時間 ≤ 閾値、だが残時間 < 処理時間 | **REJECT** | `INSUFFICIENT_REMAINING_TIME` |

### 3.4 介在レシピの扱い

- 対象グループ外のレシピはいつでも着工可能。
- 介在中も対象グループのタイマーは経過し続ける（起点は前回完了時刻で固定）。
- 介在レシピの完了は `last_complete_at` を更新しない。

```
例: レシピグループ A の制約 = 3600秒以内、レシピAの想定処理時間 = 600秒

  t=0     グループA完了 → last_complete_at = 0
  t=300   グループB着工 →                         (ALLOW, タイマー無関係)
  t=900   グループB完了 →                         (タイマー更新なし)
  t=1000  グループA着工 → elapsed=1000, remaining=2600, 処理=600
                          → 1000 ≤ 3600 かつ 2600 ≥ 600  (ALLOW)
  t=1600  グループA完了 → last_complete_at = 1600
  t=4800  グループA着工 → elapsed=3200, remaining=400, 処理=600
                          → 3200 ≤ 3600 だが 400 < 600    (REJECT: 残時間不足)
  t=5700  グループA着工 → elapsed=4100 > 3600             (REJECT: 時間超過)
```

### 3.5 ポート競合待ち（R3）の詳細

```
例: 装置Xはポート競合待ち対象

  処理中: ポートA でロット処理中
  着工指示: ポートB を使用するレシピ

  → ポートA ≠ ポートB → WAIT
  → ポートA の処理完了イベント受信後、再度判定を実行
  → 完了後、ポートB は空き → 時間制約チェック(R1)・残時間チェック(R4) に進む

  ※ 処理中ポートと今回ポートが同一の場合は WAIT 不要
```

---

## 4. データ設計（App側）

### 4.1 マスタ

#### `recipe_time_window_rule` — 時間制約ルール定義

| カラム | 型 | 説明 |
|--------|-----|------|
| `rule_id` | PK | ルールID |
| `equipment_id` | FK | 対象装置 |
| `recipe_group_id` | VARCHAR | 対象レシピグループ識別子 |
| `scope` | ENUM | `EQUIPMENT` / `PORT` |
| `max_interval_sec` | INT | 許容間隔（秒）※前回完了からの経過 |
| `enabled` | BOOL | 有効/無効 |
| `updated_at` | TIMESTAMP | 更新日時 |

#### `recipe_group_mapping` — レシピ→グループ紐付け

| カラム | 型 | 説明 |
|--------|-----|------|
| `recipe_group_id` | FK | レシピグループ |
| `recipe_id` | VARCHAR | 個別レシピID |

#### `recipe_duration_master` — レシピ想定処理時間（R4用）

| カラム | 型 | 説明 |
|--------|-----|------|
| `recipe_id` | PK | レシピID |
| `equipment_id` | FK | 装置（同レシピでも装置により異なる場合） |
| `expected_duration_sec` | INT | 想定処理時間（秒） |
| `updated_at` | TIMESTAMP | 更新日時 |

#### `port_conflict_rule` — ポート競合待ち対象装置（R3用）

| カラム | 型 | 説明 |
|--------|-----|------|
| `equipment_id` | PK | 対象装置 |
| `enabled` | BOOL | 有効/無効 |
| `updated_at` | TIMESTAMP | 更新日時 |

### 4.2 状態テーブル

#### `equipment_recipe_group_state` — 前回完了状態

| カラム | 型 | 説明 |
|--------|-----|------|
| `equipment_id` | FK | 装置 |
| `port_id` | VARCHAR / NULL | ポート（`scope=EQUIPMENT` の場合 NULL） |
| `recipe_group_id` | FK | レシピグループ |
| `last_recipe_id` | VARCHAR | 前回レシピID |
| `last_port_ids` | VARCHAR[] | 前回使用ポート一覧 |
| `last_complete_at` | TIMESTAMP | **前回完了時刻**（タイマー起点） |
| `last_card_no` | VARCHAR | 前回カードNo |
| `updated_at` | TIMESTAMP | 更新日時 |

#### `equipment_port_state` — 装置ポート処理状態（R3用）

| カラム | 型 | 説明 |
|--------|-----|------|
| `equipment_id` | FK | 装置 |
| `port_id` | VARCHAR | ポートID |
| `status` | ENUM | `IDLE` / `PROCESSING` |
| `processing_card_no` | VARCHAR / NULL | 処理中カードNo |
| `processing_recipe_id` | VARCHAR / NULL | 処理中レシピID |
| `start_at` | TIMESTAMP / NULL | 処理開始時刻 |
| `updated_at` | TIMESTAMP | 更新日時 |

### 4.3 ログテーブル

#### `start_judgement_log` — 判定履歴

| カラム | 型 | 説明 |
|--------|-----|------|
| `log_id` | PK | ログID |
| `equipment_id` | FK | 装置 |
| `port_id` | VARCHAR / NULL | 判定ポート |
| `card_no` | VARCHAR | カードNo |
| `recipe_id` | VARCHAR | 今回レシピ |
| `recipe_group_id` | VARCHAR / NULL | 対象グループ（非対象なら NULL） |
| `prev_recipe_id` | VARCHAR / NULL | 前回レシピ |
| `prev_port_ids` | VARCHAR[] / NULL | 前回使用ポート |
| `judgement` | ENUM | `ALLOW` / `REJECT` / `WAIT` / `SKIP` |
| `elapsed_sec` | INT / NULL | 前回完了からの経過秒数 |
| `remaining_sec` | INT / NULL | 残り許容秒数 |
| `recipe_duration_sec` | INT / NULL | レシピ想定処理時間 |
| `threshold_sec` | INT / NULL | 閾値 |
| `reason_code` | VARCHAR / NULL | 理由コード |
| `created_at` | TIMESTAMP | 判定日時 |

---

## 5. 処理フロー（v3）

### 5.1 予約フェーズ

```
1. App が予約要求を受信（装置 + カードNo）
2. [参考チェック]
   a. App は該装置の全有効ルールに対して
      last_complete_at からの経過時間を確認
      → 残時間が少ない場合 → 警告ログ出力（予約は通す）
   b. ※今回レシピはMESが持っているが予約IFに含まれないため
      正確な残時間チェック(R4)は不可
3. App → MES へ予約登録（装置 + カードNo）
```

### 5.2 着工フェーズ

```
4. MES → App へ着工指示
     (装置, カードNo, 今回レシピID, 今回使用ポート[],
      前回レシピID, 前回使用ポート[])
5. [突合] 着工指示の前回レシピ/ポートと App 保持値を突合
     → 不一致の場合は警告ログ（判定は App 保持値で実施）
6. [判定1: ポート競合チェック R3]
     → WAIT の場合、処理完了を待って再判定
7. [判定2: 時間制約チェック R1]
     → REJECT の場合、着工停止
8. [判定3: 残時間チェック R4]
     → REJECT の場合、着工停止
9. 全判定 ALLOW の場合:
     → 着工続行
     → equipment_port_state を PROCESSING に更新
10. 処理完了イベント受信時:
     → last_complete_at = 完了時刻 で状態更新
     → equipment_port_state を IDLE に更新
     → WAIT 中の着工があれば再判定をトリガー
```

### 5.3 完了フェーズ（タイマー更新）

```
11. 処理完了イベント受信（装置, カードNo, レシピID, 完了時刻）
12. レシピ → グループを引き当て
13. 対象グループの場合:
      → last_complete_at = 完了時刻
      → last_recipe_id, last_port_ids を更新
14. equipment_port_state を IDLE に更新
15. WAIT キューに該装置の保留着工があれば再判定を実行
```

---

## 6. 例外・運用考慮

| 項目 | 対応方針 |
|------|---------|
| **時刻同期** | App・MES 双方が完了時刻を保持。App 側の値を正とする（NTP 同期前提） |
| **前回情報の突合** | 着工指示の前回レシピ/ポートと App 保持値を突合し、不一致時は警告ログ |
| **同時着工排他** | 状態テーブルの更新は装置単位で排他制御 |
| **App再起動** | 状態テーブルは永続化済み。WAIT キューはインメモリの場合、起動時に再構築 |
| **一時解除** | ルールの `enabled` を OFF にする運用スイッチ |
| **ポートメンテナンス** | ポート停止時に `equipment_port_state` を IDLE に強制更新。タイマーは維持（安全側） |
| **WAIT タイムアウト** | ポート競合 WAIT が一定時間経過しても解消しない場合、タイムアウト REJECT |
| **異常完了** | 処理が異常終了した場合も完了時刻を記録するか、リセットするかは運用取り決め |

---

## 7. 仕様差分まとめ（v2→v3）

| 項目 | v2 | v3（今回） |
|------|-----|-----------|
| 着工指示の情報 | 今回レシピ + ポートのみ | **前回レシピID・前回使用ポートも含む** |
| MESの今回レシピID | 着工指示のみ | **MES自体が保持**（予約IFには含まれない） |
| タイマー起点 | 着工時刻 | **完了時刻**（App・MES双方が保持） |
| ポート競合制約（R3） | なし | **処理中ポート ≠ 今回ポートで WAIT**（装置単位で ON/OFF） |
| 残時間チェック（R4） | なし | **残時間 < レシピ処理時間で REJECT** |
| データ突合 | なし | 着工指示の前回情報と App 保持値の突合チェック |
| 完了フェーズ | なし | 完了イベントで `last_complete_at` 更新 + WAIT 解除 |
| テーブル追加 | − | `recipe_duration_master`, `port_conflict_rule`, `equipment_port_state` |
| 理由コード追加 | − | `PORT_CONFLICT_WAIT`, `INSUFFICIENT_REMAINING_TIME` |
| MES IF変更 | 不要 | **不要**（既存着工指示IFで対応可能） |

---

## 8. 未確定事項（確認依頼）

1. **ポート競合 WAIT のタイムアウト**: 何秒でタイムアウト REJECT にするか
2. **異常完了時の扱い**: 完了時刻を記録するか、タイマーをリセットするか
3. **レシピ想定処理時間の精度**: マスタ値に対してマージン（例: +10%）を設けるか
4. **WAIT 中の優先度**: 複数の着工が WAIT している場合の処理順（FIFO / 優先度ベース）
5. **前回情報の突合不一致時**: 警告のみか、着工を停止するか

上記が確定すれば、実装仕様（コード設計・DB DDL・エラーコード一覧・シーケンス図）に進められます。