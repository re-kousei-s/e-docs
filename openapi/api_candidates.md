# 2. API候補洗い出し

## 凡例

| 種別 | 意味 |
| --- | --- |
| HTTP | REST エンドポイント |
| WS | WebSocket（双方向・常時接続。サーバ→端末の指示やリアルタイム更新） |
| SSE | Server-Sent Events（サーバ→クライアントの一方向プッシュ。閲覧専用向き） |
| WH | Webhook 受信（外部サービス → こちら） |
| Stream | 映像ストリーム（プロトコルは要検討：WebRTC / RTMP / WS バイナリ など） |
| 内部 | サーバ内部ロジック。外に出さない |
| イベント | ドメインイベント。他の処理のトリガーになるだけで API ではない |
| 端末内 | カメラ端末アプリ内で完結 |

認証の種類

- 主催者: Google ログインのセッション
- カメラ端末: 紐付け時に発行する端末トークン
- 来場者: 認証なし（eventId でアクセス）
- スタッフ: スタッフ画面の初回アクセス時にスタッフ ID（トークン）を発行し、ローカルストレージに保存して、以降の入退室で `Authorization: Bearer` ヘッダで送る（ログインなし）

---

## ユーザ関連

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| sign up / sign in | HTTP | `GET /auth/google` → `GET /auth/google/callback` | Google OAuth。初回ログイン時にユーザ作成するので sign up と sign in は一本化 |
| sign out | HTTP | `POST /auth/logout` | |
| ログイン中アカウント取得 | HTTP | `GET /me` | 設定画面のアカウント表示用（ユースケースに追加） |
| 宛先情報の登録・変更 | HTTP | `GET /me/shipping-address` / `PUT /me/shipping-address` | 貸出ドメインに寄せるなら `/rentals/shipping-address` |

## イベント

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| イベント一覧取得 | HTTP | `GET /events` | 自分が主催・共同主催のもの。状態（開始前/開催中/終了）付き |
| イベント詳細取得 | HTTP | `GET /events/{eventId}` | フロア一覧を含める |
| イベント作成 | HTTP | `POST /events` | body にフロア配列を含めて「1つ以上」をバリデーション |
| イベント名編集 | HTTP | `PATCH /events/{eventId}` | |
| イベント開始 | HTTP + イベント | `POST /events/{eventId}/start` | body: カメラ終了用パスワード（任意）。未設置フロアは事前に警告するなら `GET` 側で返すか、`?dryRun` 的に確認。`EventStarted` を発行 → カメラ端末へ WS で観測開始 |
| イベント終了 | HTTP + イベント | `POST /events/{eventId}/end` | `EventEnded` を発行 → 端末の紐付け解除・観測停止 |
| 混雑状況の初期化 | — | `POST /events/{eventId}/end` に統合 | **イベント終了に統合で決定**。専用の API は作らない |

## フロア

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| フロア追加 | HTTP | `POST /events/{eventId}/floors` | |
| フロア名変更 | HTTP | `PATCH /floors/{floorId}` | |
| フロア並び替え | HTTP | `PUT /events/{eventId}/floors/order` | body: floorId の配列 |
| フロア削除 | HTTP | `DELETE /floors/{floorId}` | 端末が割り当てられているときの扱いを決める |

## 共同主催者

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| 招待 | HTTP | `POST /events/{eventId}/invitations` | 招待コード発行 or メールアドレス指定。メール送信自体は内部 |
| 招待の受諾 | HTTP | `POST /invitations/accept` | body: 招待コード。**ユースケースに無いが必要** |
| 招待中ゲスト一覧取得 | HTTP | `GET /events/{eventId}/members` | 画面はイベント横断なので `GET /members` でまとめて返す案も |
| ゲスト追放 | HTTP | `DELETE /events/{eventId}/members/{userId}` | |

## 端末管理（主催者側）

