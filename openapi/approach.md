# エンドポイント定義のアプローチ

[crowdodge-docs](https://github.com/crowdodge/crowdodge-docs) の進め方に合わせる。

1. ユースケース整理 → [usecase_analysis.md](./usecase_analysis.md)
2. API候補洗い出し（HTTP / WS / SSE / Webhook / 内部ロジック・イベントの仕分け） → [api_candidates.md](./api_candidates.md)
3. リソース設計（エンドポイントグループ） → [resources.md](./resources.md)

---

ここから OpenAPI（[openapi.yaml](./openapi.yaml)）。

1. エンドポイント一覧: `openapi.yaml` に各パスを書き、中身は `paths/` のリソースごとのファイルに書く
2. エンドポイント詳細: メソッド・タグ・summary・description・operationId・パス/クエリパラメータ
3. 認可: 誰が叩けるか。`security` で表現する（主催者 / カメラ端末 / スタッフ / 認証なし）
4. エラー形式: RFC 9457 の Problem Details（`application/problem+json`）
5. Req / Res 形式（DTO）: 共通のものは `components/schemas.yaml`、それ以外は `paths/` 内
6. examples
7. components 共通化

## ファイル構成

細かく分けすぎると追いにくいので、エンドポイントはリソース（タグ）ごとに 1 ファイル、共通部品は種類ごとに 1 ファイルにまとめる。
各ファイルの中身の詳しい説明はリポジトリ直下の [README.md](../README.md) と、各 YAML ファイル先頭のコメントを参照。

```text
openapi/
  openapi.yaml            # エンドポイント一覧・タグ・認証方式
  paths/                  # リソースごとに 1 ファイル（auth / events / floors / camera …）
  components/
    schemas.yaml          # 複数のエンドポイントで使うモデル（列挙型・オブジェクト・エラー）
    parameters.yaml       # パスパラメータ
    responses.yaml        # エラーレスポンス
    messages.yaml         # WebSocket のメッセージ
```

- 1 つのエンドポイントでしか使わないリクエスト・レスポンスのボディは、`paths/` の中にそのまま書く
- 2 か所以上で使うものだけ `components/schemas.yaml` に出す
- `paths/*.yaml` のキーはパスから `/` と `{}` を除いたもの（例: `/events/{eventId}/start` → `events_eventId_start`）。各キーの上にコメントで元のパスを書く

## WebSocket の書き方

OpenAPI は WebSocket のメッセージを表現できないので、次のように書く。

- 接続（ハンドシェイク）は `GET` のパスとして定義し、`101 Switching Protocols` を返す
- やり取りするメッセージは `components/messages.yaml` に定義し、description からリンクする
- ブラウザから接続する WebSocket は Authorization ヘッダを付けられないので、認証が必要なものは接続後の最初のメッセージ（`type: auth`）でアクセストークンを送る
