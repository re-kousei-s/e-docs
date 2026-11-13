# e-sharp-docs

E展 混雑可視化システムの設計ドキュメントと OpenAPI を置くリポジトリ。Issue もここに紐づける。

## 構成

```text
e-sharp-docs/
├── README.md                       このファイル。リポジトリ全体の案内
├── Taskfile.yml                    lint・バンドル・モック／ドキュメントサーバの起動などのコマンド集
├── redocly.yaml                    OpenAPI の lint ルール
├── .gitignore                      compose/.env など、各自の環境の設定をコミットしないための設定
├── compose/                        Docker でモックサーバ・ドキュメントサーバを動かす設定
└── openapi/
    ├── approach.md                 設計の進め方と、ファイルを書くときのルール
    ├── usecase_analysis.md         1. ユースケース整理
    ├── api_candidates.md           2. API 候補の洗い出しと、決定事項・保留事項
    ├── resources.md                3. リソース設計（エンドポイントのグループ分け）
    ├── openapi.yaml                OpenAPI のエントリポイント
    ├── paths/                      エンドポイントの定義（リソースごとに 1 ファイル）
    │   ├── auth.yaml               認証
    │   ├── users.yaml              自分の情報（宛先）
    │   ├── events.yaml             イベント・ダッシュボード
    │   ├── floors.yaml             フロア・混雑設定・人数の補正
    │   ├── members.yaml            共同主催者・招待
    │   ├── visitor-page.yaml       来場者用画面の設定
    │   ├── devices.yaml            端末の追加・線・カメラ設定・認識の開始/停止（主催者側）
    │   ├── camera.yaml             スマホ側（端末の選択・設定・認識結果）
    │   ├── crossings.yaml          通過の履歴とリセット
    │   ├── staff.yaml              スタッフの入退室
    │   ├── public.yaml             来場者ページ（認証なし）
    │   ├── rentals.yaml            機材の貸与・返却
    │   ├── realtime.yaml           WebSocket
    │   └── system.yaml             ヘルスチェック・システムの状況・Webhook
    └── components/                 複数のエンドポイントで共有する部品
        ├── schemas.yaml            共通モデル
        ├── parameters.yaml         パスパラメータ
        ├── responses.yaml          エラーレスポンス
        └── messages.yaml           WebSocket のメッセージ
```

## openapi/

### 設計ドキュメント（Markdown）

上から順に進めた設計の記録。OpenAPI を変えるときは、前提になっているこれらも合わせて直す。

| ファイル | 内容 | いつ見るか |
| --- | --- | --- |
| `approach.md` | 設計の手順（ユースケース → API 候補 → リソース → OpenAPI）と、ファイル分けのルール、WebSocket の書き方 | 初めて触るとき、ファイルを追加するとき |
| `usecase_analysis.md` | ドメイン（イベント・フロア・カメラ端末・スタッフ・来場者・貸出など）ごとのユースケース一覧 | 機能の抜け漏れを確認するとき |
| `api_candidates.md` | 各ユースケースを HTTP / WebSocket / Webhook / 内部処理に仕分けた表。末尾に決定事項と保留事項 | ある機能がなぜ API になっている（なっていない）のかを知りたいとき |
| `resources.md` | リソースとタグ・利用者・パスの対応表と、パス設計の方針 | 新しいエンドポイントをどこに置くか迷ったとき |

### `openapi.yaml`

OpenAPI のエントリポイント。API の概要・接続先・タグ・認証方式と、全エンドポイントの一覧だけを書く。
エンドポイントの中身は書かず、`./paths/<ファイル>.yaml#/<キー>` を参照する。

認証方式は 2 つで、既定は `OrganizerAuth`。それ以外を使うエンドポイントは各 paths で `security` を上書きする。

| 認証方式 | 利用者 | トークンの発行元 |
| --- | --- | --- |
| `OrganizerAuth` | 主催者・共同主催者 | `POST /auth/google` |
| `StaffAuth` | スタッフ | `POST /events/{eventId}/staff` |
| なし（`security: []`） | カメラ端末（スマホ）・来場者・外部サービス | — |

### `openapi/paths/`

エンドポイントの定義。ファイル 1 つがタグ（リソース）1 つに対応する。
1 つのエンドポイントでしか使わないリクエスト・レスポンスのボディは、ここに直接書く。

各ファイルのキーはパスから `/` と `{}` を除いたもの（例: `/events/{eventId}/start` → `events_eventId_start`）。
ファイル先頭のコメントに、パスとキーの対応表がある。

