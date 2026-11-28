# 公開サイトのアーキテクチャ案

調査日: 2026-11-28。builder と site Worker を分ける構成を、既存ツールの比較、EmDash の実装、Cloudflare の現行機能の調査をもとに詰めた。関連イシュー: #3、#5、#7、#8、#9、#10。

## 0. 結論

- **描画方式**: R2 の成果物から site Worker が SSR し、Workers Cache（タグ無効化と stale-while-revalidate）で受ける。実質は ISR に近い。全ページの静的書き出し（SSG）も、正本の DB から毎回描く EmDash 型の SSR も採らない。
- **限定公開**: Ghost と同じくサーバー側で分離する。ただし本文に混ぜて返すのではなく、限定ブロックを別のエンドポイントから `private, no-store` で返す。ページ HTML は全員に同じものを返せるので、キャッシュが単純になる。
- **リアルタイム化**: 後回しにしてよい。やるなら garden.ooo と同じく「見ているページだけ購読」とし、LiveSync 用の DO とは別の通知用 DO に、WebSocket Hibernation で載せる。
- **キャッシュの無効化**: Cron から purge すると効かない既知のバグがあるので、builder は site Worker の RPC 経由で、影響するページのタグだけ purge を依頼する。この経路が効くかは、site Worker を作る前に最小の検証で確かめる。

## 1. 既存ツールの比較

| ツール | 公開の選択 | 描画 | 限定公開 | 更新の反映 | 教訓 |
|---|---|---|---|---|---|
| Obsidian Publish | クライアントから手動 push。`publish: true` は選択画面で自動選択されるだけ | SPA。ブラウザが生の `.md` を取得して描画 | サイト全体のパスワードのみ | 手動 publish 直後 | 削除・リネームの反映が手動のため古いページが残る。生の Markdown をブラウザに渡すと、限定部分を隠せない |
| Quartz v4 | 既定は `draft: true` を除外（opt-out）。`ExplicitPublish` で opt-in | SSG。本番は毎回全件ビルド | なし | 再ビルドとデプロイ | slug の一覧を絞り込み前の全ファイルから作るので、非公開ページへのリンクが残る。Markdown 以外のファイルは既定で全部公開される |
| Digital Garden プラグイン | `dg-publish: true` | Obsidian 内で変換し GitHub に push、Eleventy でビルド | なし | git と CI で数分 | 埋め込み先のノートが非公開でも、本文が埋め込まれて漏れる |
| Ghost | 記事ごとに public / members / paid / tiers | SSR | サーバー側で HTML から限定部分を除去。抜粋も除去後の HTML から作る | 即時 | 会員のリクエストは `Cache-Control: private`。プレビューとギフトリンクは常にキャッシュしない |
| garden.ooo | `visibility`。無ければ `publish: true` だけ公開 | Nuxt の SSR と hydration。nginx 直で、CDN キャッシュの `Cache-Control` は無い（実サイトで確認） | ページ単位の `protected`、ブロック単位の `::gated`（GitHub Sponsors） | SSE で閲覧者にも通知 | `protected` は高速な素の HTML 経路から外し、アプリ側で判定する |
| EmDash | draft / scheduled / published。ページ単位の非公開・パスワードは無い | Astro の SSR。Workers Cache（`Cache-Tag`）と KV のオブジェクトキャッシュ | 組み込みは無い（テンプレートで判定）。x402 はページ側で `enforce()` を呼ぶ | 公開時にタグで purge。閲覧者への通知は無い | Worker の外からの書き込みでは purge されない（#2435）。Cron からの purge が効かない（#4957、#5151） |

### 比較から得た設計規則

1. **公開集合の外を参照させない**: 埋め込み、リンク先、添付ファイル、バックリンク、検索索引、グラフ、フィードは、公開ページの集合だけを対象に解決する。非公開ページへのリンクはただのテキストにする（Digital Garden と Quartz の漏洩を避ける）。
2. **非公開化と削除も自動で反映する**: `_changes` を使えば、公開と同じ経路で削除も拾える（Obsidian Publish の弱点を避ける）。
3. **限定部分はサーバー側で分離する**: 抜粋、OG の説明文、RSS、検索用テキスト、エージェント向け Markdown は、限定部分を除いた「公開用の投影」から作る（Ghost）。
4. **キャッシュは閲覧権で分ける**: 公開 HTML は共有キャッシュ、限定部分と x402 は `private, no-store`。CDN が無視するかもしれない cookie の慣習に頼らない（WordPress の弱点）。
5. **差分処理と冪等性**: 変更フィードと依存関係の追跡で、影響するページだけ作り直す。成果物は `_rev` を鍵にして冪等にする。