technical-verification（検証）の作りに合わせる。

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| 端末の追加 | HTTP | `POST /devices` | **検証どおり**。フロアと名前を指定して追加し、端末 ID を発行する。QR での紐付けはやめた |
| 端末一覧・詳細取得 | HTTP | `GET /devices` / `GET /devices/{deviceId}` | 詳細はカメラの能力（caps）・カメラの設定・線ごとの IN/OUT・受信の数字を含む |
| 端末の削除 | HTTP | `DELETE /devices/{deviceId}` | スマホが接続中なら切断する。通過の記録は残る |
| 線の編集 | HTTP | `PUT /devices/{deviceId}/lines` | **検証どおり**。管理画面で編集。1 台に名前付きで 1〜12 本、IN/OUT の反転あり。保存するとスマホにも届く |
| カメラの設定 | HTTP | `PUT /devices/{deviceId}/camera-settings` | 解像度・ズーム・レンズ。スマホが報告した caps から選ぶ |
| 認識の開始・停止 | HTTP | `PUT /devices/{deviceId}/recognize` | 端末ごと。イベントの開始・終了でも一括で開始・停止する（両方） |
| 検出枠つき画像 | HTTP | `GET /devices/{deviceId}/frame.jpg` | worker が描いた最新の静止画 |

## カメラ端末（スマホ）

認証なし（検証どおり。LAN の中で使う前提）。

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| 自分の端末を選ぶ | HTTP | `GET /camera/devices` | 全イベント・全主催者の端末を、フロアごとに返す |
| 設定と状況の取得 | HTTP | `GET /camera/devices/{deviceId}` | 2 秒ごとに読む |
| 認識結果の取得 | HTTP | `GET /camera/devices/{deviceId}/result` | 映像に検出枠を重ねる |
| サーバとの常時接続 | WS | `WS /ws/devices/{deviceId}` | 検証の `/ws/:id` と同じ約束事。スマホ → H.264 の 1 フレーム（バイナリ）と `{"caps"}`、サーバ → 状態 `{"recognize","send_fps","preview","idr","camera","lines"}` |

## 監視・配信

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| 映像の送信 | WS | `WS /ws/devices/{deviceId}` | H.264（Annex B）を 1 フレームずつ。認識中 5fps、ライブ表示中 15fps、サムネイル 1fps、それ以外は送らない |
| 映像を見る | WS | `WS /ws/devices/{deviceId}/live`（`?thumb=1` でサムネイル） | **検証どおり**。H.264 をデコードせずに中継し、認識結果も同じ WS で送る |
| マスク処理 | 端末内 | — | 追加機能扱い |

## 検知

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| 出入り検知 | 内部 | detector サービス（Python） | 映像を受けてライン通過を判定する。入室・退室の向きも detector が決める |
| フロア人数の更新 | 内部 | Python の detector が人数テーブルを直接更新 | **決定**。API は通さない。外部には公開しない |

## 混雑判定

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| 混雑基準の設定 | HTTP + イベント | `PUT /floors/{floorId}/congestion-thresholds` | `ThresholdsChanged` → 再判定 |
| 混雑判定 | 内部 + イベント | `CongestionUpdated` | 人数変化・基準変更・スタッフ人数変更をトリガーに再判定 |
| 混雑状況の手動修正 | HTTP | `PUT /floors/{floorId}/count` | **検証どおり（人数を補正）で決定**。区分の上書きはやめた。補正すると概算の範囲も 0 から数え直す |
| フロア別状況取得 | HTTP + WS | `GET /events/{eventId}/dashboard` + `WS /ws/events/{eventId}` | 初期表示は HTTP、以降の更新は WS で配信 |

## スタッフ配置

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| スタッフ人数の登録・更新 | HTTP + イベント | `PUT /floors/{floorId}/staff-count` | `StaffCountChanged` → 再判定 |
| スタッフ登録 | HTTP | `POST /events/{eventId}/staff` | スタッフ画面の初回アクセス時に呼び、スタッフ ID（トークン）を返す。クライアントはローカルストレージに保存 |
| フロア入室 / 退室 | HTTP + イベント | `POST /floors/{floorId}/staff/enter` / `POST /floors/{floorId}/staff/exit` | スタッフ画面の入室・退室ボタン。保存したスタッフ ID で個人を識別し、「まだそのフロアにいない」などを判定。`StaffCountChanged` → 再判定 |
| 滞在フロア取得 | HTTP | `GET /staff/me` | スタッフ画面を開き直したときに現在の滞在フロアとボタン状態を復元する |

