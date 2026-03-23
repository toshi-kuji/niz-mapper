# NIZキーボード Webアプリ開発ガイド

## 🎯 プロジェクト概要

Niz L84キーボードのキーマップをブラウザから直接変更できるWebアプリケーションを、Astro + Vanilla JavaScriptで構築します。VIAやNuphy IOのような使いやすいインターフェースを目指しながら、シンプルで保守しやすいコードベースを維持します。対応キーボードはNiz L84のみです。

ホスティングにはGitHub Pagesを使用します。GitHub Pagesは自動的にHTTPSを提供するため、WebHID APIの要件を満たします。キーボード定義は静的JSONファイルとしてリポジトリに含めるため、バックエンドサーバーは不要です。

## 🏗️ アーキテクチャの理解

このWebアプリケーションは、シンプルな静的サイトとして構成されます。

Astroで静的サイトを生成し、GitHub Pagesにデプロイします。ユーザーのブラウザ上では、WebHID APIを使ってNiz L84キーボードと直接USB通信を行います。キーマップの読み取りや書き込みといった、キーボードとの低レベルなやり取りは、全てブラウザ内で処理されます。ユーザーのキーボードデータは、ユーザーのブラウザ内だけで完結し、サーバーに送信されることはありません。

キーボード定義（レイアウト、各キーの位置とサイズ、キーIDの割り当てなど）やHWCODEマッピングは、静的JSONファイルとしてリポジトリに含めます。対応キーボードがNiz L84のみなので、バックエンドAPIやデータベースは不要です。Astroのビルド時にこれらの定義を読み込み、UIを生成します。

## 📋 段階的な開発アプローチ

このプロジェクトを成功させるためには、いきなり完全なシステムを作ろうとするのではなく、段階的に機能を追加していくアプローチが重要です。各段階で動作するプロトタイプを作り、それを改良していくことで、着実に完成に近づけます。

開発は二つの段階に分かれます。第一段階では、Astroプロジェクトをセットアップし、WebHID APIでNiz L84との通信を確立します。第二段階では、キーボードレイアウトUIとキーマップの読み書き機能を実装し、完成させます。

この段階的なアプローチには、いくつかの重要な利点があります。まず、早い段階で動作するものが手に入ります。完璧なシステムを目指して数ヶ月かけるよりも、一週間で動くプロトタイプを作る方が、学びも多く、モチベーションも維持しやすいでしょう。また、各段階で実際に動かしながら問題点を発見できるため、設計の誤りに早く気づけます。

それでは、各段階を詳しく見ていきましょう。

## 📋 開発の全体フロー

### 第一段階: 静的プロトタイプの構築

第一段階の目標は、WebHID APIの基本的な使い方を理解し、NIZキーボードと通信できる最小限のプロトタイプを作ることです。この段階では、サーバーもデータベースも使いません。キーボード定義は、シンプルなJSONファイルとしてプロジェクトフォルダに配置します。

まず、プロジェクトフォルダを作成します。必要なのはテキストエディタ（VS CodeやCursor推奨）と、Chromium系ブラウザ（ChromeまたはEdge）だけです。ビルドツールは不要なので、シンプルなフォルダ構成から始められます。

プロジェクトフォルダには、三つの基本ファイルを用意します。`index.html`がエントリーポイントで、`app.js`がメインのロジック、`style.css`がスタイリングを担当します。このシンプルな構成なら、後から機能を追加するのも簡単です。さらに、キーボード定義を格納する`definitions`フォルダを作り、その中に`niz-plum-84.json`のような定義ファイルを配置します。HWCODEマッピングは、全キーボード共通なので、`hwcodes.json`として別ファイルで管理します。

```
niz-webapp/
├── index.html
├── app.js
├── style.css
├── hwcodes.json
├── definitions/
│   └── niz-plum-84.json
└── README.md
```

この段階で最も重要なのは、WebHID APIの理解です。WebHID APIは、ブラウザから直接USB HIDデバイスと通信できる、比較的新しい技術です。MDNのドキュメントを読むことをお勧めしますが、基本的なパターンは次の三つに集約されます。デバイスの選択、デバイスへのコマンド送信、そしてデバイスからのレスポンス受信です。

デバイスの選択は、`navigator.hid.requestDevice()`を呼び出すことで実現します。この関数を呼ぶと、ブラウザがデバイス選択ダイアログを表示します。ユーザーがキーボードを選択すると、そのデバイスオブジェクトが返されます。セキュリティ上の理由から、この関数は必ずユーザーのジェスチャー（ボタンクリックなど）に反応して呼び出す必要があります。ページを開いた瞬間に自動的にデバイスに接続する、ということはできません。