## 2. 描画方式の比較

| 方式 | 例 | 長所 | 短所 | 判定 |
|---|---|---|---|---|
| SSG（全ページを HTML で書き出す） | Quartz、Digital Garden | 配信が最も単純で安全 | バックリンクや一覧の変更で多数のページを書き直す。限定部分を混ぜられない | 不採用 |
| SPA（Markdown をブラウザで描画） | Obsidian Publish | サーバーが軽い | JS が必須で SEO に弱い。限定部分を隠せない | 不採用 |
| 正本の DB から毎回 SSR | EmDash、garden.ooo、Ghost | 動的な機能を入れやすい | 正本の `VaultDO` を公開トラフィックにさらす。DO は全リクエストを直列に処理する | 不採用 |
| **R2 の成果物から SSR し、キャッシュで受ける** | （EmDash の構成の、データ源を R2 に置き換えたもの） | 重い変換は builder で済み、描画は軽い。テーマの変更で全ページを書き直さない。バックリンクの変更は影響するページだけ書き直す。限定部分を別経路にできる | キャッシュ無効化の設計が要る | **採用** |

## 3. 構成

```
Obsidian ──LiveSync──▶ VaultDO
                          │ _changes?since=checkpoint（internal fetch、小さなバッチ）
                          ▼
                 [builder]  既存 Worker の scheduled ハンドラ（毎分）
                   │ 1. 公開集合を計算し、変換して R2 に書く
                   │ 2. site Worker の RPC で、影響するページのタグだけ purge を依頼
                   │ 3. 通知用 DO に変更を知らせる（任意）
                   │ 4. Webmention の送信を Queue に積む（#9）
                   ▼
            R2 `garden-site` ◀── 読み取り専用 ── [site Worker] ◀── Workers Cache ◀── 読者
                                                    │
                                     /_gated/*  → D1（閲覧権）、x402
```

### 3.1 builder

- 置き場所: 既存の `livesync-workers` Worker の `scheduled` ハンドラ。`worker/publish/` に置き、`packages/` は変えない。
- 入力: `VaultDO` の `_changes`。`VaultDO` は全リクエストを `withExclusive` で直列に処理するので、1 回の件数とバイト数を小さく制限し、残りは次の Cron に回す。
- 公開判定: `visibility` を最優先し、無ければ `publish: true`（`published: true`）だけを `public`、それ以外を `private` とする。最初は `public` と `private` の二値で始め、`unlisted`（一覧・フィード・バックリンクに出さず `noindex`）を足せるスキーマにする。
- 出力（`garden-site`）:
  - `pages/<slug>.json`: 公開用の投影から作った HTML 断片、タイトル、日付、抜粋、外向きリンク、バックリンク、限定ブロックの ID 一覧、`_rev` のハッシュ。ページの描画に要る派生情報はここに焼き込み、site Worker が描画時に manifest を読まなくて済むようにする
  - `pages/<slug>.md`: 公開用の投影の Markdown（エージェント向け、後述）
  - `gated/<slug>/<blockId>.json`: 限定ブロックの中身と、必要な閲覧権（#5）や価格（#7）
  - `manifest.json`: slug とパスの対応、公開ページの一覧、リンクグラフ、世代番号。builder の依存関係の追跡と、サイト全体の一覧ページ（トップ、タグ一覧）だけが使う
  - `assets/<hash>`: 公開ページから参照された添付ファイルだけ
  - `feed.xml`（#8）、`sitemap.xml`
