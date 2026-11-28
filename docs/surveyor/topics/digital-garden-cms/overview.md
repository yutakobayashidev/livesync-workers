# Topic Overview

このトピックは、個人がノートの同期と Web 公開を一つの仕組みにまとめる「自作 CMS」と「デジタルガーデン」の動向を扱う。調査の動機は、livesync-workers の fork を、同期と公開を同居させる基盤として使う構想である。

資料の中心は baku89 の garden.ooo で、設計思想からデータ同期の実装、公開範囲の設計まで、まとまった記述が公開されている。そのほかに、garden.ooo の作者が「自作CMS仲間」として挙げたツールのうち、詳細を取得できた 4 つ（Hyperstrata、Sate、Intertex、加藤 淳さんの編集画面）を読んだ。

対照群として、Cloudflare の EmDash も加えた。livesync-workers と同じ Workers の上で動く CMS だが、個人の自作ツールではなく、WordPress の後継を狙う汎用の CMS である。

| 資料 | 主題 |
|---|---|
| [2026-baku89-garden-concept.md](2026-baku89-garden-concept.md) | garden.ooo の設計原則、やらないこと、利用モード |
| [2026-baku89-garden-data-sync.md](2026-baku89-garden-data-sync.md) | 三層のデータ構造、レプリケーション、変更通知、障害事例 |
| [2026-baku89-garden-visibility.md](2026-baku89-garden-visibility.md) | 公開範囲、永続ページ、ダダ漏れの思想 |
| [2026-baku89-garden-members-only.md](2026-baku89-garden-members-only.md) | GitHub Sponsors 連携のメンバー限定ブロック、Entitlement 構想 |
| [2026-baku89-garden-series.md](2026-baku89-garden-series.md) | 順序つきの記事シリーズ、CMS 自作の勧め |
| [2026-baku89-how-to-vibe-code-cms.md](2026-baku89-how-to-vibe-code-cms.md) | CMS 自作の 5 つの教訓 |
| [2026-baku89-jisaku-cms-nakama.md](2026-baku89-jisaku-cms-nakama.md) | 自作 CMS のリンク集 |
| [2026-orito-hyperstrata.md](2026-orito-hyperstrata.md) | 堆積するノートによるサイト形式 |
| [2026-kato-pagecms-and-my-cms.md](2026-kato-pagecms-and-my-cms.md) | Cloudflare Workers 上の自作編集画面 |
| [2026-azuma-sate.md](2026-azuma-sate.md) | 書くことに集中するエディタと公開コマンド |
| [2026-nagara-intertex.md](2026-nagara-intertex.md) | 三つの射影で編集するメタテキストエディタ |
| [2026-taylor-emdash.md](2026-taylor-emdash.md) | Workers 上の TypeScript 製 CMS、プラグインの隔離、x402 |
| [2026-jackson-cloudflare-blog-emdash.md](2026-jackson-cloudflare-blog-emdash.md) | Cloudflare Blog の EmDash への移行と負荷試験 |
| [2026-buscemi-emdash-1-0.md](2026-buscemi-emdash-1-0.md) | EmDash 1.0、AT Protocol 上のプラグインレジストリ |

資料はどれも 2026 年に書かれた Web ページで、査読はない。個人の資料の裏づけは、作者自身が運用していることと障害の記録である。EmDash の資料は企業の製品発表で、Cloudflare Blog の移行記事だけが負荷試験の数値を示している。

## 論文間の関係性

### 共通する前提: プレーンテキストを SSOT にする

すべてのツールが、データの実体をプレーンテキスト（Markdown または JSON）に置き、ツール側の DB を派生物として扱う。garden.ooo はローカルの `.md` を SSOT とし、ブラウザ DB とサーバー DB をそのミラーにする。Sate は SQLite の索引を消しても `.md` から再生成できる。Intertex は JSON と Markdown への書き出しを重視し、作者は「数年後に開けなくなること」を最も恐れていると書く。加藤さんの編集画面は GitHub 上の Markdown を直接読み書きする。

この前提の理由は二つに分かれる。一つはツールの寿命や作者の気分からデータを守ること（Intertex、Sate）、もう一つはどのエディタや AI からでも同じデータを扱えるようにすること（garden.ooo、加藤さん）である。

EmDash はこの前提を取らない。本文は Portable Text（構造化 JSON）として DB に入り、コンテンツタイプのスキーマも DB にある。移植性は「SQLite、D1、PostgreSQL のどれでも動く」ことで担保し、データの持ち出しは API、CLI、MCP を通して行う。AI から扱えるようにするという目的は garden.ooo と同じだが、手段がファイルではなく API である。複数の編集者とロールを持つ組織のサイトを想定するなら DB を正本にするのは自然な選択で、個人の自作ツールとの違いは利用者の数と権限管理の必要性から来ている。

