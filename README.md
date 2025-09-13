いいね、それやろう。
新しいチャットでも迷子にならずに続けられるように、**誰でも読めば作れる＆運用できる**超ていねい版の README をまとめたよ。これを **README.md** として貼り付ければOK。以降の会話は、このドキュメントを“仕様と手順の真実”として進められるはず。

---

# ICP×GMax 競馬予測AI — 完全README（誰でも作れる＆運用できる版）

## 0. これは何？

* 本システムは、**馬の地力（BasePower）× 当日の出力率（OutputRate）** で、その日のパフォーマンスを数値化し、**勝率・損益分岐オッズ・期待値**まで出す予測エンジンです。
* データは **SARAD**（`JVData.db` のSQLite）から取得します。
* **出力は“オリジナルの表形式”**（TARGET互換カラム順）で、既存ツールにそのまま渡せます。

## 1. 出力フォーマット（オリジナルの表）

CSV（UTF-8, BOMなし）

| カラム名       | 意味                           |
| ---------- | ---------------------------- |
| `date`     | 開催日（例 `2025-09-13 00:00:00`） |
| `course`   | 競馬場コード                       |
| `race_no`  | レース番号（整数）                    |
| `umaban`   | 馬番                           |
| `umaname`  | 馬名                           |
| `ICP`      | 最終スコア（公開値、各レース内で最大100、相対）    |
| `ICP_rank` | スコア順位（レース内）                  |
| `BE_odds`  | 損益分岐オッズ = 1 / 勝率             |
| `EV`       | 期待値 = 勝率 × 単勝オッズ（オッズ不明なら空）   |
| `p_win`    | 勝率（レース内でソフトマックス変換）           |
| `odds`     | 単勝オッズ（あればマージ）                |

> 例：`D:\IC-App\runs\sample\board.csv` に書き出し

---

## 2. フォルダ構成（推奨：新規アプリ名 **IC-App2**）

```
D:\
  IC-App2\
    scripts\
      icp_gspec_predict.py   ← 仕様準拠の計算（学習不要）
    runs\
      sample\                ← 1日分の出力置き場
      _logs\                 ← 実行ログ（任意）
    models\                  ← （将来：学習モデルを置く）
```

> 既存の IC-App と混ざらないように **IC-App2** を推奨。

---

## 3. 必要ソフト

* **Windows 10/11（64bit）**
* **Python 3.11（64bit）**
* Python パッケージ

  * `pandas`, `numpy`
  * （学習モデルを使う場合のみ `scikit-learn`, `joblib`）

インストール例（PowerShell）:

```powershell
pip install --upgrade pip
pip install pandas numpy
# 将来ML機能を使うなら:
# pip install scikit-learn joblib
```

**日本語文字化け対策**（毎回でなく初回に実施しておくと楽）:

```powershell
$OutputEncoding = [Console]::OutputEncoding = [Text.UTF8Encoding]::new($true)
$env:PYTHONUTF8 = "1"
$env:PYTHONIOENCODING = "utf-8"
chcp 65001 | Out-Null
```

---

## 4. 使うテーブル（SARAD / JVData.db）

最低限この2つがあれば動きます（列名は日本語のまま）：

* **馬毎レース情報** … エントリー（開催年月日, レース番号, 競馬場コード, 馬番, 馬名, 距離, 芝ダ障, 斤量, 走破タイム, 着差, 通過順, 脚質）
* **オッズ１** … 単勝オッズ（あれば勝手にマージ。無ければ空欄）

**速度改善のための任意インデックス（できれば推奨）**
（Pythonからも `con.execute()` で作れます）

```sql
CREATE INDEX IF NOT EXISTS idx_s_date ON "馬毎レース情報"("開催年月日");
CREATE INDEX IF NOT EXISTS idx_od1_date ON "オッズ１"("開催年月日");
```

---

## 5. 計算ロジック（仕様に完全準拠）

### 5.1 基本式

* **G-Horse = 100**（どの条件でも常に 100pt の仮想最強馬）
* **ICPスコア（当日）**

  ```
  Base_Performance = (レースレベル × ラップ補正) + 着差補正 + 時計補正
  地力スコア = ((Base_Performance + 斤量補正_地力) × 馬場バイアス倍率 × ペース倍率)
                ÷ G-Horse × 100
  ICP = 地力スコア × 斤量補正_出力倍率
  （最後にレース内で最大100に規格化して公開）
  ```

