# Introducing EmDash — the spiritual successor to WordPress that solves plugin security

The Cloudflare Blog, 2026（4 月 1 日）。GitHub の README（1.0 時点）もあわせて読んだ。

https://blog.cloudflare.com/emdash-wordpress/

https://github.com/emdash-cms/emdash

- Matt "TK" Taylor (Cloudflare)
- Matt Kane (Cloudflare)

## どんなもの？

- Cloudflare が公開した、Astro の integration として動く TypeScript 製の CMS である。WordPress の「精神的後継」を名乗り、管理画面、プラグイン、テーマという WordPress の構成を引き継ぎつつ、サーバーレスの実行基盤の上に作り直している。MIT ライセンスで、WordPress のコードは使っていない。
- 解決したい問題として最も強調しているのは、WordPress のプラグインのセキュリティである。記事は Patchstack の調査を引いて、WordPress の脆弱性の 96% がプラグインに由来すると述べる。
- 発表時点は v0.1.0 のプレビューで、作者らは AI コーディングエージェントを使って 2 か月で作ったと書いている。

## 先行研究と比べてどこがすごいの？

- **プラグインの隔離**: WordPress のプラグインは本体と同じ PHP プロセスで動き、DB とファイルシステムに直接触れる。EmDash のプラグインは Dynamic Worker（Worker Loader で起動する V8 isolate）の中で動き、manifest で宣言した capability だけを binding として受け取る。外部ネットワークへのアクセスも既定では無く、必要なら接続先のホスト名を宣言する。
- **マーケットプレイスへの依存の解消**: 実行環境が隔離されていれば、利用者はマーケットプレイスの審査や評判ではなく、宣言された権限を見て導入を判断できる。プラグインは本体とコードを共有しないので、GPL を継承する必要もなく、作者が自由にライセンスを選べる、と記事は主張する。
- **scale-to-zero**: WordPress はサーバーを常時用意する必要がある。EmDash は Workers 上でリクエストごとに isolate を起動し、リクエストが無ければゼロまで縮む。課金は CPU 時間だけになる。
- **AI からの操作を前提にする**: すべてのインスタンスが MCP サーバーを内蔵し、CLI と Agent Skills を同梱する。管理画面でできる操作を、エージェントからも同じ範囲で行える。

## 技術や手法の"キモ"はどこにある？

- **コンテンツの保存形式**: リッチテキストは HTML ではなく Portable Text（構造化 JSON）で保存する。編集画面は TipTap である。WordPress が HTML にメタデータをコメントで埋め込むのに比べ、表示形式（Web、アプリ、メール、API）から内容を切り離せる。
- **スキーマは DB に置く**: コンテンツタイプはコードではなく DB 上で定義し、管理画面から作れる。コレクションごとに型付きカラムを持つ実テーブルが作られ、開発者は `npx emdash types` で稼働中のスキーマから TypeScript の型を生成する。
- **ビルドしない配信**: ページは Astro の Live Collections（`getEmDashCollection()`）で DB を直接読み、サーバーで描画する。内容を変えても再ビルドは要らない。
- **移植可能な抽象**: SQL は Kysely、ストレージは S3 API で抽象化している。Cloudflare では D1、R2、KV を使い、ほかに SQLite、Turso、PostgreSQL、AWS S3、ローカルファイル、Redis で動く。Node.js ではプラグインを `workerd` の子プロセスで隔離する。
- **テーマは DB に触れない**: テーマは Astro プロジェクト（ページ、レイアウト、コンポーネント、スタイル、コンテンツタイプを定義する seed JSON）であり、DB 操作はできない。WordPress の `functions.php` が何でもできるのと対照的である。
- **x402 の内蔵**: どのコンテンツを有料にするか、価格、ウォレットのアドレスを設定すると、未払いのリクエストに HTTP 402 を返し、支払い後に通す。購読契約なしの従量課金で、主な支払い手として AI エージェントを想定している。
- そのほか、passkey を既定にした認証と 4 段階のロール（Administrator、Editor、Author、Contributor）、改訂履歴と下書きと予約公開、FTS5 による全文検索、WXR や REST API からの WordPress 移行がある。

## どうやって有効だと検証した？

- 発表記事の段階では、定量的な評価は無い。Playground と Deploy ボタンで試せることを示すにとどまる。
- 実運用での検証は、後の Cloudflare Blog の移行（[2026-jackson-cloudflare-blog-emdash.md](2026-jackson-cloudflare-blog-emdash.md)）で示された。

## 議論はあるか？

- **データの実体が DB にある**: 本文は Portable Text として DB に入り、スキーマも DB にある。このトピックの他のツールがプレーンテキストを SSOT にするのに対し、EmDash は DB を SSOT にする従来型の CMS である。移植性は「どの DB でも動く」ことで担保しており、「どのエディタでも開ける」ことでは担保していない。
- **Cloudflare 依存の懸念**: Node.js でも動くとはいえ、プラグインの隔離は `workerd` に、キャッシュや配信の最適化は Cloudflare の製品に寄っている。WordPress の創設者 Matt Mullenweg は、発表翌日の CMSWire の記事で、Cloudflare とシームレスに連携し、ベンダーを乗り換えにくくしたい人には「incredible choice」だと皮肉り、セキュリティモデルは Cloudflare でしか機能しないとも述べた。また、プラグインが WordPress のあらゆる挙動を変えられることは欠陥ではなく機能だとして、隔離という前提そのものにも反論している。なお、1.0 の README と発表は Node.js 上でも `workerd` で隔離すると説明しており、後者の指摘とは食い違う。
- **資料間の表記揺れ**: プラグインの capability 名が、発表記事のコードでは `read:content`、README では `content:read` になっている。発表記事の本文はさらに、フック名 `content:afterSave` を capability と呼んでいる。実装時は最新のドキュメントで確認する必要がある。
- **エコシステムの小ささ**: プラグインとテーマの数は、WordPress とは比べられない段階である。

## 次に読むべき論文はあるか？

1. Kody Jackson et al. (2026). [The Cloudflare Blog — brought to you by EmDash](https://blog.cloudflare.com/cloudflare-blog-uses-emdash/). The Cloudflare Blog.
   - 大規模サイトでの移行と負荷試験の記録。
2. Scott Buscemi et al. (2026). [EmDash 1.0: the stable CMS with a secure plugin registry](https://blog.cloudflare.com/emdash-cms-plugin-registry/). The Cloudflare Blog.
   - 1.0 と AT Protocol を使ったプラグインレジストリ。
3. Cloudflare. [Dynamic Workers](https://developers.cloudflare.com/dynamic-workers/). Cloudflare Docs.
   - プラグイン隔離の基盤。livesync-workers と同じ Workers の上で、信頼しないコードを動かす仕組み。
4. [Portable Text](https://www.portabletext.org/).
   - EmDash の本文の保存形式。Markdown との変換可能性を確かめる出発点。
5. [x402](https://www.x402.org/).
   - HTTP 402 を使う従量課金の標準。garden.ooo の GitHub Sponsors による限定公開と比べる材料。
