# akashic-agvw-multiplayer (進捗記録)

このプロジェクトは、`akashic-system` と `agvw` を組み合わせて、`rocket-game` をマルチプレイ（Passiveモード）でブラウザ起動させるための環境構築・実証の記録です。

## 現在の進捗状況 (2026/04/15)
結論から述べると、**「サーバー側でのゲーム起動」には成功しましたが、「ブラウザからのマルチプレイ参加」でエラーが発生しており、現在その原因を調査中**です。

### ◯ 達成できていること
*   **データベースの正常化**: オリジナル設定に含まれていた DB 名の不整合（`akashic` vs `akashic_dev`）を解消し、マイグレーションと API サーバーの参照先を統一しました。
*   **サーバー側インスタンスの起動**: `game-runner` を介して、サーバー内で `rocket-game` を Active モード（実行主体）で起動させることに成功しました（`status: running` を確認済み）。
*   **インフラの安定化**: Docker コンテナがホスト側の設定を正しく読み込めるようマウント設定を調整し、Redis 接続エラー等を排除しました。
*   **クライアント側ロジックの自動化**: `index.html` にて、API プロキシ（18080）を経由して「最新の Play ID 取得」および「プレイトークン発行」を行うフローを実装しました。

### ✕ 発生している問題
*   **セッション検証エラー (`failed to validate session`)**: ブラウザが `playlog-server` (4001) に接続を試みた際、トークンの検証に失敗し、ゲーム画面が表示されません。

### 原因に関する見立て
現在、以下のいずれかが原因であると推測し、検証を進めています。
1.  **トークン照合の不備**: `system-api-server` が発行したトークンを、`playlog-server` が参照できていない（保存先 Redis/DB の不一致）。
2.  **秘密鍵の不一致**: トークンの署名に使われる `permissionSecret` が、両サーバー間で同期されていない可能性。
3.  **検証ルートの通信エラー**: `playlog-server` から API サーバーへの検証問い合わせが、コンテナ間ネットワーク内でタイムアウトしている可能性。

---

## 最新の実行手順

環境を再現し、現在のエラー状態を確認するための手順です。

### 1. リソースの配置
デスクトップのオリジナル環境から、以下の構成でファイルを配置してください。
* `engines/` -> `volumes/akashic-resources/engines/`
* `rocket-game/` -> `volumes/akashic-resources/rocket-game/`
  * ※ `rocket-game/game.json` が直接見える階層にする必要があります。

### 2. インフラの起動
```bash
docker-compose up -d
```

### 3. データベースの初期化
ホスト側（PowerShell等）から実行します。
```bash
# database.json の host が localhost であることを確認してください
npm run db:migrate:mysql
npm run db:migrate:mongo
```

### 4. サーバー側ゲームセッションの開始 (PowerShell)
```powershell
# 1. Play の作成
$play = Invoke-RestMethod -Uri "http://localhost:3100/v1.0/plays" -Method Post -ContentType "application/json" -Body '{"gameCode": "rocket-game"}'
$playId = $play.data.id

# 2. インスタンスの起動
$body = @"
{
  "gameCode": "rocket-game",
  "entryPoint": "gsk/engines/dist/3.13.4-0/entry.js",
  "cost": 1,
  "modules": [
    { "code": "dynamicPlaylogWorker", "values": { "playId": "$playId", "executionMode": "active" } },
    { "code": "akashicEngineParameters", "values": { "gameConfigurations": ["gsk/rocket-game/game.json"], "verbose": true } }
  ]
}
"@
Invoke-RestMethod -Uri "http://localhost:3100/v1.0/instances" -Method Post -ContentType "application/json" -Body $body
```

### 5. ブラウザでの確認
以下の URL を開くと、現在のエラー状態（`failed to validate session`）を確認できます。
`http://localhost:18080/contents/index.html`