### 5.2 レースレベル（Race Level）

* 当日レースの出走馬について、**過去レースでの「能力Proxy」**（時計補正＋着差補正）を馬ごとに取り、**上位3頭の平均**を当日のレースレベルとする（仕様「出走馬の地力スコア平均（または上位3頭平均）」に準拠）。
* **能力Proxy**の作り方（暫定）

  * 過去走の **条件平均タイム**（同一：競馬場コード×芝/ダ×距離）との差分を“時計補正”（速いほど＋）へ換算
  * **着差補正（秒）**： 0.0秒差→+8、0.3秒差→+6、0.5秒以上→+2（間は線形）
  * その和の **最大値** を「その馬の最大能力Proxy」として採用

> 注）本設計では **「絶対的基準馬（G-Horse=100）」は“公開スケールの物差し”**。
> レース間の強弱は **レースレベル**で吸収。強いメンバーが揃えばレベルが高く、弱ければ低くなる。

### 5.3 ラップ補正

* first / middle / last の “得意型一致”なら **+0.05〜+0.08** を加点
* 初期実装では中立（`+0.06`）で稼働。ラップ型が用意でき次第、馬別に一致/不一致で自動加点。

### 5.4 着差補正（当日）

* レース前なので当日は 0（未知）。**過去レースからレベル推定にのみ使用**。

### 5.5 時計補正（当日）

* レース前なので 0（未知）。**過去レースからレベル推定にのみ使用**。

### 5.6 斤量補正（NEW：仕様通りに2系統）

* **地力補正（pt）**： `(当日斤量 − 基準斤量) × 1.0 pt/kg`

  * 基準斤量はその馬の過去平均斤量（当日斤量が不明な場合は=基準→補正0）
* **出力率補正（倍率）**：差分 **0.5kgあたり±1%**（軽い→上方、重い→下方）

### 5.7 馬場バイアス補正（倍率）

* `inner`（内有利）/`outer`（外有利）/`flat`（中立）
* 強度：`weak`=0.03、`mid`=0.05、`strong`=0.08

  * 例：内有利で「外」走なら +S、内で“好走”なら −S
  * 当面は位置/好走は中立で運用（将来UI入力で指定可能）

### 5.8 展開（ペース）補正（倍率）

* `slow`（遅）/`mid`（平均）/`fast`（速）

  * 遅い：先行有利（先行＋0.07 / 差し−0.07）
  * 速い：差し有利（先行−0.07 / 差し＋0.07）
  * 中立：補正なし
* 脚質は直近レースから推定（無ければ“差し”）

### 5.9 公開スケール（必ず100以下）

* 同一レース内で `max(ICP) = 100` となるように正規化（相対比較しやすくするため）。

### 5.10 勝率・損益分岐・期待値

* **勝率**：レース内の ICP をソフトマックス変換
  `p_i = exp(ICP_i) / Σ_j exp(ICP_j)`
* **損益分岐オッズ**： `1 / p_i`
* **期待値（EV）**： `p_i × 単勝オッズ`（オッズがなければ空）

---

## 6. 実行方法（超かんたん）

### 6.1 スクリプトを置く

`D:\IC-App2\scripts\icp_gspec_predict.py` に保存。

### 6.2 1日分を出力（9/13の例）

```powershell
cd D:\IC-App2\scripts

python icp_gspec_predict.py `
  --db D:\SaraD\JVData.db `
  --date 2025-09-13 `
  --out D:\IC-App2\runs\sample\board.csv
```

> 馬場・ペース中立（`--bias flat --pace mid`）で計算。
> 出力は **オリジナル表** で `runs\sample\board.csv` に保存。

### 6.3 馬場＆ペースを指定

```powershell
python icp_gspec_predict.py `
  --db D:\SaraD\JVData.db `
  --date 2025-09-13 `
  --bias inner --bias-strength strong `
  --pace slow `
  --out D:\IC-App2\runs\sample\board.csv
```

---

## 7. ダブルクリックで実行（バッチ同梱）

### 7.1 `RUN_Predict_Today.bat`（今日の分を一発出力）

```bat
@echo off
setlocal
set BASE=D:\IC-App2
set PY=%LocalAppData%\Programs\Python\Python311\python.exe
set DB=D:\SaraD\JVData.db

for /f %%i in ('powershell -NoProfile -Command "Get-Date -Format yyyy-MM-dd"') do set DATE=%%i

"%PY%" -u "%BASE%\scripts\icp_gspec_predict.py" ^
  --db "%DB%" ^
  --date %DATE% ^
  --out "%BASE%\runs\sample\board.csv"

if errorlevel 1 (echo [NG] failed& pause) else (echo [OK] wrote "%BASE%\runs\sample\board.csv"& pause)
```