### 分岐点: 書いた記事を書き換えるか、積み重ねるか

デジタルガーデンの定義（未完成のメモを公開し、時系列で並べず、手入れし続ける）は、garden.ooo と Hyperstrata が共有している。分かれるのは、更新の仕方と時間の扱いである。

- **garden.ooo** は同じページを手入れし続ける。「完成概念をガバガバにする」方針で、順序が必要な場面はシリーズ機能で後から補う。
- **Hyperstrata** は、手入れし続ける負担こそがデジタルガーデンの弱点だとみる。記事は原則として書き換えず、改訂は古い記事を引用した新しい記事として積む。時系列は捨てず、引用ネットワークによる分岐と合流として残す。

二者は対立しているというより、維持管理の負担を誰がどう負うかで違う解を選んでいる。garden.ooo も `permanent` フラグと将来のバージョン管理で「固定したいページ」を区別しており、Hyperstrata の不変性に近い要素を持つ。

### 公開範囲の多段階化

garden.ooo の公開範囲は、公開と非公開の二値ではなく、検索エンジン、サイト内の一覧、リンクを知っている人の三つの経路ごとに到達可否を分ける。Sate の「Private」チェック（一覧と検索から隠す）も、規模は小さいが同じ発想である。どちらも、一つの保管場所に私的なものと公的なものを混在させたまま、見える範囲を制御するという設計を選んでいる。

garden.ooo はさらに、閲覧者が誰かによる制御も持つ。ページ単位の `protected` に加えて、公開ページの本文に `::gated` ブロックを混ぜ、GitHub Sponsors で支援した人が GitHub でログインしたときだけ中身を見せる。決済と会員管理は外部サービスに任せ、garden.ooo は「誰がどのコンテンツを見てよいか」（Entitlement）の判定だけを持つ、という役割分担が構想されている。

EmDash は別の方向から有料コンテンツを扱う。x402 を内蔵し、指定したコンテンツへの未払いのリクエストに HTTP 402 を返し、支払いがあれば通す。garden.ooo の限定公開が「支援者という人」を GitHub ログインで識別するのに対し、x402 は会員登録なしの従量課金で、主な支払い手として AI エージェントを想定している。どちらも、閲覧のたびに判定する動的な経路を必要とする点は共通である。

### 書く道具と公開の仕組みの分離

Sate は公開ボタンに任意のコマンドを割り当てるだけで、公開の中身は利用者に任せる。加藤さんは、ブログ本体（GitHub の Markdown）と編集画面（Workers）を分けた。garden.ooo は Local mode と Server mode を分け、サーバーは公開や同期が必要になって初めて使う。書く場所と公開する場所を、データ形式（Markdown）だけを介して疎に結合する構成が繰り返し現れる。

EmDash は逆に、書く場所（管理画面）と公開する場所（Astro のページ）を一つの DB で密に結合する。ページは Live Collections で DB を直接読んでサーバーで描画するので、静的ビルドは無い。その代わり、Cloudflare Blog の移行では Workers Cache と KV のオブジェクトキャッシュを重ね、リクエストの 70% をキャッシュから返している。静的出力を作らず、動的描画とキャッシュの組み合わせで配信性能を確保する構成である。静的出力との性能比較は資料に無い。

### 同期の実装から得られる教訓

garden.ooo の同期ドキュメントは、このトピックで唯一、同期の失敗例を具体的に記録した資料である。slug 衝突による無限再送、DB 再構築後の `seq` 世代ずれによる片方向同期、サーバー優先の競合解決によるファイルの巻き戻し、そして devlog の「定期的に全ページの1割が勝手に消える」。どれも、同期の不具合がそのままデータ消失や更新の停止につながった例である。

EmDash の移行記事にも、規模は違うが同じ傾向が見える。配信側は 7,000 RPS の Burst を含む負荷試験を経て本番構成を決め、移行後も目立った問題は出なかったが、移行後も残った不具合は予約公開や編集画面といった書き手の側に集中した。読者に見える性能より、書き手の操作が確実に反映されることのほうが、CMS では先に問題になる。

## 今後の展望

### livesync-workers の fork への示唆

garden.ooo の三層構造は、livesync-workers を使った構成と対応づけられる。Obsidian の vault が Local File（SSOT）に、LiveSync プラグインのローカル DB が Local DB に、VaultDO が Remote DB にあたる。この対応から、fork で同期と公開を同居させる設計に次の示唆が得られる。