- 限定ブロックの記法: garden.ooo に合わせて Obsidian 上で読める形にする（例: callout `> [!gated]`、または `::gated` ブロック）。どちらにするかは未決。
- 依存関係: ノート A のリンクが変わったら、旧リンク先と新リンク先の `pages/*.json` を作り直す。manifest のリンクグラフで追跡し、作り直したページのタグだけを purge 対象にする。公開ページの追加・削除・タイトル変更があったときだけ、一覧ページ用の `nav` タグも purge する。
- 公開集合の外を参照させない規則（1 章の規則 1）は、限定ブロックの中身にも適用する。閲覧権のある人にも、非公開ノートの埋め込みは見せない。
- 安全装置:
  - DO の作り直しの検出: `_local/publish-builder` に epoch を書き、消えていたら全件再構築する（`instance_start_time` は常に `"0"` で使えない）。update_seq が checkpoint より小さい場合も全件再構築する。
  - 非公開化は常に即時に反映する（fail-closed）: frontmatter の `visibility` や `publish` を明示的に変えた場合は、件数に関係なく、その実行で成果物を消して purge する。本人が隠したいページを、承認待ちの間も公開し続けてはいけない。
  - 大量削除の保留は「文書が消えた」場合に限る: 削除（`_deleted`）や、照合でノートが見つからない場合だけを数える。1 回で公開ページの 10% 以上、かつ 5 件以上を消すことになる場合は、その分の削除を保留して管理画面で警告し、承認を待つ（garden.ooo の「1 割が消える」事故と同じ型を防ぐ）。ページ数が少ないうちに 1〜2 件の削除で止まらないよう、絶対数の下限を併用する。
  - 1 日 1 回の全件照合: `listNoteStats` と manifest を突き合わせる。あわせて、公開ページから参照されなくなった `assets/*` と、元のブロックやページが無くなった `gated/*` を消し、該当のタグを purge する。非公開化したページの `gated/<slug>/*` は、照合を待たずに非公開化と同じ実行で消す。
  - 変換規則やテーマの変更時はバージョン番号を上げて全件再構築する（既存の `CURRENT_INDEX_VERSION` と同じ型）。

### 3.2 site Worker

- binding は `garden-site`（読み取り）、閲覧権の D1、通知用 DO だけ。`VaultDO`、`garden`（`fts/` に非公開ノートを含む全文索引がある）、`OAUTH_KV`、Vectorize は持たせない。
- 公開ページ: `pages/<slug>.json` だけからレイアウトを組み立てて返す。`Cache-Control: public, max-age=…, stale-while-revalidate=…` と `Cache-Tag: page:<slug>` を付け、Workers Cache で受ける。サイト全体の一覧に依存するページ（トップ、タグ一覧、フィード）にだけ `nav` タグを付ける。全ページに共通のタグは付けない。1 ページの更新で全ページのキャッシュが消えるからである。
- 無効化: `purge(tags)` を RPC メソッドとして公開し、builder から service binding で呼ぶ。
  - 内容の変更には `invalidate()` ではなく `purge()` を使う。`invalidate()` はエントリーを stale にするだけで削除しない（公式ドキュメント）。stale のエントリーが次のリクエストに返るかどうかはドキュメントに書かれておらず、SWR と組み合わさると古い HTML が返る恐れがある。
  - 1 回の purge で送れるタグ数は、ダッシュボードでは 100 個まで（公式ドキュメント）。API の上限は未確認なので、builder はタグを一定数ごとに分けて送る。
  - site Worker の中から purge する理由:
  - Workers Cache のキャッシュキーには、パス、エントリーポイント、Worker のバージョンが含まれる（公式ドキュメント）。キャッシュは site Worker のものなので、無効化も site Worker の中から行うのが確実である。purge が他の Worker やエントリーポイントからどこまで効くかは、ドキュメントに明記が無い。
  - Cron から呼んだ purge は成功を返すのに何も消えない既知の不具合がある（emdash-cms/emdash #4957、#5151、cloudflare/developer-platform#163）。公式ドキュメントも、Cron の呼び出しは「常にキャッシュを介さずに実行される」とだけ書き、Cron からの purge が効くかは述べていない。#5151 は、通常のエントリーポイントからの RPC による purge なら効くと報告している。
  - purge は契約プランに関係なく Free プランの回数制限（エージェント調査では毎分 5 回、バースト 25）に従う（公式ドキュメント）。1 回の実行で変わったタグをまとめて 1 回で送る。
  - Worker のバージョンが既定でキャッシュキーに入るので、デプロイのたびにキャッシュは空から始まる。
