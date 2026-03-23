# Vite+ で Astro 6 プロジェクトをセットアップする手順

Astro は Vite 上に構築されているため、Vite+（`vp` CLI）と完全に互換性があります。Vite+ を使うことで、Node バージョン管理から高速ビルド（Rolldown ベースで最大7倍高速）まで、統一されたツールチェーンで Astro プロジェクトを管理できます。

## 前提条件

- Node.js 22.12.0 以上
- pnpm（Vite+ が自動検出）

## 1. Vite+ のインストール

```bash
# Vite+ CLI をグローバルにインストール
pnpm add -g vite-plus
```

インストール後、`vp` コマンドが使えることを確認:

```bash
vp --version
```

## 2. Astro プロジェクトの作成

```bash
# Vite+ でプロジェクトをスキャフォールド
vp create niz-mapper

# プロジェクトディレクトリに移動
cd niz-mapper
```

プロジェクト作成時にフレームワーク選択で **Astro** を選択します。

既存の Astro プロジェクトがある場合は `vp migrate` で変換できます:

```bash
vp migrate
```

## 3. 依存関係のインストール

```bash
# vp install は pnpm を自動検出し、正しい Node バージョンを使用
vp install
```

`npm install` や `pnpm install` の代わりに `vp install` を使うことで、Node バージョンの整合性が保証されます。

## 4. astro.config.mjs の設定

プロジェクトルートの `astro.config.mjs` を編集します:

```javascript
import { defineConfig } from 'astro/config';

export default defineConfig({
  // GitHub Pages 用の設定
  site: 'https://<username>.github.io',
  base: '/niz-mapper',
});
```

### Vite プラグインの追加（必要に応じて）

Astro は内部で Vite を使っているため、Vite プラグインを直接追加できます:

```javascript
import { defineConfig } from 'astro/config';
import myVitePlugin from 'my-vite-plugin';

export default defineConfig({
  site: 'https://<username>.github.io',
  base: '/niz-mapper',
  vite: {
    plugins: [myVitePlugin()],
  },
});
```

## 5. 開発サーバーの起動

```bash
# Vite+ の最適化された環境で Astro dev サーバーを起動
vp dev
```

ブラウザで `http://localhost:4321` にアクセスして動作確認します。

## 6. プロダクションビルド

```bash
# Rolldown ベースの高速ビルド
vp build
```

ビルド成果物は `dist/` ディレクトリに出力されます。

## 7. コードのチェック

```bash
# Oxlint（リント）+ Oxfmt（フォーマット）を Rust 速度で実行
vp check
```

従来の ESLint と比べて 50〜100 倍高速です。

## コマンド対応表

| 従来のコマンド | Vite+ コマンド | 説明 |
|---|---|---|
| `pnpm create astro@latest` | `vp create` | プロジェクト作成 |
| `pnpm install` | `vp install` | 依存関係インストール |
| `pnpm run dev` | `vp dev` | 開発サーバー起動 |
| `pnpm run build` | `vp build` | プロダクションビルド |
| `eslint . && prettier --write .` | `vp check` | リント＋フォーマット |

## Astro 6 + Vite+ の主なメリット

- **高速ビルド**: Rolldown（Rust ベース）により、標準 Rollup 比で最大 7 倍高速
- **高速 Markdown/MDX**: Astro 6 が Vite Environment API 上にdev サーバーを再構築し、最大 5 倍高速化
- **統一ランタイム**: ローカル開発環境とプロダクション環境（GitHub Pages 等）の一致を保証
- **Rust 駆動ツール**: リント (Oxlint)、フォーマット (Oxfmt) が従来比 50〜100 倍高速