### 7.2 `RUN_Predict_Date.bat`（指定日を出力）

```bat
@echo off
setlocal
if "%~1"=="" (
  echo Usage: RUN_Predict_Date.bat YYYY-MM-DD
  pause
  exit /b 1
)
set BASE=D:\IC-App2
set PY=%LocalAppData%\Programs\Python\Python311\python.exe
set DB=D:\SaraD\JVData.db

"%PY%" -u "%BASE%\scripts\icp_gspec_predict.py" ^
  --db "%DB%" ^
  --date %1 ^
  --out "%BASE%\runs\sample\board.csv"

if errorlevel 1 (echo [NG] failed& pause) else (echo [OK] wrote "%BASE%\runs\sample\board.csv"& pause)
```

> どちらも `D:\IC-App2` に保存してダブルクリックで動きます。
> **管理者権限は不要**。黒画面が出て結果と `[OK]` が見えます。

---

## 8. よくあるエラーと対処

* **“ファイルまたはディレクトリが存在しません”**
  → フォルダが無い（`runs\sample` など）。先に作成してください。
* **日本語が “????” になる / “no such table: ???????”**
  → 文字コード問題。PowerShellで UTF-8 化（上の「日本語文字化け対策」）を実施。
* **`can't open file ... .py`**
  → 実行場所が違う or パスが違う。`cd` で `scripts` に移動してから実行。
* **遅い**

  * まずインデックス作成（上記SQL）
  * 同一PCで大量日付を回す場合は、1レース単位での処理へ分割などを検討
* **出力が0行**

  * 該当日のデータが `JVData.db` に無い。SARAD側で取得済みか確認

---

## 9. 仕組みの復習（短縮版）

1. **当日出走馬**を `馬毎レース情報`（＋あれば `オッズ１`）から取得
2. **過去走**を cutoff日より前で取得し、条件平均タイムを作成
3. **能力Proxy**（時計補正＋着差補正）の **最大値**を馬ごとに推定
4. 当日の **レースレベル** = 出走馬の能力Proxy **上位3頭平均**
5. 当日 **Base\_Performance** を組み立て、**斤量/馬場/ペース**で補正
6. **G-Horse=100** を基準にスケール → さらにレース内で最大100に正規化
7. **勝率（ソフトマックス）** → **損益分岐オッズ** → **期待値**
8. **オリジナル表**で書き出し

---

## 10. 将来拡張（このREADMEで引き継ぎOK）

* **ラップ補正を実データ化**

  * first/middle/last の実測ラップ×馬ごとの“得意型”を判定して加点
* **当日距離・コース・斤量の厳密取得**

  * いまは過去平均/最多などで近似 → 番組表・成績テーブルから当日値を参照
* **走行位置（内/外）・好走フラグ**

  * UIや実況/走破データから判定し、馬場バイアス補正を活性化
* **学習モデル（Ridge/NN）併用**

  * BasePowerの推定やラップ適応の係数最適化を ML で上書き
  * ただし **公開値の定義（G-Horse=100・レース内最大100・ソフトマックス）** は **固定**

---

## 11. いざという時の最小チェック

* `python -V` → 3.11.x か？
* `pip show pandas` → 入ってるか？
* `sqlite3 D:\SaraD\JVData.db`（任意）→ テーブル名に日本語が見えるか？
* `python icp_gspec_predict.py --help` → 使い方が出るか？

---

## 12. 連絡事項（運用ルール）

* **TARGETへは自動で書き込みません**。常に **CSV（オリジナル表）** で出力 → 好きなツールに読み込む。
* 馬場/ペースは最初は **`--bias flat --pace mid`** でOK。慣れたら状況に合わせて指定。

---

## 付録：スクリプト本体（再掲）

> `D:\IC-App2\scripts\icp_gspec_predict.py`
> 依存：`pandas`, `numpy` のみ

（ここに現行の **icp\_gspec\_predict.py** をそのまま貼り付け）

---

この README を新しいチャットに読み込ませれば、**この手順と仕様で**そのまま作業を引き継げるはず。
追加で、バッチやUI、ラップ実装を足していくときも、このREADMEを“正”にして進めればブレません。