- 限定ブロック: ページ HTML には場所だけを置き、中身は `/_gated/<slug>` からクライアントがまとめて取得する（1 ページに複数のブロックがあってもリクエストは 1 回）。閲覧者が読めるブロックだけを返す。このエンドポイントは閲覧権の判定（#5）または x402（#7）を通ったときだけ中身を返し、常に `Cache-Control: private, no-store` を付ける。
  - EmDash の x402 パッケージはキャッシュ用のヘッダーを一切付けないので、キャッシュするルートと組み合わせると、払っていない人に有料の HTML が返りうる。自前の実装では、ヘッダーをエンドポイント側で固定する。
  - JS が無い環境では「メンバー限定」の表示だけになる。限定部分を検索エンジンに載せる必要はないので、問題にならない。
- エージェント向け: Markdown は `/<slug>.md` という別の URL で返し、HTML からは `<link rel="alternate" type="text/markdown">` で示す。Workers Cache は `Vary: Accept` をキャッシュキーに含めると明記しているが（公式ドキュメント）、ブラウザや途中のキャッシュまで含めると URL を分けるほうが事故が少ない。`Accept: text/markdown` の返し分けを足す場合も、URL の分離を正とする。Cloudflare の Markdown for Agents は Pro 以上の機能だが、builder が元の Markdown を持っているので自前で返せる。
- microformats2（h-card、h-entry）をテンプレートに入れ、#9 と #10 に備える。

### 3.3 閲覧権と認証（#5、#7）

- 閲覧権は garden.ooo の Entitlement の考え方に従い、サービスごとのアダプタの結果を D1 に正規化して保存する。`VaultDO` には入れない。
- 認証: GitHub と Discord は OAuth。Resend の購読者はマジックリンク。メール送信は Resend の API を使う（Cloudflare Email Sending は public beta で Paid プラン限定）。フォームには Turnstile を付ける。
- x402: Cloudflare Agents SDK の `x402-hono` ミドルウェアが Workers で使える。Monetization Gateway は米国の事業者向けの closed beta なので、個人サイトでは使えない前提にする。
- 自分だけの領域（下書きのプレビューなど）が必要なら、Cloudflare Access を Worker に付けるのが手軽である（2026-08 からパス単位の保護が可能）。ブロック単位の判定には使えない。

### 3.4 リアルタイム化（任意、最後の段階）

- garden.ooo は閲覧者にも SSE で変更を通知する。閲覧者は見ているページの `content_id` だけを購読し、表示から 3 秒後に接続し、非表示のタブでは止める。public / garden / unlisted のページだけを通知し、非公開化や削除は通知しない。
- Cloudflare では、通知用の DO を別に作り、WebSocket Hibernation で接続を持つ。休止中の接続時間は課金されないので、個人サイトの規模ならほぼ無料になる。LiveSync 用の `VaultDO` には載せない（直列処理の待ち行列が増える）。
- 順序: builder が R2 に書く → site Worker の RPC でキャッシュを purge する → 通知用 DO に「ページ X が変わった」を送る → 閲覧者が再取得する。purge より先に通知すると、古いキャッシュを取りに行ってしまう。
- 再取得はキャッシュを迂回させる: purge がデータセンター間に行き渡る前に再取得が来る可能性があるので、通知に `_rev` のハッシュを含め、再取得の URL に付ける（例: `?v=<hash>`）。キャッシュキーが変わるので、古いエントリーには当たらない。
- 遅延: Cron は最短 1 分なので、反映は 1〜2 分になる。これ以上速くするには `VaultDO` の書き込み時に builder を起こす仕組みが要り、upstream のパッケージに手を入れることになる。デジタルガーデンでは 1〜2 分で十分と考える。

## 4. Cloudflare の機能の採否