| ファイル | タグ | 利用者 | 入っているパス |
| --- | --- | --- | --- |
| `auth.yaml` | Auth | 主催者 | `/auth/google` ログイン、`/auth/refresh` トークン更新、`/auth/signout` ログアウト、`/auth/me` ログイン中ユーザ |
| `users.yaml` | Users | 主催者 | `/users/me/shipping-address` 貸与機材の送付先 |
| `events.yaml` | Events | 主催者 | `/events` 一覧・作成、`/events/{eventId}` 詳細・編集、`/start` 開始、`/end` 終了（人数も 0 に戻す）、`/dashboard` ダッシュボード |
| `floors.yaml` | Floors | 主催者 | `/events/{eventId}/floors` 追加、`/floors/order` 並び替え、`/floors/{floorId}` 名前変更・削除、`/congestion-thresholds` 混雑基準、`/count` 人数の補正、`/staff-count` スタッフ人数 |
| `members.yaml` | Members | 主催者 | `/events/{eventId}/members` メンバー一覧・除外、`/events/{eventId}/invitations` 招待、`/invitations/accept` 招待の承諾 |
| `visitor-page.yaml` | VisitorPage | 主催者 | `/events/{eventId}/visitor-page` 来場者用画面の設定、`/images` 画像アップロード |
| `devices.yaml` | Devices | 主催者 | `/devices` 一覧・追加（フロアを指定）、`/devices/{deviceId}` 詳細・削除、`/lines` 線の保存、`/camera-settings` カメラの設定、`/recognize` 認識の開始・停止、`/frame.jpg` 検出枠つき画像 |
| `camera.yaml` | Camera | カメラ端末（スマホ） | `/camera/devices` 端末の一覧（端末選択）、`/camera/devices/{deviceId}` 設定と状況、`/result` 最新の認識結果。認証なし |
| `crossings.yaml` | Crossings | 主催者 | `/events/{eventId}/crossings` 通過の履歴・イベント単位のリセット、`/floors/{floorId}/crossings`・`/devices/{deviceId}/crossings` フロア・端末単位のリセット |
| `staff.yaml` | Staff | スタッフ | `/events/{eventId}/staff` 登録、`/staff/me` 状態取得、`/floors/{floorId}/staff/enter`・`/exit` 入室・退室 |
| `public.yaml` | Public | 来場者 | `/public/events/{eventId}` 来場者ページの初期表示 |
| `rentals.yaml` | Rentals | 主催者 | `/rentals/catalog` 機材とプラン、`/rentals` 履歴・申請、`/rentals/{rentalId}/return` 返却発送 |
| `realtime.yaml` | Realtime | カメラ端末・主催者・来場者 | `/ws/devices/{deviceId}` スマホとの接続（映像の受信・状態の指示）、`/ws/devices/{deviceId}/live` 端末の映像（ライブ 15fps / サムネイル 1fps）、`/ws/events/{eventId}` ダッシュボードの更新通知、`/ws/public/events/{eventId}` 来場者ページの更新通知（WebSocket。接続だけを定義） |
| `system.yaml` | System | インフラ・外部サービス・主催者 | `/health`・`/ready` ヘルスチェック、`/webhooks/payment` 決済結果の受信（この 3 つは `/v1` なしの URL）、`/system/status` システムの状況 |

### `openapi/components/`

複数のエンドポイントで共有する部品。種類ごとに 1 ファイル。

| ファイル | 内容 | 参照の書き方（paths から） |
| --- | --- | --- |
| `schemas.yaml` | 2 か所以上で使うモデル。区切りコメントで「列挙型」「プロパティ」「オブジェクト」「リクエストボディ」「レスポンスボディ」「エラー」に分かれている | `../components/schemas.yaml#/Floor` |
| `parameters.yaml` | `{eventId}` `{floorId}` などのパスパラメータ | `../components/parameters.yaml#/EventId` |
| `responses.yaml` | 400〜502（413・415 を含む）のエラーレスポンス（本文は Problem Details）。paths 側で `description` を上書きしてエラー理由を書く | `../components/responses.yaml#/Conflict` |
| `messages.yaml` | WebSocket でやり取りするメッセージ。OpenAPI では表現できないので、ここに JSON Schema で置き、paths の description から名前で参照する | description に「`components/messages.yaml` の `CameraServerMessage`」のように書く |

## compose/

Docker で OpenAPI をバンドルし、モックサーバとドキュメントサーバを動かす設定。`Taskfile.yml` から呼ぶ。