コマンドの送信は、`device.sendReport()`または`device.sendFeatureReport()`を使います。NIZキーボードの場合、cho45さんの解析によれば、Feature Reportを使っているようです。送信するデータは、Uint8Array形式のバイト列です。どんなバイト列を送ればどんな動作をするのかは、cho45さんのRubyコードに全て記述されているので、それを参考にJavaScriptに移植していきます。

レスポンスの受信は、`inputreport`イベントを使います。デバイスにイベントリスナーを登録しておくと、デバイスからデータが送られてきたときに、そのコールバック関数が呼ばれます。受信したデータも、Uint8Array形式のバイト列として渡されます。このバイト列を解析して、必要な情報を取り出します。

この第一段階で作るプロトタイプは、次のような機能を持ちます。ユーザーがボタンをクリックすると、デバイス選択ダイアログが表示されます。ユーザーがNIZキーボードを選択すると、そのキーボードに接続して、バージョン情報を取得します。そして、現在のキーマップを読み取って、コンソールに表示します。UIは最小限でかまいません。接続ボタンと、ステータス表示のテキストエリアがあれば十分です。

この段階の目標は、完璧なアプリを作ることではありません。WebHID APIの使い方を実際に手を動かして学び、NIZキーボードとの通信が成功することを確認することです。ここで得られた知識と経験が、次の段階の土台となります。

### プロトコル理解とコード解析

次に、cho45さんのRubyコードを徹底的に読み解きます。このステップは、実装の前に必ず行うべき重要な作業です。プロトコルを理解せずにコードを書き始めると、後で行き詰まったり、無駄な試行錯誤に時間を費やしたりすることになります。

cho45さんのリポジトリ（https://github.com/cho45/niz-tools-ruby）から、特に`niz.rb`ファイルを開きます。このファイルには、NIZとの通信プロトコルの全てが記述されています。Rubyに馴染みがなくても、コードの構造を追っていけば、何が行われているかは理解できるはずです。

まず注目すべきは、`open`メソッドです。ここでUSB HIDデバイスを開いて、初期化コマンドを送っています。どんなバイト列を送っているのか、一つ一つ確認してください。例えば、最初に`0xAA 0x55`のような特定のバイト列を送っているかもしれません。これはマジックナンバーと呼ばれるもので、「これからNIZのプロトコルで通信しますよ」という合図のようなものです。多くのハードウェアプロトコルでは、このような識別子が使われます。

次に`version`メソッドを見ます。これはキーボードのファームウェアバージョンを取得する処理です。特定のコマンドを送信して、レスポンスを待ち、返ってきたバイト列からバージョン番号を取り出しています。このパターンは、以降の全ての操作の基本となります。コマンドを送る、レスポンスを待つ、バイト列を解析する、という三段階の処理です。

`read_all`メソッドは、全てのキーのマッピングを読み取る処理です。おそらく、キーIDとレベルIDを指定してそのキーの設定を問い合わせ、レスポンスとしてHWCODEを受け取る、という処理を繰り返しているはずです。Ruby版ではProgressBarを使っていますが、これは処理に時間がかかるため、ユーザーに進捗を見せる必要があるからです。Webアプリでも同様に、進捗表示が必要になります。

`write_all`メソッドも同様に、全キーのマッピングを書き込む処理です。重要なのは、cho45さんのコメントに「キーマップの書きかえコマンドを送ると、まず既存のキーマップが全てリセットされる」と書かれている点です。これはNIZのプロトコルの仕様なので、必ず「読み取り→変更→書き込み」という流れで処理する必要があります。ユーザーが一つのキーだけを変更したくても、内部的には全キーマップを扱うことになります。この制約を理解しておかないと、実装で困ることになるでしょう。

そして最も重要なのが`HWCODE`の定義です。これはNIZ独自のキーコード体系で、USB HIDのスキャンコードとは別の番号が割り当てられています。例えば、HWCODEの4番はAキーを表すかもしれませんが、USB HID Scanc

odeの4番とは必ずしも一致しません。cho45さんが「気合で作った」と書いているこのマッピングテーブルは、あなたの実装にとって金鉱のようなものです。この対応表があるからこそ、Webアプリが実装できるのです。

このマッピングテーブルを、そのままJavaScriptのオブジェクトとして移植します。この作業は地味ですが、正確にやらないとキーが意図しない動作をしてしまうので、慎重に進めてください。一つ一つのエントリーを確認しながら、コピーしていきます。後で間違いを見つけると修正が大変なので、最初から丁寧にやることが結局は早道です。

```javascript
// cho45さんのRubyコードから移植したHWCODEマッピング
const HWCODE = {
  4: { name: "A", category: "basic", usb: 0x04 },
  5: { name: "B", category: "basic", usb: 0x05 },
  // ... 全てのコードを移植
  68: { name: "Left GUI", category: "modifier", usb: 0xE3 },
  127: { name: "Mute", category: "media", usb: 0xE2 },
  // cho45さんの定義を全て含める
};
```