| 機能 | 状態（2026-11 時点） | 採否 | 用途 |
|---|---|---|---|
| Workers Cache | 全プランで利用可（2026-07 開始）、`invalidate()` は 2026-09 追加 | 採用 | 公開ページの共有キャッシュ、タグ無効化、SWR |
| Cache API（`caches.default`） | データセンター単位 | 不採用 | Workers Cache で置き換える |
| Service binding / RPC | GA | 採用 | builder から site Worker への無効化依頼 |
| Queues | Free プランでも利用可（2026-02 から） | 採用（#9） | Webmention 送信 |
| Workflows | GA、ステップ課金 | 保留 | 変換 → OG 画像 → 無効化 → 外部通知の多段処理が増えたら |
| DO の WebSocket Hibernation | GA | 任意 | 閲覧者へのリアルタイム通知 |
| D1 | ― | 採用 | 閲覧権、受信した Webmention |
| `x402-hono` | Agents SDK に同梱 | 採用（#7） | 従量課金 |
| Cloudflare Access | Worker 単位の保護に対応 | 任意 | 自分専用のプレビュー |
| Turnstile | GA、無料 | 採用 | マジックリンクのフォーム |
| Analytics Engine | Free で利用可 | 任意 | 閲覧数（garden.ooo の view count に相当） |
| Images の変換 | 月 5,000 件まで無料 | 任意 | 画像のリサイズ |
| OG 画像 | satori などを Workers で使う。Browser Run は高い | 任意 | 共有用の画像 |
| AI Crawl Control | 全プラン | 任意 | AI クローラーの許可・拒否 |
| Markdown for Agents | Pro 以上 | 不採用 | 自前の content negotiation で代替 |
| Dynamic Workers | open beta、Paid | 将来 | 利用者が書く変換処理の隔離 |
| Email Sending | public beta、Paid | 不採用（当面） | Resend で代替 |
| AI Search | GA | 不採用 | 既存の Vectorize と重なる |

### 4.1 試したい機能のメモ

