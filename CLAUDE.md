# CLAUDE.md — 引き継ぎメモ

**こっぺナビ**: 原付（50cc / 125cc）向けのツーリング用 Web アプリ（PWA）。今後、徒歩のナビと心霊スポットの表示を足す予定（足したら公開し直す）。ユーザーは日本語で話す。返事も日本語で、専門用語はかみくだいて説明する。

- 公開版: https://koppepandayooo.github.io/koppe-navigator/（リポジトリ `koppepandayooo/koppe-navigator`。アプリの表示名は「こっぺナビ」）
- 前のリポジトリ（gentsuki-touring）は、PR マージの記録にメールアドレスが残っていたので消して、履歴なしの 1 コミットで作り直した。**GitHub のサイトで PR をマージするときは、コミットに使うメールアドレスが `…@users.noreply.github.com` になっているか確認する**
- `main` に push すると GitHub Actions（`.github/workflows/pages.yml`）が `web/` を GitHub Pages に公開する（1 分ほど）
- ビルドなし・ライブラリのインストールなし。素の HTML / CSS / JS（MapLibre GL だけ CDN から読む）

## ファイル

| パス | 中身 |
|---|---|
| `web/index.html` | 画面の骨組み。CSS/JS は `?v=数字` でキャッシュよけ（変更したら数字を上げる） |
| `web/app.js` | 全部のロジック（約 1300 行。`// ===== 見出し =====` で区切ってある） |
| `web/style.css` | 見た目。ダークは `:root[data-theme=dark]`（JS がテーマを決めて付ける） |
| `web/sw.js` | Service Worker。アプリ本体はネット優先・オフライン時だけキャッシュ |
| `web/data/regulations.json` | 二輪通行規制区間（日本二輪車普及安全協会のデータ、476 件） |
| `web/voice/phrases.json` / `web/voice/zundamon/` | ずんだもんの決まり文句と音声（MP3 と `index.json`）。`tools/make_voice.py` が VOICEVOX エンジン（127.0.0.1:50021、話者 3 = ずんだもんノーマル）で作る |
| `tools/scrape_jmpsa.py` | 規制区間データの取得。協会は毎年 6 月に更新するので、その後に再実行 |

ローカルで動かす: `python -m http.server 8810 --directory web` → http://localhost:8810

## 使っている外部サービス（全部無料・キー不要）

- ルート: Valhalla 公開サーバー `valhalla1.openstreetmap.de`（`motor_scooter` コスティング）。同時 2 本まで・失敗したら再試行（`valhalla()`）。重くしすぎない
  - `/trace_attributes` で車線数・トンネル/橋・道路の種類を取る（結果はキャッシュ）
- 地図: OpenFreeMap の `liberty` スタイルを読み込んで加工（日本語化・店アイコンと 3D 建物を消す・道路番号標識を間引く）。夜用は `nightify()` で色を差し替えた自作配色（既製の dark は真っ黒すぎて不評だった）
- 検索: Photon（候補）＋ 国土地理院 住所検索 ＋ Nominatim（見つからないときと逆ジオコーディング、1 秒 1 回まで）
  - ひらがな・カタカナだけの入力は `kanaToKanji()` で漢字にして（Google の `www.google.com/transliterate`、CORS 可・非公式なので使えなければ空を返す）一緒に探す。Photon も Nominatim も読みがなでは引けないため
- 道路の通称: Nominatim の `namedetails` から `alt_name` を取って端末に保存（`requestAlias()`）。手書きの表はない
- ストリートビュー: Google マップの URL を開くだけ（API キー不要）。アプリ内埋め込みはキーが要るのでやっていない

## 原付向けの機能（このアプリの売り）

1. **二輪通行規制区間**: 協会の詳細ページに埋め込まれた Google マップの `q=緯度,経度` から始点・終点の座標を取得。座標がない行は住所から推定（`approx: true`、判定には使わず地図表示だけ）
   - 通過判定 `findRegHits()`: 始点→終点の順でルートが 35m 以内を通るか。上下線が別登録（`hasTwin`）なら向きも見る。地下道・陸橋の規制で、ルートの該当区間がトンネル/橋でなければ「横の側道」とみなす
   - 自動回避: Valhalla の `exclude_polygons`。外周合計 10km の上限があるので長い区間は入口・出口だけふさぐ（`regBlockPolygons()`）
   - 対象判定 `regApplies()`: 一種は「一般原付」、二種は「二輪」
