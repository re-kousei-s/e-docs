# 3. リソース設計

| リソース | タグ | 主な利用者 | パス |
| --- | --- | --- | --- |
| auth: 認証 | Auth | 主催者 | `/auth/*` |
| users: 自分の情報（宛先） | Users | 主催者 | `/users/me/*` |
| events: イベント | Events | 主催者 | `/events`, `/events/{eventId}`, `/events/{eventId}/start` など |
| floors: フロア・混雑基準・人数の補正・スタッフ人数 | Floors | 主催者 | `/events/{eventId}/floors`, `/floors/{floorId}/*` |
| members: 共同主催者・招待 | Members | 主催者 | `/events/{eventId}/members`, `/events/{eventId}/invitations`, `/invitations/accept` |
| visitor-page: 来場者用画面の設定 | VisitorPage | 主催者 | `/events/{eventId}/visitor-page/*` |
| devices: 端末の追加・線・カメラ設定・認識（主催者側） | Devices | 主催者 | `/devices/*` |
| camera: スマホ側（端末の選択・設定・認識結果） | Camera | カメラ端末 | `/camera/devices/*` |
| crossings: 通過の履歴 | Crossings | 主催者 | `/events/{eventId}/crossings`, `/floors/{floorId}/crossings`, `/devices/{deviceId}/crossings` |
| staff: スタッフの入退室 | Staff | スタッフ | `/events/{eventId}/staff`, `/staff/me`, `/floors/{floorId}/staff/*` |
| public: 来場者ページ | Public | 来場者 | `/public/events/{eventId}` |
| rentals: 機材の貸与・返却 | Rentals | 主催者 | `/rentals/*` |
| realtime: WebSocket | Realtime | 主催者・カメラ端末・来場者 | `/ws/*` |
| system: ヘルスチェック・システムの状況・外部連携 | System | — / 主催者 | `/health`, `/ready`, `/webhooks/*`, `/system/status` |

## 方針

- カメラまわりは technical-verification（検証）の作りに合わせる。端末は管理画面でフロアを指定して追加し、スマホは一覧から選ぶ。スマホ側は認証なし（LAN の中で使う前提）。
- スマホが使う API は `/camera/*`、主催者が使う API は `/devices/*` に分ける。
- スマホとの WebSocket のメッセージは、検証のアプリをそのまま使えるよう、検証と同じ形（キーは snake_case）にする。
- 子リソースの更新・削除は、親を辿らずに ID だけで指せるパスにする（例: `/floors/{floorId}`）。作成と一覧は親の下（例: `/events/{eventId}/floors`）。
- 来場者向けは `/public/*` に分け、認証なしで `eventId` でアクセスする。