コードを読む際のコツは、実際に手を動かしてメモを取ることです。「このコマンドは何をするのか」「このバイトは何を意味するのか」を自分の言葉でまとめておくと、後で実装するときに迷いません。Rubyのコードとあなたが書くJavaScriptのコードを並べて見比べられるようにしておくと、移植作業が楽になります。

### 最小限の接続テストの実装

理論的な理解ができたら、実際にコードを書き始めます。最初は、デバイスに接続して基本的な情報を取得するだけの最小限の実装から始めましょう。この段階では、キーマップの読み書きはまだ実装しません。まずは接続して、バージョン情報を取得できることを確認します。

`index.html`は本当にシンプルで構いません。ボタンが一つあって、それをクリックするとデバイスに接続する、というだけです。WebHID APIはユーザーのジェスチャー（クリックなど）に反応して呼び出す必要があるので、必ずボタンクリックから開始します。自動的に接続を試みるようなコードは、ブラウザのセキュリティポリシーによってブロックされます。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>NIZ Keyboard Configurator</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div class="container">
    <h1>NIZ Keyboard Configurator</h1>
    <button id="connect-btn">キーボードに接続</button>
    <div id="status">未接続</div>
    <div id="info"></div>
  </div>
  <script src="app.js"></script>
</body>
</html>
```

`app.js`では、まず`navigator.hid`が使えるかどうかをチェックします。古いブラウザやFirefoxでは使えないので、エラーメッセージを表示する必要があります。このチェックを怠ると、ユーザーは何が問題なのか分からず混乱してしまいます。

```javascript
// WebHID APIのサポート確認
if (!navigator.hid) {
  document.getElementById('status').textContent = 
    'このブラウザはWebHID APIをサポートしていません。Chrome、Edge、またはOperaをお使いください。';
  document.getElementById('connect-btn').disabled = true;
}

// 接続ボタンのイベントリスナー
document.getElementById('connect-btn').addEventListener('click', async () => {
  try {
    await connectToKeyboard();
  } catch (error) {
    console.error('接続エラー:', error);
    document.getElementById('status').textContent = `エラー: ${error.message}`;
  }
});

async function connectToKeyboard() {
  // ステータス更新
  document.getElementById('status').textContent = 'デバイスを選択してください...';
  
  // デバイス選択ダイアログを表示
  // この段階では、フィルタは使わず全てのHIDデバイスを表示
  const devices = await navigator.hid.requestDevice({ filters: [] });
  
  if (devices.length === 0) {
    document.getElementById('status').textContent = 'デバイスが選択されませんでした';
    return;
  }
  
  const device = devices[0];
  
  // デバイス情報を表示
  const vendorId = `0x${device.vendorId.toString(16).padStart(4, '0')}`;
  const productId = `0x${device.productId.toString(16).padStart(4, '0')}`;
  document.getElementById('info').textContent = 
    `Vendor ID: ${vendorId}, Product ID: ${productId}`;
  
  // 接続を確立
  await device.open();
  document.getElementById('status').textContent = '接続成功！';
  
  // バージョン情報を取得してみる
  // （ここでcho45さんのコードから学んだコマンドを送信）
  // 実際のコマンドバイト列は、cho45さんのコードを参照して実装
}
```

このコードで、デバイスへの接続までは成功するはずです。ここまで動いたら、次はcho45さんのコードを参考にして、実際のコマンドを送信してみます。バージョン情報取得のコマンドは比較的シンプルなので、最初の実装に適しています。

レスポンスを待つ処理は、Promiseでラップすると扱いやすくなります。`inputreport`イベントは非同期で発火するため、async/awaitパターンと組み合わせるためにPromiseが必要です。

```javascript
// コマンドを送信してレスポンスを待つヘルパー関数
function sendCommandAndWaitResponse(device, commandBytes) {
  return new Promise((resolve, reject) => {
    // タイムアウトを設定（3秒待ってもレスポンスがなければエラー）
    const timeout = setTimeout(() => {
      device.removeEventListener('inputreport', handler);
      reject(new Error('レスポンスのタイムアウト'));
    }, 3000);
    
    // レスポンスハンドラー
    const handler = (event) => {
      clearTimeout(timeout);
      device.removeEventListener('inputreport', handler);
      // レスポンスデータを返す
      resolve(new Uint8Array(event.data.buffer));
    };
    
    // イベントリスナーを登録
    device.addEventListener('inputreport', handler);
    
    // コマンドを送信
    device.sendFeatureReport(commandBytes[0], commandBytes.slice(1))
      .catch(error => {
        clearTimeout(timeout);
        device.removeEventListener('inputreport', handler);
        reject(error);
      });
  });
}
```

このヘルパー関数を使えば、コマンドの送受信がasync/awaitで書けるようになり、コードが読みやすくなります。タイムアウト処理も含まれているので、通信が失敗した場合でも、永遠に待ち続けることはありません。

### 第二段階: サーバーレスAPIの導入

第一段階で動作するプロトタイプができたら、次はサーバーレスAPIを導入します。この段階の目標は、キーボード定義をサーバー側で管理する仕組みを構築することです。ユーザーがキーボードを接続したら、そのベンダーIDとプロダクトIDをAPIに送信して、対応する定義を取得します。

サーバーレスという選択肢は、個人プロジェクトにとって非常に魅力的です。従来のサーバー運用では、サーバーマシンの管理、OSのアップデート、セキュリティパッチの適用、負荷分散の設定など、多くの運用作業が必要でした。しかしサーバーレスプラットフォームを使えば、これらの作業は全てクラウドプロバイダーが行ってくれます。あなたはAPIのコードを書くことだけに集中できます。

さらに、コストの面でも有利です。Cloudflare Workersを例に取ると、無料プランで1日あたり10万リクエストまで処理できます。NIZキーボード設定ツールのようなニッチなツールで、この制限に到達することはまずないでしょう。つまり、完全に無料でスケーラブルなバックエンドが構築できるのです。

Cloudflare Workersの開発環境をセットアップするには、まずNode.jsをインストールします。次に、Wranglerというコマンドラインツールをインストールします。Wranglerは、Workersのプロジェクト管理、ローカル開発、デプロイを担当するツールです。

```bash
# Wranglerのインストール
npm install -g wrangler