## 来場者ページ

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| 混雑状況取得 | HTTP | `GET /public/events/{eventId}` | 認証なし。画面設定（マップ・スライドショー）もまとめて返すと 1 リクエストで済む |
| 来場者用画面設定の取得 | HTTP | `GET /events/{eventId}/visitor-page` | 主催者の編集画面用 |
| 来場者用画面設定の更新 | HTTP | `PUT /events/{eventId}/visitor-page` + `POST /events/{eventId}/visitor-page/images` | 画像は multipart か署名付き URL |
| リンク・QR発行 | HTTP | `GET /events/{eventId}/visitor-page` に公開 URL を含める | QR 画像はフロントで生成すれば専用 API 不要 |
| リアルタイム反映 | WS | `WS /ws/public/events/{eventId}` | **WebSocket で決定**。認証なし・受信専用 |

## 貸出

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| 貸与機材・プラン一覧取得 | HTTP | `GET /rentals/catalog` | 機材と単価、プランをまとめて返す |
| 貸与申請 | HTTP | `POST /rentals` | 決済ページの URL を返す |
| 決済 | 外部 + WH | `POST /webhooks/payment` | 決済結果を受けて申請を確定 |
| 申請完了メール送信 | 内部 | `RentalConfirmed` → メール送信 | |
| 貸与履歴取得 | HTTP | `GET /rentals` | 状態：貸与中 / 返却済み |
| 在庫引当・発送 | 内部 / 外部 | — | 外部サービス想定。運営側の作業 |

## 返却

| ユースケース | 種別 | 候補 | メモ |
| --- | --- | --- | --- |
| 返却物の発送登録 | HTTP | `POST /rentals/{rentalId}/return` | 伝票番号など |
| 返却受理 | 運営側 / WH | 運営用の管理 API か、外部サービスからの WH | 主催者向け API ではない |

---

## 決定事項

1. **スタッフの認証方式**: 別画面（スタッフ画面）で入室・退室ボタンを押す。スタッフ ID はローカルストレージに保存し、ヘッダで送る（ログインなし）。登録は認証なしで、スタッフ画面のリンクを知っていれば誰でもできる。
2. **カメラ端末との常時接続**: WebSocket（`WS /ws/devices/{deviceId}`）。検証と同じ約束事（13 を参照）。
3. **手動修正の対象**: 人数を補正する（検証どおり）。区分の上書きはやめた。
4. **ダッシュボード WebSocket の認証**: 接続は認証なしで受け付け、接続後の最初のメッセージ（`type: auth`）でアクセストークンを送る。
5. **来場者ページの更新**: WebSocket（`/ws/public/events/{eventId}`）。認証なし・受信専用。
6. **来場者ページの ID**: eventId をそのまま使う。公開用の ID は発行しない。
7. **混雑状況の初期化**: イベント終了に統合する。終了時に全フロアの人数を 0 に戻す。
8. **出入りの検知**: Python の detector が検知し、人数テーブルも直接更新する。ラインの向きの判定も detector 側。API はラインの座標を保存するだけ。
9. **画像アップロード**: PNG / JPEG / WebP、1 ファイル 2 MB まで。
10. **カメラのプレビュー**: 検証どおり。端末のページはライブ 15fps、一覧のサムネイルは 1fps。H.264 を中継し、認識結果も同じ WS で送る（`/ws/devices/{deviceId}/live`）。認証は最初のメッセージ。
11. **端末の登録**: 検証どおり。管理画面でフロアと名前を指定して追加し、スマホは全端末の一覧から選ぶ。スマホ側は認証なし。
12. **認識の開始・停止**: イベントの開始・終了で一括開始・停止し、途中で端末ごとにも開始・停止できる。
13. **線**: 検証どおり。管理画面で編集し、1 台に名前付きで 1〜12 本。保存するとスマホにも届く。
14. **人数の補正**: 検証どおり。スタッフが数えた人数を設定し、そこから数え直す。概算の範囲（補正後の通過件数 × 誤差率）も返す。
15. **カメラの設定**: 解像度・ズーム・レンズを管理画面から指定できる。
16. **追加した機能**: 通過の履歴、履歴のリセット（イベント・フロア・端末）、検出枠つき画像、システムの状況。

## 保留

1. **決済サービス**: 未定。決まるまで決済 Webhook の形式は仮。
2. **本番ドメイン**: 未定。仮の `api.example.com` を使う。
3. **フロア削除時の端末**: カメラ端末が残っているフロアを削除したときの扱い。
4. **バッテリー残量**: スマホから送るか（検証のスマホは送っていない）。