2. **二段階右折**: 片側 3 車線以上（OSM の lane_count）の道路からの右折を「可能性」として表示。避けるときは、まず普通のルートを出し、裏で「大通りを避けた案」と「右折先の道の入口をふさいだ案」を探して、少なければ差し替える（1 回 = 2 分の減点で比較）
3. **入れない目的地**: 歩行者用の道（`noVehicle`）や、手前が長い管理用道路・林道（`serviceTail > 0.7km` → `publicOnly`）は一般道の終わりで原付を停めて、そこから Valhalla の `pedestrian` で歩きのルートを出す（`walkRoute()`、見つからなければ直線）。地図に点線（`walk` レイヤー）、時間は原付＋徒歩、ナビは停める所（`markParking()` で最後の案内を差し替え）に着いたら `nav.walking` にして歩きの案内に切り替える（歩きは再検索しない）。歩きのときだけ店・寺社・公園などの名前とアイコン（スタイルの `poi_r…` レイヤー、普段は `visibility: none`）を `setPoiVisible()` で出し、地図を平ら・ズーム 17 にする（目印がなくて見にくい、と言われたため）。Valhalla が「No path could be found」を返したとき（最寄りの道が原付の道とつながっていない。例: 史跡や公園の中の砂利道）は、まず目的地・経由地、次に出発地を `publicOnly`（`min_road_class: residential`）にして探し直す。理由は `walkReason` で表示
   - **区間ごとの行き方**: 経由地・目的地の入力欄の左の `.mode` ボタン（🛵 / 🚶、`data-mode`）。`tripPlan()` が `ride`（原付で行く地点、最後が停める所）と `walk`（その後歩く地点）に分け、`planRoute()` は原付の部分だけを Valhalla に、`walkRoute()` は `walkPlan` の地点を通る歩きのルートを出す（複数区間なら `nav.wLeg` で順に案内）。歩きの後にまた原付、出発地から歩き、はエラー表示（未対応）。QR には `md` で行き方を入れる
   - **バイク駐車場**: 「🅿 駐車場」で `findParking()` が Overpass から目的地の 2km 四方（`[bbox:]`、`around` より速い）の `amenity=motorcycle_parking`・`parking` で `motorcycle=yes`・`bicycle_parking` で `moped/motorcycle=yes` を取り、800m 以内があればそれだけ、なければ 2km 以内を出す。普通の駐輪場（原付可か不明）は出さない（ユーザーの判断）。学校・大学・幼稚園の敷地の中（Overpass の `map_to_area` で判定し、`out ids` で番号だけ返す）、`access=private/no` など関係者用、名前や運営者に「大学」「高校」などが入るもの（`staffOnly()`）は外す。`access=customers` は「お客さん用」と表示。保存した結果のキーは `v2:` 始まり（学校を外す前の結果を捨てるため）。Overpass は混むと何を聞いても 30 秒前後かかる（小さな問い合わせや状態ページでも。待てば返ってくる）。予備の kumi / private.coffee はエラーを返すことが多い。なので 1 サーバー 60 秒まで待つ（10 秒で切ったら全部失敗した。学校の判定を入れてから 40 秒以上かかることもある）。待つ間は Photon の reverse（名前のある駐車場だけ）を先に出す。結果は端末に 1 週間保存（`parkStore`）し、歩く部分があるルートを出したら裏で先に探す。選ぶと `places.park` になり、原付のルートの終点がそこになる