| 機能 | 何に使うか | 実装の要点 | 注意点 |
|---|---|---|---|
| 元の Markdown を返す | AI エージェントに、HTML ではなく Markdown でページを読ませる | builder が `pages/<slug>.md` を書き出し、site Worker が `/<slug>.md` で返す。HTML からは `<link rel="alternate" type="text/markdown">` で示す。`Accept: text/markdown` での返し分けは任意で、足すなら `Vary: Accept` を付ける | 公式の [Markdown for Agents](https://developers.cloudflare.com/fundamentals/reference/markdown-for-agents/) は Pro 以上で、HTML をエッジで変換するもの。こちらは元の Markdown を持っているので Free でも返せる。返すのは限定部分を除いた公開用の投影に限る |
| `x402-hono` | 有料のページやブロックへの従量課金（#7） | Cloudflare Agents SDK に同梱の Hono ミドルウェアを `/_gated/*` に掛ける。MCP 向けには `paidTool` もある（[Agents payments](https://developers.cloudflare.com/agents/tools/payments/)） | 応答には常に `Cache-Control: private, no-store` を付ける（EmDash の x402 パッケージはキャッシュ用のヘッダーを付けない）。ステータスが beta か GA かの表記は確認できなかった |
| Analytics Engine | 閲覧数の記録（garden.ooo の view count に相当） | site Worker から `writeDataPoint` でページごとに書き、SQL API で集計する。cookie を使わずに数えられる | Free プランで 1 日 10 万データポイントまで。課金が始まっているかは未確認 |
| Cloudflare Access | 自分だけが見られる下書きプレビュー | site Worker の `/_preview/*` に Access のポリシーを掛け、`ctx.access.getIdentity()` で本人か確かめる（2026-08 から Worker 単位で保護できる。[Access for Workers](https://developers.cloudflare.com/workers/configuration/cloudflare-access/)） | パス単位の保護なので、ブロック単位の限定公開（#5）には使えない。プレビューの応答はキャッシュしない。Free プランの人数上限は未確認 |

## 5. 段階的な導入

1. 契約プランを確認し（6 章）、#3 のバックアップを決める。どちらも builder の設計全体の前提になる。
2. slug とリダイレクトの規則を決める。公開後に変えにくいので、builder を作る前に決める。
3. builder の最小版: `publish: true` のページを `garden-site` に書き出す。非公開化の即時反映、削除の反映、大量削除の保留を最初から入れる。
4. purge の検証: Cron から service binding で呼んだ site Worker の RPC メソッドの中の purge が、fetch でキャッシュしたエントリーに効くかを、最小の検証用 Worker で確かめる。段階的デプロイで複数のバージョンが同時に動いているときに、タグの purge がバージョンをまたいで効くかも確かめる。効かなければ、無効化の経路を組み直す（例: builder から site Worker へ HTTP の fetch で依頼する）。
5. site Worker の最小版: ページ表示、404、Workers Cache、RPC による purge。
6. wikilink の解決（公開集合だけ）、バックリンク、画像。
7. フィード（#8）、エージェント向け Markdown、h-card と h-entry、Bridgy Fed（#10）。
8. Webmention（#9）。
9. 限定ブロックと閲覧権（#5）、x402（#7）。
10. リアルタイム通知（任意）。

## 6. 未決の論点

- **契約プラン**: Workers Free の CPU 時間は 1 回 10 ms で、EmDash でも Cron が Free の上限で止まる不具合があった（#3858）。builder の Markdown 変換は Free では収まらない可能性が高い。現在のアカウントが Paid かを確認する。
- Obsidian 方言（wikilink、埋め込み、callout）の変換器。Quartz の `ofm.ts`（unified ベース）が参考になる。garden.ooo も unified への移行を想定している。
- 限定ブロックの記法（callout か `::gated` か）。
- テンプレートに何を使うか（Hono の JSX が最も軽い。Astro は Workers Cache との統合が組み込みだが、データ源が R2 なら利点は小さい）。
- slug の決め方とリネーム時のリダイレクト（5 章の段階 2 で決める）。EmDash は slug の変更時に 301 を自動で作り、リダイレクトの連鎖を畳む。garden.ooo は frontmatter に不変の `content_id`（ULID）を持たせてリネームを追跡する。frontmatter はノートと一緒に LiveSync で同期されるので、この fork でも同じ方式を取りやすい。
- 予約公開（未来の `date`）を扱うか。扱うなら UTC に正規化する（EmDash #2896）。

## 7. 出典

- 比較対象の公式資料: [Obsidian Publish](https://help.obsidian.md/publish/publish)、[Quartz private pages](https://github.com/jackyzha0/quartz/blob/v4/docs/features/private%20pages.md)、[Digital Garden の GardenPageCompiler](https://github.com/oleeskild/obsidian-digital-garden/blob/main/src/compiler/GardenPageCompiler.ts)、[Ghost post-gating.js](https://github.com/TryGhost/Ghost/blob/main/ghost/core/server/api/endpoints/utils/serializers/output/utils/post-gating.js)、[Ghost frontend-caching.js](https://github.com/TryGhost/Ghost/blob/main/ghost/core/frontend/web/middleware/frontend-caching.js)
- EmDash: [route caching](https://docs.emdashcms.com/deployment/route-caching/)、[object cache](https://docs.emdashcms.com/deployment/object-cache/)、[x402 payments](https://docs.emdashcms.com/guides/x402-payments/)、[#4957](https://github.com/emdash-cms/emdash/issues/4957)、[#5151](https://github.com/emdash-cms/emdash/issues/5151)、[#2435](https://github.com/emdash-cms/emdash/issues/2435)
- Cloudflare: [Workers Cache](https://developers.cloudflare.com/workers/cache/)、[Workers Cache invalidate](https://developers.cloudflare.com/changelog/post/2026-09-29-workers-cache-invalidate/)、[DO pricing](https://developers.cloudflare.com/durable-objects/platform/pricing/)、[Agents payments](https://developers.cloudflare.com/agents/tools/payments/)、[Access for Workers](https://developers.cloudflare.com/workers/configuration/cloudflare-access/)、[Markdown for Agents](https://developers.cloudflare.com/fundamentals/reference/markdown-for-agents/)
- garden.ooo: [replication](https://baku89.com/ja/garden/replication)（SSE）、[layout: html](https://baku89.com/ja/garden/layout-html)（素の HTML 経路）、[visibility](https://baku89.com/ja/garden/visibility)、[Markdown ガイド](https://baku89.com/ja/garden/manual/markdown-guide)（`::gated`）、[devlog](https://baku89.com/ja/garden/devlog)（Entitlement）
- サーベイ: [docs/surveyor/topics/digital-garden-cms/overview.md](../surveyor/topics/digital-garden-cms/overview.md)
