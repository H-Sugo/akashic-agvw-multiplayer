# akashic-agvw-multiplayer

このリポジトリは、
｢akashicの自作ゲームをagvw利用でブラウザ起動し、マルチプレイのクライアント側で動かす｣という目標の
進捗記録用です。

## プロジェクトの特長
 1. クライアント側ロジックの構築 (index.html)
   * API連携の自動化: ブラウザ側の JavaScript で system-api-server
     を叩き、接続に必要な「プレイトークン」を自動取得するフローを実装しました。これにより、ユーザーによる手動の ID
     入力やコマンド操作を不要にしました。
   * Passiveモードへの対応: agv.ExecutionMode.Passive
     を指定し、サーバー側で動くゲームの「観戦・参加者」として正しく同期接続する仕組みを確立しました。
   * 起動ボタンの設置:
     ブラウザのセキュリティ制限（AudioContext）により音が鳴らなくなる問題を防ぐため、ユーザーのクリックをトリガーに起動
     する UI を採用しました。

2. 通信インフラの整備 (nginx.conf)
   * リバースプロキシの導入: ブラウザの強力なセキュリティ制限（CORSエラー）により、別ポートの API
     へのアクセスが遮断される問題を解決しました。
   * 仕組み: ポート 18080 (Webサーバー) の /api/ へのアクセスを、内部でポート 3100 (APIサーバー)
     へ転送するように設定。ブラウザから見て「同じドメイン内での通信」に見せかけることで、安全かつ確実な通信を実現しまし
     た。

3. ゲーム設定の最適化 (content.json)
   *  パス解決の修正: エンジンや素材の読み込みパスを相対パスからルートパス (/contents/...)
     に変更。ブラウザがどの階層からアクセスしても、確実に必要なファイルを見つけられるようにしました。
   * 通信ライブラリの追加: マルチプレイ通信に不可欠な playlogClient.js を読み込みリストに追加しました。

4. インフラの安定化 (docker-compose.yml)
   * 起動順序の制御: depends_on を追加し、Webサーバー（nginx）が起動する前に必ず API
     サーバーが立ち上がっているように設定しました。これにより、起動時の名前解決エラー（コンテナが落ちる問題）を根絶しま
     した。

## セットアップ手順

### 1. 依存関係のインストール
プロジェクトの部品を結合し、必要なツールをインストールします。

```bash
npm run prepare
```

### 2. インフラの起動
Docker を使用してサーバー群をバックグラウンドで起動します。

```bash
docker-compose up -d
```

### 3. データベースの初期化
起動後、数秒待ってからデータベースのテーブルを作成します。

```bash
npm run db:migrate
```

### 4. サーバー側でのゲーム起動 (Play ID: 1)
以下の PowerShell コマンドを実行して、サーバー側でゲームを立ち上げます。

```powershell
$play = Invoke-RestMethod -Uri "http://localhost:3100/v1.0/plays" -Method Post -ContentType "application/json" -Body '{"gameCode": "rocket-game"}'
$playId = $play.data.id
$body = @"
{
  "gameCode": "rocket-game",
  "entryPoint": "gsk/contents/engines/dist/3.13.4-0/entry.js",
  "cost": 1,
  "modules": [
    { "code": "dynamicPlaylogWorker", "values": { "playId": "$playId", "executionMode": "active" } },
    { "code": "akashicEngineParameters", "values": { "gameConfigurations": ["gsk/rocket-game/game.json"] } }
  ]
}
"@
Invoke-RestMethod -Uri "http://localhost:3100/v1.0/instances" -Method Post -ContentType "application/json" -Body $body
```

### 5. ブラウザで確認
最新のプレイに自動接続する以下の URL を開きます。

`http://localhost:18080/contents/index.html`
