# CountLens（iPhone版） — 写真から物体を検出して数量を計測するアプリ

写真を撮影／読み込むと、AIが対象物を自動検出。数えたい対象を選んで計測し、
推定単価・金額、撮影場所(GPS)・メモの記録、結果のExcel(.xlsx)出力ができます。

## ファイル構成（フォルダごとアップロードしてください）

| ファイル | 内容 |
|---|---|
| `index.html` | 入口（自動で iPhone版アプリへ移動） |
| `CountLens-iPhone.dc.html` | iPhoneアプリ本体 |
| `ios-frame.jsx` | iPhoneデバイスフレーム部品 |
| `support.js` | 実行ランタイム（削除・編集しないでください） |
| `.nojekyll` | GitHub Pages の Jekyll 処理を無効化（**必須**。無いと表示が崩れます） |

## GitHub Pages での公開

1. リポジトリ **Settings ▸ Pages** ▸ Source を **Deploy from a branch** / **main / (root)** にして保存
2. 数十秒後、`https://<ユーザー名>.github.io/<リポジトリ名>/` で公開

## 主な機能

- 📷 写真を撮影 / 🖼 ライブラリから選択
- 🟩 AIによる物体検出（対象物に外接する長方形フレーム＋番号バッジ）
- ✅ チェックで計測対象を選択・✏️ 手動で追加/削除
- 🔢 カウント閾値（最小数量未満を除外）・💴 推定単価/金額/総額
- 📍 撮影場所のGPS自動取得・メモ
- 🗂 計測リスト／対象物別集計／📊 Excel(.xlsx)出力（2シート）
- 📂 保存済みExcelを開いて追記 → 同名で保存

## AI検出について（重要）

公開ページでAI検出を使うには、アプリ内の「**🔑 AI検出のセットアップ**」から
**ご自身の Anthropic APIキー** を入力してください（この端末のブラウザ内にのみ保存、
画像の送信先は Anthropic API のみ）。APIキーは https://console.anthropic.com/settings/keys で発行できます。
キー未設定でも、写真表示・手動カウント・Excel出力は利用できます。

## 動作要件

- カメラは **https**（GitHub Pages）または **localhost** で利用できます。
- Excelのセル装飾は `xlsx-js-style`、GPS読み取りは `exifr` をCDNから読み込みます（オンライン環境が必要）。