1. **公開の既定値は非公開にする**: garden.ooo は `visibility` キーが無いとき、`publish: true` または `published: true` のページだけを公開し、それ以外を `private` とする。Obsidian Publish と同じ `publish` キーを使えるので、公開ビルダーの判定規則としてそのまま採用できる。
2. **公開範囲は経路ごとに分ける**: 最初は `public` と `private` の二値で始めてよいが、`unlisted`（一覧とバックリンクに出さない）を加える余地をスキーマに残しておく。公開ページへの変更通知や一覧で、非公開化の事実を漏らさない配慮も必要になる。
3. **変更の検出は `_changes` と seq で行う**: garden.ooo の SSE は、`seq` をイベント ID にして切断中の取りこぼしを再送で埋める。livesync-workers の `_changes?since=seq` は同じ性質を持つので、公開ビルダーは checkpoint として seq を保存し、差分だけを処理すればよい。garden.ooo の DB 再構築時の事故が示すとおり、VaultDO を作り直したときに checkpoint がサーバーの seq より先に進んでいたら、全件を再構築する保護を入れておくべきである。
4. **CouchDB 互換であることの意味**: garden.ooo は単一ユーザー前提で rev tree を捨て、単一の整数 seq に簡素化した。livesync-workers は LiveSync プラグインとの互換のために rev tree を持つ。公開ビルダーが読むのは各文書の勝者リビジョンだけなので、rev tree の複雑さはビルダー側に漏れない。
5. **限定公開は静的出力と相性が悪い**: 先に勧めた構成（公開ビルダーが静的な成果物を書き出し、閲覧は VaultDO に触れない）は、誰が見ても同じ内容のページを前提にしている。GitHub Sponsors 向けのような限定公開を載せるなら、限定部分を静的出力から除き、閲覧者を認証して Entitlement を判定する動的な経路を別に用意する必要がある。garden.ooo も `protected` のページを高速な HTML 経路から外し、アプリ側で判定している。限定公開は公開の初版には含めず、後から足せる形にしておくのが妥当である。
6. **拡張を入れるなら Dynamic Workers で隔離する**: EmDash はプラグインを Dynamic Worker に閉じ込め、manifest で宣言した能力だけを binding で渡す。公開ビルダーに利用者が書いた変換処理（独自の記法やショートコード）を載せるなら、同じ Workers 上で同じ方式を取れる。VaultDO や R2 への binding を直接渡さず、読み取り専用の API だけを渡す、という切り方になる。
7. **EmDash を公開先にする案は、正本が二つになる**: 公開ビルダーが EmDash の API や MCP にページを書き込めば、管理画面や x402 を自作せずに済む。ただし、Markdown から Portable Text への変換が要り、EmDash の DB が vault とは別の正本になる。EmDash 側で編集された内容を vault に戻す経路が無ければ、二つの正本が食い違う。採るなら、EmDash を編集不可の配信先として扱い、vault を唯一の正本に保つ必要がある。
8. **バックアップを先に用意する**: 同期の不具合で全ページの 1 割が消えた garden.ooo の記録は、イシュー #3（vault データのバックアップ）の優先度を上げる根拠になる。公開を載せる前に、vault のバックアップと復元手順を確立しておくべきである。

### 未解決の論点

- **E2E 暗号化と公開の両立**: garden.ooo は Server mode で元の `.md` を E2E で保管する構想を持つが、公開用の HTML をサーバーがどう得るかは説明していない。livesync-workers は E2EE を切る前提なので、この問題は避けられるが、そのぶんサーバーが平文を持つことになる。
- **書き換えと積み重ねのどちらを支えるか**: 公開サイトで Hyperstrata 型の引用ネットワーク（References と Cited By）を出すか、garden.ooo 型のバックリンクを出すかで、公開ビルダーに必要なリンク解析が変わる。どちらもノート間のリンクを収集する点は共通なので、最初はバックリンクの収集だけを作り、表示の仕方は後から選べる。
- **Markdown と Portable Text の対応**: EmDash を公開先にするなら、Obsidian の Markdown（wikilink、埋め込み、callout を含む）を Portable Text にどこまで欠損なく写せるかを確かめる必要がある。資料には変換の仕組みの説明が無い。
- **未取得の資料**: garden.ooo の「同期インテント・ジャーナル」「開発ガイド」「Webmention」は本文が非公開で、loam、bayou / cairn、edane は詳細を確認できなかった。とくに同期インテント・ジャーナル（削除済みページの復活を防ぐ仕組み）は、LiveSync の論理削除やチャンク GC と比較する価値がある。
