# CountLens（iPhone版）

写真から物体を検出して数量を計測するWebアプリ。**`index.html` 1ファイルで完結**します（依存ファイル不要）。

## 公開手順（重要・この通りにしてください）

1. リポジトリ内の**既存ファイルはすべて削除**してください（古い・階層がずれたファイルが残っていると表示が崩れます）。
2. このフォルダの **`index.html`** と **`.nojekyll`** を**リポジトリのルート**にアップロード。
   - `.nojekyll` は隠しファイルでドラッグでは入らないことがあります。その場合は GitHub 上で **「Add file ▸ Create new file」→ ファイル名に `.nojekyll` と入力 → そのまま Commit**（中身は空でOK）。
3. **Settings ▸ Pages** ▸ Source = **Deploy from a branch** / **main / (root)** で保存。
4. 数十秒後、`https://<ユーザー名>.github.io/<リポジトリ名>/` を開く。

## AI検出について

公開ページでAI検出を使うには、アプリ内 **「🔑 AI検出のセットアップ」** から
ご自身の **Anthropic APIキー** を入力してください（端末のブラウザ内のみ保存、送信先は Anthropic API のみ）。
キーは https://console.anthropic.com/settings/keys で発行。未設定でも写真表示・手動カウント・Excel出力は使えます。

## 注意

- カメラは https（GitHub Pages）または localhost で利用できます。
- Excel装飾(`xlsx-js-style`)・GPS読取(`exifr`)はCDNを読み込むためオンライン環境が必要です。
