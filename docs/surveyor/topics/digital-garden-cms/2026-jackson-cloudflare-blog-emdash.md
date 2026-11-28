# The Cloudflare Blog — brought to you by EmDash

The Cloudflare Blog, 2026（8 月 24 日）

https://blog.cloudflare.com/cloudflare-blog-uses-emdash/

- Kody Jackson (Cloudflare)
- Diogo Carneiro (Cloudflare)
- Amy Dutton (Cloudflare)

## どんなもの？

- Cloudflare が自社ブログを外部ベンダーの CMS から EmDash に移した記録である。移行日は 2026 年 8 月 12 日で、デザインの刷新も同時に行った。
- 「Customer Zero」（自社製品を最初の顧客として使い、問題を先に踏む）という社内方針の実践例として書かれている。
- 移行で見つかった EmDash の不足、負荷試験の設計、本番構成、切り替え手順、移行後の結果を順に報告する。

## 先行研究と比べてどこがすごいの？

- EmDash の発表記事（[2026-taylor-emdash.md](2026-taylor-emdash.md)）には定量的な裏づけが無かった。本記事は、通常 75 RPS、ピークで 5,000 RPS を超える実サイトで EmDash を運用した最初のまとまった報告である。
- 移行の過程で、Workers KV を使うオブジェクトキャッシュ、Hyperdrive 経由の PlanetScale 接続、Workers Cache への対応が EmDash に追加された。1.0 の発表は、これらがいずれもこの移行から生まれた機能だと書いている。

## 技術や手法の"キモ"はどこにある？

- **機能の確認**: 公開と非公開化、新規作成、予約公開、メディア追加という日常の操作を一通り試した。不足は、規模（メディア、検索、署名欄）、多言語化と SEO と CSP、長い記事を書くときの編集画面の使い勝手に集中した。最大の見落としは予約公開で、v0.19.0 まで動かなかった。
- **負荷試験**: k6 で 3 つのシナリオを作った。本番の基準値の 3 倍まで徐々に上げる Ramp、0 から 100 RPS へ 10 分かけて上げて壊れる点を探す Breakpoint、いきなり 7,000 RPS をかける Burst である。合格基準は、5xx が 0.01% 以下、p95 が 500 ms 以下、p99 が 1,000 ms 以下とした。
- **本番構成**: EmDash を Worker で動かし、その前に Workers Cache を置き、KV のオブジェクトキャッシュを挟み、DB は Hyperdrive 経由の PlanetScale にした。静的ファイルの 99.5%、リクエスト全体の 70% をキャッシュから返している。
- **切り替え**: 旧ブログと新ブログの前にプロキシ Worker を置き、バージョンを示す cookie で振り分けた。新ブログが 500 を返したら旧ブログに戻す。プロキシから新ブログへは service binding で直接渡すので、DNS や TLS を経由しない。当日は 1%、5%、15% と段階的に流量を上げ、その日のうちに 100% に移した。

## どうやって有効だと検証した？

- 旧構成と比べて p95 のレイテンシが平坦になり、負荷時の周期的なスパイクが消えたとグラフで示している。Lighthouse のスコアも示している。最大 850 RPS を配信しながらエラーはごくわずかだったという。
- Agents Week では 9 日間に 28 本を公開し、約 300 万 PV を受けた。フロントエンドは最大 450 RPS を問題なく処理し、28,000 RPS の DDoS も Cloudflare の防御で吸収した。
- 一方で、編集側では予約公開まわりのバグが再び見つかっている。

## 議論はあるか？

- **配信と編集で成熟度が違う**: 配信側の性能は十分だったが、編集側の不具合（予約公開、編集画面の細かな挙動）は移行後も続いた。CMS の評価では、読者側の性能より、書き手の操作の信頼性が先に問題になることを示している。
- **キャッシュに頼った性能**: リクエストの 30% はキャッシュを通らない。性能の多くは、Cloudflare の新製品（Workers Cache、Hyperdrive）を組み合わせた結果であり、EmDash 単体の性能とは切り分けられていない。
- **記述の時系列の不整合**: 移行日を 8 月 12 日としながら、新ブログで 8 月 10 日の DDoS を吸収したと書いている。どちらかの日付の誤りと思われる。
- 比較対象の旧 CMS ベンダーの名前と、その構成は書かれていない。

## 次に読むべき論文はあるか？

1. Matt "TK" Taylor et al. (2026). [Introducing EmDash — the spiritual successor to WordPress that solves plugin security](https://blog.cloudflare.com/emdash-wordpress/). The Cloudflare Blog.
   - EmDash の設計の全体像。
2. Cloudflare. [Workers Cache](https://blog.cloudflare.com/workers-cache/). The Cloudflare Blog.
   - 本番構成の最前段に置いたキャッシュ。
3. EmDash. [Object cache](https://docs.emdashcms.com/deployment/object-cache/). EmDash Docs.
   - 移行のために作られた KV ベースのキャッシュ。
4. Cloudflare. [Markdown for Agents](https://blog.cloudflare.com/markdown-for-agents/). The Cloudflare Blog.
   - エージェント向けに読みやすい形でコンテンツを返す設計の背景。
