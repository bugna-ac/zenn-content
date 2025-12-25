50代の社内SEのような立場で働いている。1990年代の古いシステムの保守や運用に携わりつつ、最新技術の導入にも積極的に取り組んでいる。最近は、VS Code 上のAIエージェントによるプロジェクト作成とObsidianによる業務管理構築に関心があり、社内のデータマネジメントの最適化を目指している。趣味は星見とバイクとガジェット収集で、週末はSUPと模様替えでリセットすることが多い。

# npx zenn preview でZennのガイドにアクセスエラー

## 現象・症状
- npx zenn previewは、Markdownファイルを、Zenn Editor で閲覧できる、Zenn Cliのツールである
- node でWebサーバを起動しており、ローカルのMarkdownはレンダリングされる
- しかし、Zennが作成している、「記事の作成ガイド」　「マークダウン記法」などは外部リンクである
- イントラ内のPCから、プログラムで外部ページにアクセスする際、ZScaler が間に入り、取得できないことがある
	- Python , Javascript
- そこで、PC内の証明書の中から、ZScalerの証明書を取得し、BASE64のPEM（拡張子は、.crt 、.pem）で保存することで、この証明書を外部ページの接続時に使うことで、接続ができるようになる
- これは、EdgeなどのWebブラウザーでも、同様の処理がされているが、目に見えないので、認識しにくい社内環境である

```powershell
d: request to https://zenn.dev/api/articles/zenn-cli-guide failed, reason: unable to get local issuer certificate
    at ClientRequest.<anonymous> (S:\git\zenn-content\node_modules\zenn-cli\dist\server\zenn.js:292:62477)
    at ClientRequest.emit (node:events:520:28)
    at TLSSocket.socketErrorListener (node:_http_client:502:9)
    at TLSSocket.emit (node:events:520:28)
    at emitErrorNT (node:internal/streams/destroy:170:8)
    at emitErrorCloseNT (node:internal/streams/destroy:129:3)
    at process.processTicksAndRejections (node:internal/process/task_queues:82:21) {
  type: 'system',
  errno: 'UNABLE_TO_GET_ISSUER_CERT_LOCALLY',
  code: 'UNABLE_TO_GET_ISSUER_CERT_LOCALLY',
  erroredSysCall: undefined
}
```

## 解決方法

- 環境変数に、上記の証明書を示すことで、Nodeが、アクセスできるに用になる

```powershell
$env:NODE_EXTRA_CA_CERTS="S:\git\ZscalerRootCA.crt";npx zenn preview
```

## 技術的背景

ご指摘のとおり、Zscalerが社内TLSを中間者（MITM）として終端し、社内配布の「ZscalerRootCA.crt」を信頼しないNode.jsが証明書の連鎖を検証できず、`UNABLE_TO_GET_ISSUER_CERT_LOCALLY` を出している可能性が高い。**Pythonは`verify`やCAファイルで回避できるが、Node.jsでも同様の“追加CAを信頼させる”方法がある**。推奨順に対処法を整理する。

---

## 最も安全・推奨の対処法：`NODE_EXTRA_CA_CERTS` を使う

Node.jsはデフォルトでOSの証明書ストア（特にWindows）を参照せず、内蔵のOpenSSLバンドルCAを使うため、社内CAは未信頼になりがちである。**`NODE_EXTRA_CA_CERTS`** を環境変数で設定すると、**すべてのTLS接続で追加CAを信頼**するようになる。

1. **ZscalerのルートCAをPEM形式で用意**
    
    - ファイル例: `C:\certs\ZscalerRootCA.crt`（PEM / Base64形式の`-----BEGIN CERTIFICATE-----`〜`END CERTIFICATE`）
    - CER/DERの場合はPEMに変換しておく（社内配布がPEMならそのままでよい）。
2. **環境変数を設定**
    
    - **PowerShell**（セッション限定）
```powershell
		$env:NODE_EXTRA_CA_CERTS="C:\certs\ZscalerRootCA.crt"
		node -v  # テスト起動		
```
	- **cmd.exe** （セッション限定）
```cmd
        set NODE_EXTRA_CA_CERTS=C:\certs\ZscalerRootCA.crt
        node -v
```
        
    - **永続化（Windows ユーザー環境変数）**
    システムの「環境変数」設定で `NODE_EXTRA_CA_CERTS` に同じパスを登録。
    
3. **zenn-cli の実行前に反映**\ 例：
```bat
    set NODE_EXTRA_CA_CERTS=C:\certs\ZscalerRootCA.crt 
    npx zenn-cli preview
```    
	もしくは `npm run` など、起動シェルで環境変数を有効にした状態で実行する。

