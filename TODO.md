# TODO: Niz Mapper

## 1. プロジェクトセットアップ

- [x] Vite+ (`vp` CLI) をグローバルインストール
- [x] Vite+ で Astro 6 プロジェクトを作成 (`vp create astro`, minimal テンプレート)
  - 手順詳細: `docs/vite-plus-astro-guide.md`
- [x] 依存関係インストール済み (pnpm)
- [x] `.gitignore` にNode.js/Astro用の除外設定
- [x] Gitリポジトリ初期化・初回コミット済み
- [x] GitHub Pagesデプロイ用のAstro設定 (`astro.config.mjs` に `site`, `base` 設定)
- [x] GitHub Actions ワークフロー作成 (`.github/workflows/deploy.yml`, `withastro/action` 使用)
- [x] GitHubリモートリポジトリを作成し、push (public: GitHub Pages無料プラン要件)
- [x] GitHub Pages の設定を有効化（Settings → Pages → GitHub Actions）
- [x] 空のAstroサイトがGitHub Pagesにデプロイされることを確認

## 1.5. 法務・ライセンス確認

- [x] cho45/niz-tools-ruby のライセンスを確認（ライセンス未指定 → コード非コピー方針を明記）
- [x] README に Legal セクション作成（免責・相互運用性・商標・Acknowledgments・Takedown窓口）
- [x] MIT License を追加（LICENSE ファイル）

## 2. プロトコル解析・データ準備

### cho45/niz-tools-ruby の解析
- [x] `niz.rb` を読み、NIZ通信プロトコルの全体像を把握
  - [x] `open` メソッド: VID `0x0483` / PID `0x512A` で接続、version取得で初期化
  - [x] `version` メソッド: `0xf9` 送信 → バージョン文字列受信、先頭数字がキー数
  - [x] `read_all` メソッド: `0xf2` 送信 → キー数×3レイヤー分のデータをループ受信
  - [x] `write_all` メソッド: `0xf1` で全リセット → `0xf0` で1キーずつ送信 → `0xf6` で終了
  - [x] 各コマンドのバイト列とレスポンスフォーマットをドキュメント化 → `docs/niz-protocol-spec.md`

### 静的データファイルの作成
- [x] `public/data/hwcodes.json` を作成: cho45氏のHWCODEマッピングをJSON化
  - 各エントリに `name`, `category` (basic/modifier/function/navigation/numpad/media/mouse/backlight/setting/system) を含める
- [x] `public/data/niz-l84.json` を作成: Niz L84のキーボード定義
  - `device`: id, name, manufacturer, vendorId, productId, keyCount
  - `layout.keys[]`: 85キー分の `id`, `x`, `y`, `width`, `height`, `label`
  - **要実機確認**: 物理85キー vs モデル名L84 の差異（スプリットスペースバーが1 key_id?）
  - **要実機確認**: key_id（仮番号）と実際のファームウェアkey_idのマッピング

## 3. WebHID通信モジュールの実装

### 基盤となるヘルパー
- [ ] `src/lib/niz-hid.js` を作成
- [ ] `navigator.hid` のサポートチェック関数
- [ ] `connectDevice()`: Niz L84への接続（vendorId/productIdフィルタ付き `requestDevice`→`open`）
- [ ] `sendCommandAndWaitResponse(device, commandBytes)`: Feature Report送信＋inputreportイベント待機（3秒タイムアウト付きPromise）
- [ ] `logBytes(label, bytes)`: 送受信バイト列の16進数ログ出力

### NIZプロトコル操作
- [ ] `getVersion(device)`: バージョン情報取得コマンド送信・レスポンス解析
- [ ] `readKeymap(device, onProgress)`: 全キーマップ読み取り（198回ループ、プログレスコールバック付き）
- [ ] `writeKeymap(device, keymap, onProgress)`: 全キーマップ書き込み（全リセット→全書き込み、プログレスコールバック付き）

### 接続テスト
- [ ] Niz L84 の実際の Product ID を実機で確認（cho45氏のコードでは `0x512A`、開発ガイドでは `0x5710`）
- [ ] 実機で `read_all` し、ファームウェアのキー数を確認（物理85キー vs モデル名L84 の差異を解消。スプリットスペースバーが2つの独立した key_id を持つか確認）
- [ ] ファームウェアの key_id と物理キー位置のマッピングを確定（`niz-l84.json` の仮番号を実際の key_id に更新）
- [ ] 最小限のテストページで Niz L84 に接続できることを確認
- [ ] バージョン情報の取得が成功することを確認
- [ ] キーマップの読み取りが成功し、コンソールでデータを確認

## 4. キーボードレイアウトUIの実装

### レイアウト表示
- [ ] `src/pages/index.astro` にメインページを作成
- [ ] 「キーボードに接続」ボタンと接続ステータス表示
- [ ] `niz-l84.json` からレイアウト定義を読み込み、キーボードを描画
  - 各キーを `<div>` で absolute positioning（x, y, width, height をユニット→px変換）
  - キーラベル表示
- [ ] キーのCSS: キーキャップ風の見た目（グラデーション、ボーダー、影）
- [ ] キーホバー時のアニメーション（浮き上がり効果）
- [ ] キークリックで選択状態（色変更でハイライト）

### キーマップ表示
- [ ] 接続後、キーマップを自動読み取り（プログレスバー付き）
- [ ] 読み取ったキーマップを各キーに反映表示（HWCODEから名前に変換）
- [ ] 選択中のキーの現在の割り当てを詳細表示

## 5. キーリマップ機能の実装

### キー機能選択UI
- [ ] キー選択時に表示される機能選択パネルを作成
- [ ] HWCODEをカテゴリ別にタブ表示（基本キー / 修飾キー / ファンクションキー / メディアキー 等）
- [ ] 各カテゴリ内でキーをクリックして割り当てを変更

### キーマップの編集・書き込み
- [ ] メモリ上のキーマップデータを変更（UI操作に連動）
- [ ] 変更があったキーの視覚的なマーキング（未保存の変更を示す）
- [ ] 「書き込み」ボタン: 全キーマップをキーボードに書き込み
  - 書き込み中のプログレスバー表示
  - 「USBケーブルを抜かないでください」の警告表示
  - 書き込み完了/失敗のフィードバック
- [ ] 「リセット」ボタン: 変更を破棄してキーボードから再読み取り

## 6. エラーハンドリングとUX

- [ ] WebHID非対応ブラウザの検出と案内メッセージ（Chrome/Edge/Operaを推奨）
- [ ] デバイス未選択時のハンドリング
- [ ] 接続中のUSBケーブル切断検知と再接続案内
- [ ] 通信タイムアウト時のエラーメッセージ（「USBケーブルを確認して再接続してください」）
- [ ] 書き込み失敗時のリカバリー案内
- [ ] 全非同期操作のtry-catchラップ

## 7. 最終検証・デプロイ

- [ ] Niz L84 実機でのE2Eテスト
  - [ ] 接続→バージョン取得
  - [ ] キーマップ全読み取り
  - [ ] 1キー変更→全書き込み→再読み取りで変更が反映されていることを確認
  - [ ] 書き込み中のケーブル抜き差し等の異常系テスト
- [ ] Chrome / Edge での動作確認
- [ ] Firefox / Safari で非対応メッセージが表示されることを確認
- [ ] GitHub Pagesでの本番デプロイ確認（HTTPSでWebHIDが動作すること）
- [ ] README.md の作成（使い方、対応ブラウザ、注意事項）
