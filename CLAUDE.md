# NIZ Keyboard Web Configurator

## Project Overview
NIZキーボードのキーマップをブラウザから変更できるWebアプリ。VIA/NuPhyライクなUIを目指す。
- 対応キーボード: **Niz L84 のみ** (Vendor ID: `0x0483`, Product ID: `0x5710`)
- 参考実装: [cho45/niz-tools-ruby](https://github.com/cho45/niz-tools-ruby) (Ruby版のNIZツール)
- 詳細ガイド: `docs/niz-webapp-development-guide.md`

## Tech Stack
- **Frontend**: Astro (静的サイト生成) + Vanilla JS (WebHID通信)
- **Hosting**: GitHub Pages (HTTPS自動提供 → WebHID要件を満たす)
- **キーボード定義**: 静的JSONファイルとしてリポジトリに含める（バックエンド不要）

## Key Technical Details

### WebHID API
- Chromium系ブラウザ専用 (Chrome, Edge, Opera)。Safari/Firefox非対応
- HTTPS必須（localhost は例外）
- ユーザージェスチャー（ボタンクリック等）が必要。自動接続は不可
- NIZキーボードは Feature Report を使用
- 通信パターン: `device.sendFeatureReport()` → `inputreport` イベントでレスポンス受信

### NIZ Protocol (cho45氏の解析に基づく)
- **HWCODE**: NIZ独自のキーコード体系（USB HID Scancodeとは別番号）。cho45氏が解析済み
- キーマップ書き換えコマンド送信時、**既存キーマップが全リセットされる** → 必ず「全読み取り → 変更 → 全書き込み」の流れが必要
- 全キー読み書きは198回の操作が必要（数秒かかる）→ プログレス表示が必須

## References
- WebHID API: https://developer.mozilla.org/en-US/docs/Web/API/WebHID_API
- cho45's repo: https://github.com/cho45/niz-tools-ruby
- VIA (reference UI): https://github.com/the-via/app