> この方法は**NodeのグローバルTLSスタックで追加CAを信頼**させるため、HTTPクライアント実装（`https`, `node-fetch`, `axios`など）に依存せず効く。社内CAの入替・更新時も、同じ環境変数を差し替えるだけでよい。

---

## 追加の考慮点（Zscalerのプロキシ設定が必要な場合）

Zscalerの構成によっては、**明示的プロキシ**（PAC/固定プロキシ）を通す必要があり、**証明書の問題だけでなく、プロキシ経由の設定も要る**。その場合は以下も併用する。

- **環境変数でプロキシを指定**（一般的なCLIはこれを尊重）
```bat
    set HTTPS_PROXY__=http://proxy.company.local:8080_
    
    _set HTTP_PROXY=http://proxy.company.local:8080
    
    set NO_PROXY=localhost,127.0.0.1,::1
```
- **NodeのHTTPクライアント側でプロキシAgentを明示**\ ライブラリによっては `https-proxy-agent` や `proxy-agent` を指定しないとHTTPSでプロキシへCONNECTしない。`zenn-cli`は内部で`node-fetch`等を使っている可能性があり、**ツール側が環境変数を尊重しない場合は、プロキシ設定の効きが弱い**。\ → まずは**`NODE_EXTRA_CA_CERTS`だけで再試行**し、ダメなら**`HTTPS_PROXY`も併せて**設定して挙動を見るのがよい。

---

## 代替・一時的な方法（非推奨）

- **個別コードでCAを渡す（自作サーバ/スクリプト向け）**\ 例：`https`標準モジュール
```javascript
    const https = require('https');
    
    const fs = require('fs');
    
      
    
    const agent = new https.Agent({
    
      ca: fs.readFileSync('C:/certs/ZscalerRootCA.crt', 'utf8'),
    
    });
    
      
    
    https.get('https://zenn.dev/api/articles/zenn-cli-guide', { agent }, (res) => {
    
      // …
    
    });
```

   `axios`や`node-fetch`でも`agent`を渡せる。**ただしCLI（zenn-cli）の内部には介入できない**ため、**グローバル変数の`NODE_EXTRA_CA_CERTS`の方が現実的**である。
    
- **`NODE_TLS_REJECT_UNAUTHORIZED=0`（強く非推奨）**\ 証明書検証を**完全に無効化**する。セキュリティ上危険であり、**社内規定違反の可能性が高い**。短期の切り分けテスト以外で使うべきでない。
```bat
    set NODE_TLS_REJECT_UNAUTHORIZED=0
```
    → 問題の再現切り分けにのみ使用し、**恒常運用では絶対に避ける**べきである。
    

---

## 切り分けチェック

1. **CA設定が効いているかを単体確認**
```bat
    set NODE_EXTRA_CA_CERTS=C:\certs\ZscalerRootCA.crt
    
	node -e "require('https').get('https://zenn.dev', r=>console.log('status',r.statusCode)).on('error',console.error)"
```
`status 200` 等が返れば、CAは信頼されている。
    
2. **プロキシが必須か確認**\ 社内のPACやZscalerポリシーで**直出しを禁止**していると、`ECONNRESET`やタイムアウト/接続拒否になる。`HTTPS_PROXY`を設定して同様のテストを行い、挙動変化を確認。
    

---

## よくある落とし穴

- **証明書形式がDER（バイナリ）で渡されている**\ NodeはPEM（Base64/ASCII）を想定する。`-----BEGIN CERTIFICATE-----`で始まる形式か確認。
    
- **中間CAが必要**\ Zscaler運用で企業ルートCAに加え**中間CAが存在**する場合がある。**ルート〜中間をすべて連結したPEM**を`NODE_EXTRA_CA_CERTS`に指定することで失敗が減る。
    
- **ファイルパスに日本語やスペース**\ 一部環境でパス解決に失敗することがあるため、**短い英数字のみのパス**を推奨。
    
- **Windowsの証明書ストアにインポートしただけでは効かない**\ NodeはOSストアを使わない（例外構成ありうるが一般的ではない）。**`NODE_EXTRA_CA_CERTS`が確実**である。
    

---

## 結論と推奨手順

1. Zscalerの**ルート/中間CAをPEMで用意**（連結OK）。
2. **`NODE_EXTRA_CA_CERTS`** にそのファイルを設定。
3. 必要なら **`HTTPS_PROXY`/`HTTP_PROXY`/`NO_PROXY`** を併用。
4. まずは `NODE_TLS_REJECT_UNAUTHORIZED=0` を使わずにテスト。
5. それでも失敗する場合は、**zenn-cliがプロキシ環境変数を尊重していない可能性**があるため、実行ログやネットワーク経路（PAC/明示プロキシ）を精査。
