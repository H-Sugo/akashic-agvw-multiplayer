# akashic-agvw-multiplayer

このリポジトリは、
｢akashicの自作ゲームをagvw利用でブラウザ起動し、マルチプレイのクライアント側で動かす｣という目標の
進捗記録用です。

## プロジェクトの特長
- **CORS回避**: nginx (ポート18080) をリバースプロキシとして設定し、`/api/` 経由でシステムAPI (ポート3100) にアクセス可能にしています。
- **自動接続ロジック実装済み**: `index.html` がサーバー上の最新プレイIDを自動取得し、接続用トークンを取得して起動します。
- **インフラ一括起動**: Docker Compose を使用して、マルチプレイに必要な全サーバーを即座に立ち上げられます。

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
