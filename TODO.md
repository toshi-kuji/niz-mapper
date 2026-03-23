# TODO: Niz Mapper

## 1. プロジェクトセットアップ

- [x] Vite+ (`vp` CLI) をグローバルインストール
- [x] Vite+ で Astro 6 プロジェクトを作成 (`vp create astro`, minimal テンプレート)
  - 手順詳細: `docs/vite-plus-astro-guide.md`
- [x] 依存関係インストール済み (pnpm)
- [x] `.gitignore` にNode.js/Astro用の除外設定
- [x] Gitリポジトリ初期化・初回コミット済み
- [ ] GitHub Pagesデプロイ用のAstro設定 (`astro.config.mjs` に `site`, `base` 設定)
- [ ] GitHub Actions ワークフロー作成 (`.github/workflows/deploy.yml`, `withastro/action` 使用)
- [ ] GitHubリモートリポジトリを作成し、push
- [ ] GitHub Pages の設定を有効化（Settings → Pages → GitHub Actions）
- [ ] 空のAstroサイトがGitHub Pagesにデプロイされることを確認

## 2. プロトコル解析・データ準備

### cho45/niz-tools-ruby の解析
- [ ] `niz.rb` を読み、NIZ通信プロトコルの全体像を把握
  - [ ] `open` メソッド: 初期化コマンドのバイト列を特定
  - [ ] `version` メソッド: バージョン取得コマンドとレスポンス形式を特定
  - [ ] `read_all` メソッド: キーマップ読み取りの流れを理解（キーID×レベルIDの組み合わせ）
  - [ ] `write_all` メソッド: キーマップ書き込みの流れを理解（全リセット→全書き込み）
  - [ ] 各コマンドのバイト列とレスポンスフォーマットをドキュメント化

### 静的データファイルの作成
- [ ] `public/data/hwcodes.json` を作成: cho45氏のHWCODEマッピングをJSON化
  - 各エントリに `name`, `category` (basic/modifier/media/function等), `usb` (USB HIDスキャンコード) を含める
- [ ] `public/data/niz-l84.json` を作成: Niz L84のキーボード定義
  - `device`: id, name, manufacturer, vendorId (`0x0483`), productId (`0x5710`), keyCount (84)
  - `layout.keys[]`: 各キーの `id`, `x`, `y`, `width`, `height`, `label`
  - Niz L84の物理レイアウトを正確に反映する（キーサイズ・位置をユニット単位で定義）

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