# 新しいプロジェクトを作成
wrangler init niz-api

# ローカル開発サーバーを起動
wrangler dev
```

Workersのコードは、非常にシンプルです。リクエストを受け取り、適切なレスポンスを返す関数を書くだけです。まずは最小限のAPIから始めて、徐々に機能を追加していきます。

```javascript
// src/index.js - Cloudflare Workersのエントリーポイント
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    
    // CORSヘッダーを設定（フロントエンドからのアクセスを許可）
    const corsHeaders = {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    };
    
    // プリフライトリクエストへの対応
    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }
    
    // ルーティング
    if (url.pathname === '/api/keyboard') {
      return handleGetKeyboard(url, corsHeaders);
    } else if (url.pathname === '/api/hwcodes') {
      return handleGetHWCodes(corsHeaders);
    }
    
    return new Response('Not Found', { status: 404, headers: corsHeaders });
  }
};

// キーボード定義を取得するハンドラー
async function handleGetKeyboard(url, corsHeaders) {
  const vendorId = url.searchParams.get('vendorId');
  const productId = url.searchParams.get('productId');
  
  if (!vendorId || !productId) {
    return new Response('Missing parameters', { 
      status: 400,
      headers: corsHeaders 
    });
  }
  
  // この段階では、定義をハードコーディング
  // 後でデータベースから取得するように変更する
  const definitions = {
    '0x0483:0x5710': {
      device: {
        id: 'niz-plum-84',
        name: 'NiZ Plum 84',
        manufacturer: 'NiZ',
        vendorId: '0x0483',
        productId: '0x5710',
        keyCount: 84
      },
      // ... 完全な定義
    }
  };
  
  const key = `${vendorId}:${productId}`;
  const definition = definitions[key];
  
  if (!definition) {
    return new Response('Keyboard not found', { 
      status: 404,
      headers: corsHeaders 
    });
  }
  
  // キャッシュヘッダーを設定（1時間キャッシュ）
  return new Response(JSON.stringify(definition), {
    headers: {
      ...corsHeaders,
      'Content-Type': 'application/json',
      'Cache-Control': 'public, max-age=3600'
    }
  });
}

// HWCODEマッピングを取得するハンドラー
async function handleGetHWCodes(corsHeaders) {
  // cho45さんのマッピングをJavaScriptオブジェクトとして定義
  const hwcodes = {
    4: { name: "A", category: "basic" },
    5: { name: "B", category: "basic" },
    // ... 全てのHWCODE
  };
  
  // 長期間キャッシュ（HWCODEは滅多に変更されない）
  return new Response(JSON.stringify({ hwcodes }), {
    headers: {
      ...corsHeaders,
      'Content-Type': 'application/json',
      'Cache-Control': 'public, max-age=86400' // 24時間
    }
  });
}
```

このコードでは、まだデータベースは使っていません。定義はJavaScriptのオブジェクトとしてハードコーディングされています。これは意図的な設計です。いきなり複雑なシステムを作るのではなく、まずシンプルな形で動かして、問題ないことを確認してから、次の段階に進みます。

フロントエンド側では、キーボード定義をAPIから取得するように変更します。

```javascript
async function connectToKeyboard() {
  // ... デバイス選択の部分は同じ ...
  
  // APIに定義を問い合わせ
  const definition = await fetchKeyboardDefinition(vendorId, productId);
  
  if (!definition) {
    alert('このキーボードはまだ対応していません。\n' +
          'Vendor ID: ' + vendorId + '\n' +
          'Product ID: ' + productId + '\n' +
          'この情報をGitHubのIssueで報告していただけると、対応を検討します。');
    return null;
  }
  
  console.log(`接続: ${definition.device.name}`);
  
  // デバイスを開く
  await device.open();
  
  // 定義を使ってUIを構築
  renderKeyboardLayout(definition);
  
  return { device, definition };
}

