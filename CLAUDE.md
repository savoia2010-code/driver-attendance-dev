# driver-attendance-dev プロジェクト概要

配送ドライバー向けの出退勤管理ブラウザアプリ。
GitHub Pages でホスティング、Google Apps Script + Googleスプレッドシートをバックエンドとして使用。

## ファイル構成

- `index.html` — アプリ本体（単一HTMLファイル）
- `コード.js` — Google Apps Script（GASバックエンド）
- `.clasp.json` — clasp設定（GASスクリプトIDを含む）
- `appsscript.json` — GASマニフェスト

## 接続情報

- **GitHub Pages URL**: https://savoia2010-code.github.io/driver-attendance-dev/
- **GitHubリポジトリ**: https://github.com/savoia2010-code/driver-attendance-dev.git
- **Googleスプレッドシート**: https://docs.google.com/spreadsheets/d/1-8LeSGnv7jIafBlOy6iX_u_2j_PoAhfeTPruWRBoEHA/edit
- **GAS URL**: ※デプロイ後に `index.html` 内の `GAS_URL` 定数に記載

## デプロイ手順

### index.html を更新する場合
```bash
git add index.html
git commit -m "変更内容"
git push
```

### GAS（コード.js）を更新する場合
```bash
clasp push
# その後、GASエディタで既存デプロイを新バージョンで更新
# （clasp deployは新デプロイを作るため使わない）
```

## 技術仕様

### フロントエンド（index.html）
- 単一HTMLファイル、ダーク/インダストリアルテーマ
- iPhone Safari最適化（font-size:16px、カスタム確認モーダル）
- localStorage（当日状態・PIN）+ GAS（永続化）
- `_store` セーフラッパー（localStorage不可時はメモリ動作）
- `DOMContentLoaded` でinit()実行、`catch(e){}` 必須（iOS Safari対応）
- gasPost: `Content-Type: text/plain` でPOST送信

### カラー設定
```css
--bg: #0f0f0f; --surface: #1a1a1a; --surface2: #242424; --border: #2e2e2e;
--accent: #fb923c; --danger: #ef4444; --success: #22c55e; --info: #3b82f6;
--text: #f0f0f0; --text2: #b8b8b8; --text3: #909090;
```

### ストレージキー
```javascript
STORAGE_KEY     = 'driver_attendance_v2'
DRIVERS_KEY     = 'driver_list_v1'
CUR_DRIVER_KEY  = 'current_driver_v1'
CHECKERS_KEY    = 'checker_list_v1'
```

### todayState 構造
```javascript
{
  start, end, startDate,           // 打刻時刻・始業日（宿泊判定用）
  startMileage, endMileage,
  breaks: [{start, end, startMileage, place}],
  type: 'normal'|'holiday',
  note,                            // GPS現在地
  destination, kumitate, bara, onetouch, kobutsu, other,
  lodging, allowance,
  alcBDate, alcBMethod, alcBRemote, alcBResult, alcBChecker,  // 運転前酒気
  alcADate, alcAMethod, alcARemote, alcAResult, alcAChecker,  // 運転後酒気
  visits: [{place, report}],
  fuelEntries: [{shop, fuel, fuelRegular, adblue}],
  report,
}
```

### 宿泊フロー
- **休憩開始時** に日付またぎを検出（`_realTodayKey() !== todayState.startDate`）
- 確認ダイアログ → 運転後酒気確認（1日目）→ Day1保存
- `_waitingLodgingDay2 = true` をセット、翌日モードへ
- **休憩終了時** に `_waitingLodgingDay2` が true なら運転前酒気確認（2日目）
- 日報タブは2日目モード（配送情報・積込・出張報告書を非表示、alcBのみ表示）

### テスト用機能
- 打刻タブの `🔧 始業日` バッジを**1.5秒長押し**で始業日を手動変更可能

### GAS スプレッドシート構成
- シート名: `ドライバー名_年度月`（例: `田中_2026年度3月`）
- 締め日: 26日以降は翌月シートに書き込み
- 行1: タイトル、行2: ヘッダー、行3〜33: データ（前月26日〜当月25日）
- 行34: 繰り返しヘッダー、行35: 合計行
- 土日祝: オレンジ色、宿泊行: 青色（優先）
- 総列数: 118列（`TOTAL_COLS = 118`）

### COL定数（主要列）
```
A(1):date  B(2):destination  H(8):lodging  I(9):allowance
L(12):workTime  M(13):nightTime  N(14):overtime
P(16):startMeter  R(18):endMeter
U(21)〜AD(30): 酒気帯び確認（前・後）
AE(31)〜AH(34): 給油情報
AI(35):departTime  AJ(36):returnTime
AO(41)〜: 休憩データ（6件、各3列）
AV(59)〜: 訪問先・報告事項（30件、各2列）
```

## 注意事項

- 複数行JSの編集はPython `str.replace()` via bash heredocが確実
- 構文チェック: `<script>` ブロックをPythonで抽出 → `node --check /tmp/test.js`
- `window.confirm()` は使用不可（iPhone Safari対応のためカスタムモーダルを使用）
- `catch(e){}` を省略しない（古いiOS対応）
- テンプレートリテラルや日本語文字列の `str_replace` は失敗しやすい