| ファイル | 内容 |
| --- | --- |
| `compose.bundler.yaml` | Redocly CLI で `openapi/` を 1 ファイル（`openapi.bundled.yaml`）にまとめ、`e-sharp-docs_dist` ボリュームに出力する。lint もこのコンテナで実行する |
| `compose.services.yaml` | バンドル結果を使うサーバ。`mock`（Prism のモックサーバ）と `docs`（nginx + Scalar のドキュメント） |
| `compose.yaml` | 上の 2 つをつなぐ設定。バンドルが終わってからサーバを起動する |
| `.env.example` | ポート番号の例（`MOCK_PORT` / `DOCS_PORT`）。`compose/.env` にコピーして使う。`.env` はコミットしない |
| `docs/index.html` | ドキュメントサーバのページ（Scalar）。「試す」の送り先はモックサーバ（`http://localhost:4010`） |
| `docs/default.conf` | ドキュメントサーバの nginx 設定 |

## ルート直下のファイル

| ファイル | 内容 |
| --- | --- |
| `Taskfile.yml` | よく使うコマンド。下の「動かし方」を参照 |
| `redocly.yaml` | lint ルール。`recommended` をベースに、4xx 必須ルールと `example.com` 禁止ルールを外している（本番ドメインが決まったら後者は戻す） |
| `.gitignore` | `.env` と `.env.*` を除外し、`.env.example` だけはコミットする |

## 動かし方

ホスト側に Docker（Docker Desktop）が必要。Task（`winget install Task.Task`）は任意で、無ければ下の `docker compose` を直接打つ。

```sh
# 初回だけ: ポートの設定を作る（Windows の PowerShell なら Copy-Item compose\.env.example compose\.env）
cp compose/.env.example compose/.env

task openapi:up      # バンドル → モックサーバ・ドキュメントサーバを起動
task openapi:down    # 停止
task openapi:lint    # lint だけ
```

Task が無い場合（`openapi:up` と同じ）:

```sh
docker compose -f compose/compose.bundler.yaml -f compose/compose.services.yaml -f compose/compose.yaml up -d
```

| コマンド | 内容 |
| --- | --- |
| `task openapi:lint` | lint だけ実行する |
| `task openapi:bundle` | 1 ファイルにバンドルする |
| `task openapi:up` | バンドルしてモックサーバとドキュメントサーバを起動する |
| `task openapi:srv:up` / `openapi:srv:down` | バンドル済みの状態でサーバだけ起動・停止する |
| `task openapi:down` | すべて停止する |
| `task openapi:reset` | ボリュームを消して起動し直す |
| `task openapi:rm-dist` | バンドル結果のボリューム（`e-sharp-docs_dist`）だけ消す |

起動したら:

- ドキュメント: `http://localhost:<DOCS_PORT>`（`.env` の値。例 8080）
- モック: `http://localhost:<MOCK_PORT>`（例 4010）。パスの先頭に `/v1` は付けない。認証が要る API は `Authorization: Bearer <何でもよい>` を付ける
- ポートが使用中（`port is already allocated`）なら、`compose/.env` の値を変えて起動し直す

## 未決事項（TODO）と現状の方式

仕様の中で `TODO:` と書いてある箇所の一覧。どれも「現状:」の方式で仮に定義してあり、決まったら該当ファイルを直してこの表から消す。

| 項目 | 場所 | 決めること | 現状の方式 |
| --- | --- | --- | --- |
| フロア削除時の端末 | `paths/floors.yaml` `DELETE /floors/{floorId}` | 端末が残っているフロアを削除したときの扱い | 未定義 |
| バッテリー残量 | `components/schemas.yaml` `Device.batteryLevel`、`components/messages.yaml` `CameraClientMessage` | スマホからバッテリー残量を送るか | 項目だけ定義。検証のスマホは送っていない |
| 決済 Webhook の形式 | `paths/system.yaml` `POST /webhooks/payment` | 決済サービスに合わせた署名ヘッダとボディ | ボディは任意の JSON オブジェクト、署名ヘッダは未定義。検証失敗で 400、受理で 204 |
| 本番ドメイン | `openapi.yaml` `servers`、`paths/realtime.yaml`、`paths/system.yaml`、`components/schemas.yaml` `ProblemDetail.type`、`redocly.yaml` | 本番の API ドメイン | 仮の `api.example.com`。ProblemDetail の type にドメイン制約なし。lint の example.com 警告は止めている |