async function fetchKeyboardDefinition(vendorId, productId) {
  try {
    const response = await fetch(
      `https://niz-api.your-domain.workers.dev/api/keyboard?vendorId=${vendorId}&productId=${productId}`
    );
    
    if (response.status === 404) {
      return null; // 未対応のキーボード
    }
    
    if (!response.ok) {
      throw new Error(`API error: ${response.status}`);
    }
    
    return await response.json();
  } catch (error) {
    console.error('定義の取得に失敗:', error);
    throw error;
  }
}
```

この実装により、フロントエンドはどのキーボードモデルが存在するかを事前に知る必要がなくなります。ユーザーがキーボードを接続すると、そのIDをAPIに送信し、APIが「対応している/していない」を教えてくれます。未対応の場合は、ユーザーにVendor IDとProduct IDを表示して、その情報をGitHubで報告してもらうように促します。こうすることで、コミュニティからの情報提供を集めやすくなります。

### 第三段階: データベースの統合

第二段階までで、APIを使った動的な定義取得は実現できました。しかし、定義がコードにハードコーディングされているため、新しいキーボードに対応するたびにコードを変更してデプロイする必要があります。第三段階では、データベースを導入することで、この問題を解決します。

Cloudflare D1は、Cloudflare Workersと統合されたSQLiteベースのデータベースサービスです。SQLiteはシンプルで習得しやすく、小規模から中規模のアプリケーションに最適です。D1も無料プランがあり、個人プロジェクトなら十分な容量が提供されます。

まず、D1データベースを作成します。

```bash
# D1データベースを作成
wrangler d1 create niz-keyboards

# wrangler.tomlに設定を追加（コマンドの出力に表示される情報を使用）
```

`wrangler.toml`ファイルに、データベースのバインディング設定を追加します。

```toml
name = "niz-api"
main = "src/index.js"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "DB"
database_name = "niz-keyboards"
database_id = "your-database-id-here"
```

次に、データベースのスキーマを作成します。

```sql
-- schema.sql

-- キーボード定義テーブル
CREATE TABLE keyboards (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    vendor_id TEXT NOT NULL,
    product_id TEXT NOT NULL,
    name TEXT NOT NULL,
    manufacturer TEXT NOT NULL,
    definition_json TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(vendor_id, product_id)
);

-- HWCODEマッピングテーブル
CREATE TABLE hwcodes (
    code INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    category TEXT NOT NULL,
    description TEXT
);

-- インデックスを作成（検索を高速化）
CREATE INDEX idx_keyboards_ids ON keyboards(vendor_id, product_id);
CREATE INDEX idx_hwcodes_category ON hwcodes(category);
```

スキーマをデータベースに適用します。

```bash
wrangler d1 execute niz-keyboards --file=./schema.sql
```

次に、初期データを投入します。cho45さんのHWCODEマッピングと、あなたが持っているNIZキーボードの定義をSQLのINSERT文に変換します。

```sql
-- seed.sql

-- HWCODEマッピングの挿入
INSERT INTO hwcodes (code, name, category, description) VALUES
(4, 'A', 'basic', 'Aキー'),
(5, 'B', 'basic', 'Bキー'),
-- ... 全てのHWCODEを挿入

(68, 'Left GUI', 'modifier', '左Windowsキー / 左Commandキー'),
(127, 'Mute', 'media', 'ミュート');