4. **パソコンとスマホ**: ナビ開始ボタンはタッチ端末だけ（CSS の `(hover: hover) and (pointer: fine)` で隠す）。パソコンの現在地は誤差が大きい（500m 超）ので出発地に使わない
5. **ナビ**: 進行方向が上・斜めから見た地図、日本語音声（Valhalla の案内文からローマ字と道路番号を消し、通称に置き換える）、ルート外れ（50m 以上が 3 回）で再検索。再検索では走っている向きを `heading` で渡して、いきなり U ターンさせない。電波がない（`navigator.onLine` が false）ときは再検索を止めて元のルートで案内し、失敗したら 15 秒→30 秒…と間をあける（`online` で再開）。Valhalla が「道が見つからない」と答えた失敗（`planRoute.lastError.fatal`）は電波のせいにせず、別の案内にする。出発地を入力したときはそのルートのまま、ルートに乗るまで「出発地点まで」の表示で待つ
6. **ずんだもんの声**: `speak(text, ids)`。ids の音声が全部あればずんだもんの音声をつないで流し（Web Audio）、なければ text を端末の声で読む。道路名などは入れない（決まり文句だけ）。音を出す準備 `unlockVoice()` はナビ開始のタップ直後（await より前）に呼ぶ。設定画面に「VOICEVOX:ずんだもん」の表記が必要（利用規約）
   - クラウドの作業環境でも VOICEVOX エンジンは動かせる: GitHub のリリース（`voicevox_engine-linux-cpu-x64-<版>.7z.001`）は直接ダウンロードでき、py7zr で展開して `./run` で起動
7. **スマホに送る**: 地点と設定を URL の `#r=` に詰めて（deflate＋base64url）QR コードに。スマホは `receiveShared()` で読んで同じ条件で再検索し、距離と曲がる回数が近い候補を選ぶ。QR は qrcode-generator を CDN から必要なときだけ読む
8. **大通りを避ける**（設定「大通り: なるべく避ける」= `use_primary` 0.15 以下、初期設定）: Valhalla の `use_primary` は国道クラス（trunk/primary）にしか効かず、県道クラスや少し速いだけの大通りを選んでしまう（「大通りを通らなくていい所を通る」と不評だった）。なので `{ use_primary: 0.02 }` と `{ shortest: true }`（距離最短は細い道を通りやすい）の案も出し、`/trace_attributes` の `road_class` で大通りの距離（`bigKm.main` 国道クラス・`bigKm.sub` 県道クラス）を数えて `bigRoadPenalty()`（150 秒/km・90 秒/km）を点数に足す。候補ボタンに「大通り ○km」
9. **曲がる回数**: `TURNS`（右左折・U ターン）を数え、設定 `turns`（秒/回）を点数に足す。0 以外なら `use_primary: 0.9, maneuver_penalty: 120` の案も出す（Valhalla の設定だけでは回数はほぼ減らない）

## ユーザーの好み・これまでの判断

- 見た目は Google マップっぽく、すっきり。ごちゃごちゃした地図は嫌い
- 地図データ（OSM）の名前が実際と違うことがある → マイスポット・長押しで道の呼び名変更で対応。手で書いた補正データ（以前の `spots.json` / `road-names.json`）は「1 か所だけ直すのは意味が分からない」と言われて消した。手書きの場当たり的な補正は入れず、地図データなど外から取れるものを使う
- ナビはスマホでしか使わない。パソコンはルートを調べる用
- 「走ったルートをアカウントで保存」は後回しと言われている

## 次にやりたいこと（候補）

- 走行ログの保存（アカウント・サーバーが要る。後回し）
- 原付二種で「原付通行禁止（moped=no）」の道も通れるようにする（今は一種と同じ判定）
- スマホ実機での確認（GPS・ナビ・長押し・画面回転）はまだ

## 作業の注意

- ナビのシミュレーション（`gentsuki-sim.js`）はユーザーが「公開したくない」と言ったので、リポジトリに入れない。パソコンのブラウザのコンソールに貼り付けて使う、手元だけのファイル

- 動作確認はブラウザで。JS を変えたら `index.html` の `?v=` を上げる
- Valhalla・Nominatim・OpenFreeMap は公開サーバーなので、テストで連打しない
- **公開リポジトリなので、ユーザーの住んでいる場所が分かる情報を書かない**。地名・駅名・道路名・施設名を README・CLAUDE.md・コードのコメント・コミットメッセージ・PR の説明に入れない。例が要るときは「○○線」のように伏せるか、遠く離れた有名な場所（清水寺など）を使う。テストで使った場所も書かない
- コミットの作者は `koppepandayooo` と GitHub の noreply アドレス（リポジトリのローカル設定）。ほかの名前・メールアドレスを公開リポジトリに入れない