-- キーボード定義の挿入
INSERT INTO keyboards (vendor_id, product_id, name, manufacturer, definition_json) VALUES
('0x0483', '0x5710', 'NiZ Plum 84', 'NiZ', '{
  "device": {
    "id": "niz-plum-84",
    "name": "NiZ Plum 84",
    "manufacturer": "NiZ",
    "vendorId": "0x0483",
    "productId": "0x5710",
    "keyCount": 84
  },
  "protocol": {
    "levels": [...]
  },
  "layout": {
    "keys": [...]
  }
}');
```

データを投入します。

```bash
wrangler d1 execute niz-keyboards --file=./seed.sql
```

これで、データベースの準備が整いました。次に、Workersのコードを変更して、データベースから定義を取得するようにします。

```javascript
// ハードコーディングされた定義を削除し、データベースクエリに置き換え
async function handleGetKeyboard(url, env, corsHeaders) {
  const vendorId = url.searchParams.get('vendorId');
  const productId = url.searchParams.get('productId');
  
  if (!vendorId || !productId) {
    return new Response('Missing parameters', { 
      status: 400,
      headers: corsHeaders 
    });
  }
  
  // データベースから定義を検索
  // env.DBは、wrangler.tomlで設定したバインディング名
  const result = await env.DB.prepare(
    'SELECT definition_json FROM keyboards WHERE vendor_id = ? AND product_id = ?'
  ).bind(vendorId, productId).first();
  
  if (!result) {
    return new Response('Keyboard not found', { 
      status: 404,
      headers: corsHeaders 
    });
  }
  
  // キャッシュヘッダーを設定（1時間キャッシュ）
  return new Response(result.definition_json, {
    headers: {
      ...corsHeaders,
      'Content-Type': 'application/json',
      'Cache-Control': 'public, max-age=3600'
    }
  });
}

async function handleGetHWCodes(env, corsHeaders) {
  // データベースから全HWCODEを取得
  const results = await env.DB.prepare(
    'SELECT code, name, category, description FROM hwcodes ORDER BY code'
  ).all();
  
  // オブジェクト形式に変換
  const hwcodes = {};
  for (const row of results.results) {
    hwcodes[row.code] = {
      name: row.name,
      category: row.category,
      description: row.description
    };
  }
  
  // 長期間キャッシュ
  return new Response(JSON.stringify({ hwcodes }), {
    headers: {
      ...corsHeaders,
      'Content-Type': 'application/json',
      'Cache-Control': 'public, max-age=86400'
    }
  });
}
```

この変更により、新しいキーボードに対応する際は、データベースにINSERT文を実行するだけで済むようになります。コードの変更やデプロイは不要です。

```bash
# 新しいキーボード定義を追加
wrangler d1 execute niz-keyboards --command="INSERT INTO keyboards (vendor_id, product_id, name, manufacturer, definition_json) VALUES ('0x1234', '0x5678', 'NiZ Plum 87', 'NiZ', '{...}')"
```

さらに便利なのは、将来的にWeb管理画面を作れば、ブラウザから直接定義を追加・編集できるようになることです。SQLコマンドを打つ必要すらなくなります。

### UIの実装とユーザー体験

ここまでで、バックエンドとデータ管理の仕組みが整いました。次は、ユーザーが実際に操作するフロントエンドのUIを作り込んでいきます。UIの設計では、使いやすさと直感性を最優先に考えます。

UIは大きく分けて三つのセクションで構成されます。まず、デバイス接続エリアです。ここには「キーボードに接続」ボタンと、接続状態の表示があります。接続されたら、キーボードのモデル名やファームウェアバージョンを表示します。ユーザーは、正しいキーボードに接続できたことを視覚的に確認できるので、安心して作業を進められます。

次に、キーボードレイアウト表示エリアです。ここがメインのインターフェースで、実際のキーボードのレイアウトを視覚的に表現します。各キーをHTML要素として配置し、CSSでスタイリングします。キーをクリックすると選択状態になり、そのキーに割り当てられている現在の機能が表示されます。そして、下部のキー機能選択エリアから新しい機能を割り当てられます。

キー機能選択エリアでは、使用可能な全ての機能をカテゴリ別に表示します。例えば、「基本キー」「修飾キー」「メディアキー」「ファンクションキー」などのタブを作り、それぞれのカテゴリ内で選択できるようにします。この部分は、HWCODEマッピングのカテゴリ情報を元に動的に生成できます。

レイアウトの実装には、CSS Gridが最適です。各キーの位置とサイズは、キーボード定義のJSONから取得できるので、それを元にCSSのstyleプロパティを動的に設定します。

```javascript
function renderKeyboardLayout(definition) {
  const container = document.getElementById('keyboard-layout');
  container.innerHTML = ''; // クリア
  
  // キーボード全体のサイズを計算
  const maxX = Math.max(...definition.layout.keys.map(k => k.x + k.width));
  const maxY = Math.max(...definition.layout.keys.map(k => k.y + k.height));
  
  // コンテナのサイズを設定
  const keyUnit = 60; // 1ユニット = 60px
  container.style.width = `${maxX * keyUnit}px`;
  container.style.height = `${maxY * keyUnit}px`;
  container.style.position = 'relative';
  
  // 各キーのHTML要素を生成
  definition.layout.keys.forEach(keyDef => {
    const keyElement = document.createElement('div');
    keyElement.className = 'key';
    keyElement.dataset.keyId = keyDef.id;
    
    // CSSで位置とサイズを設定
    keyElement.style.position = 'absolute';
    keyElement.style.left = `${keyDef.x * keyUnit}px`;
    keyElement.style.top = `${keyDef.y * keyUnit}px`;
    keyElement.style.width = `${keyDef.width * keyUnit - 4}px`; // 4pxはマージン
    keyElement.style.height = `${keyDef.height * keyUnit - 4}px`;
    
    // ラベルを表示
    const label = document.createElement('span');
    label.className = 'key-label';
    label.textContent = keyDef.label;
    keyElement.appendChild(label);
    
    // 現在の割り当てを表示（後で実装）
    const assignment = document.createElement('span');
    assignment.className = 'key-assignment';
    keyElement.appendChild(assignment);
    
    // クリックイベントを設定
    keyElement.addEventListener('click', () => {
      selectKey(keyDef.id);
    });
    
    container.appendChild(keyElement);
  });
}

function selectKey(keyId) {
  // 全てのキーの選択状態を解除
  document.querySelectorAll('.key').forEach(el => {
    el.classList.remove('selected');
  });
  
  // クリックされたキーを選択状態に
  const keyElement = document.querySelector(`[data-key-id="${keyId}"]`);
  keyElement.classList.add('selected');
  
  // 現在の割り当てを表示
  // ... 実装
  
  // キー機能選択エリアを更新
  // ... 実装
}
```

CSSでは、キーの見た目をスタイリングします。

```css
.key {
  background: linear-gradient(180deg, #f0f0f0 0%, #e0e0e0 100%);
  border: 1px solid #c0c0c0;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  cursor: pointer;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  transition: all 0.2s ease;
}

.key:hover {
  background: linear-gradient(180deg, #f5f5f5 0%, #e5e5e5 100%);
  transform: translateY(-1px);
  box-shadow: 0 3px 6px rgba(0,0,0,0.15);
}

.key.selected {
  background: linear-gradient(180deg, #4a90e2 0%, #357abd 100%);
  color: white;
  border-color: #2968a8;
}

.key-label {
  font-size: 14px;
  font-weight: 500;
}

.key-assignment {
  font-size: 10px;
  color: #666;
  margin-top: 4px;
}

.key.selected .key-assignment {
  color: rgba(255,255,255,0.8);
}
```

このスタイリングにより、キーは実際のキーボードのキーキャップのような見た目になります。ホバー時のアニメーションや、選択時の色の変化により、ユーザーは自分の操作がシステムに認識されていることを直感的に理解できます。

### エラーハンドリングとユーザーフィードバック

Webアプリケーションでは、さまざまな問題が発生する可能性があります。USBケーブルが抜けた、通信がタイムアウトした、ブラウザがWebHID APIをサポートしていない、など。これらに適切に対処することが、良いユーザー体験を提供する鍵となります。

全ての非同期操作をtry-catchでラップして、エラーが起きたら分かりやすいメッセージを表示します。「通信エラーが発生しました。USBケーブルを確認して、再接続してください」のような具体的な指示があると、ユーザーは何をすべきか分かります。単に「エラーが発生しました」だけでは、ユーザーは途方に暮れてしまいます。

また、処理中は適切なローディング表示を出すことも重要です。キーマップの読み書きには数秒かかるため、その間、プログレスバーと現在の作業内容を表示します。「キーマップを読み取り中... 45/198」のような表示があれば、ユーザーは進捗を確認でき、安心して待つことができます。

```javascript
// 進捗表示の実装
function updateProgress(current, total, message) {
  const percentage = Math.round((current / total) * 100);
  document.getElementById('progress').value = percentage;
  document.getElementById('progress-text').textContent = 
    `${message} (${current}/${total} - ${percentage}%)`;
}

// エラーメッセージの表示
function showError(message, technicalDetails) {
  const errorDiv = document.createElement('div');
  errorDiv.className = 'error-message';
  errorDiv.innerHTML = `
    <strong>エラーが発生しました</strong>
    <p>${message}</p>
    ${technicalDetails ? `<details><summary>技術的な詳細</summary><pre>${technicalDetails}</pre></details>` : ''}
  `;
  document.body.appendChild(errorDiv);
  
  // 5秒後に自動的に消える
  setTimeout(() => errorDiv.remove(), 5000);
}
```

書き込み操作では、特に慎重なエラーハンドリングが必要です。途中で通信エラーが起きた場合、キーボードが不完全な状態になる可能性があります。そのため、書き込み中は「USBケーブルを絶対に抜かないでください」という警告を目立つように表示すべきです。また、書き込みが失敗した場合のリカバリー方法も、ユーザーに明示的に伝える必要があります。

## 🔧 技術的な補足事項

### WebHID APIの制約と対策

WebHID APIは比較的新しい技術なので、いくつか注意点があります。まず、HTTPSが必須です。開発中はlocalhostで動作しますが、公開時はHTTPSが必要です。GitHub Pagesなら自動的にHTTPSが有効なので、この要件は自然に満たされます。

また、ユーザージェスチャーが必要なので、ページを開いた瞬間に自動的にデバイスに接続することはできません。必ずボタンクリックなどのアクションを経由する必要があります。これはセキュリティ上の制約で、ユーザーの明示的な同意なしにUSBデバイスにアクセスできてしまったら危険だからです。

ブラウザのサポート状況も確認が必要です。現在、WebHID APIはChrome、Edge、OperaなどのChromium系ブラウザでのみ動作します。SafariやFirefoxでは使えません。アプリケーションの起動時に、ブラウザのサポート状況をチェックして、未対応の場合は適切なメッセージを表示する必要があります。

### デバッグのヒント

通信のログをしっかり取ることが、デバッグの鍵です。送信するバイト列と、受信したバイト列を全てコンソールに出力しましょう。バイト列は16進数で表示すると、cho45さんのRubyコードと比較しやすくなります。

```javascript
function logBytes(label, bytes) {
  const hex = Array.from(bytes)
    .map(b => b.toString(16).padStart(2, '0'))
    .join(' ');
  console.log(`${label}: ${hex}`);
}

// 使用例
logBytes('送信', commandBytes);
logBytes('受信', responseBytes);
```

また、cho45さんのRuby版を並行して動かして、同じ操作で同じバイト列が送受信されているか比較すると、問題の切り分けが楽になります。違いが見つかれば、そこが問題の原因です。

### パフォーマンスの考慮

198回の読み書き操作は、一つ一つは速くても、全体では数秒かかります。ユーザーに待たせる時間を最小化するため、不要な待機時間を削除したり、可能なら並列処理を検討したりすることも考えられます。

ただし、NIZのファームウェアが並列リクエストに対応しているかどうかは不明なので、まずは順次処理で実装して、動作を確認してから最適化を考えるべきです。早すぎる最適化は、プログラミングの大敵です。まず動くものを作り、それから速くする、という順序が正しいアプローチです。

## 🎓 学びのポイント

このプロジェクトを通じて、いくつかの重要なスキルが身につきます。WebHID APIの実践的な使い方、USB HID通信の理解、非同期JavaScriptのパターン、サーバーレスアーキテクチャの実装、そしてデータ駆動設計の実践です。

特に、既存のコード（Rubyで書かれたもの）を別の言語（JavaScript）に移植する経験は、非常に価値があります。単なる翻訳ではなく、それぞれの言語の特性や、実行環境の違いを理解しながら、同等の機能を実現する必要があるからです。プログラミング言語は違っても、プログラミングの本質的な考え方は共通です。このプロジェクトを通じて、その普遍的な原理を体得できるでしょう。

また、コミュニティに貢献するという経験も得られます。オープンソースの世界では、「巨人の肩に乗る」という言葉があります。cho45さんの成果があるからこそ、あなたはWeb版を作れる。そしてあなたが作ったWeb版を使って、また誰かが新しいものを作るかもしれない。この循環が、技術コミュニティを豊かにしていきます。

## 📚 参考リソース

開発中に役立つリンクをまとめておきます。

WebHID API: https://developer.mozilla.org/en-US/docs/Web/API/WebHID_API
cho45さんのリポジトリ: https://github.com/cho45/niz-tools-ruby
VIA（参考実装）: https://github.com/the-via/app
Cloudflare Workers: https://developers.cloudflare.com/workers/
Cloudflare D1: https://developers.cloudflare.com/d1/

これらのリソースを活用しながら、一歩ずつ進めていってください。最初は難しく感じるかもしれませんが、各ステップをクリアしていくごとに、理解が深まっていくはずです。

## 🚀 次のステップ

基本的な機能が完成したら、さらに改良を加えていくことができます。例えば、キーマップのプロファイル機能を実装して、複数の設定を保存・切り替えできるようにしたり、Web管理画面を作って、ブラウザからキーボード定義を編集できるようにしたり、コミュニティがキーボード定義を投稿できる仕組みを作ったりすることができます。

しかし、まずは基本機能を完成させることが最優先です。完璧を目指して永遠に完成しないより、80%の完成度で公開して、コミュニティからフィードバックを得ながら改善していく方が、ずっと価値があります。完璧な計画は存在しません。実際に使ってもらうことで初めて、本当に必要な機能が見えてきます。

頑張ってください！🚀
