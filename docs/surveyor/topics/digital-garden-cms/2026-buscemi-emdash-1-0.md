# EmDash 1.0: the stable CMS with a secure plugin registry

The Cloudflare Blog, 2026（9 月 28 日）

https://blog.cloudflare.com/emdash-cms-plugin-registry/

- Scott Buscemi (Cloudflare)
- Matt Kane (Cloudflare)
- Noah Pham (Cloudflare)

## どんなもの？

- EmDash の安定版 1.0 の発表である。4 月のプレビューから 5 か月間、データの安全性、DB マイグレーション、編集ワークフロー、多言語化、プラグインのセキュリティ、性能に取り組んだと報告する。
- 同時に、AT Protocol（Bluesky の基盤プロトコル）の上に作った分散型のプラグインレジストリと、AI サイトビルダー EmDash Build（alpha）を公開した。
- コアパッケージは semver に従い、破壊的変更はメジャーリリースでだけ行う。公式プラグインとプラグイン用のツールはまだ 0.x である（README の Status 節）。

## 先行研究と比べてどこがすごいの？

- **レジストリが所有権を持たない**: 従来のレジストリは、発行者のアカウント、正本のパッケージ記録、利用者が探すカタログの 3 つの役割を一社が兼ねる。そのため、アカウント停止や掲載削除やサービス終了があると、発行者は同じ ID と履歴を持って他所へ移れない。EmDash のレジストリでは、パッケージとリリースの記録は発行者自身の atproto アカウントに署名つきで置かれ、EmDash はそれを集めて表示するカタログの一つにすぎない。他者も同じ記録から別のカタログを作れる。
- **モデレーションと公開の分離**: 既定のカタログは名前、説明、リンク、画像をモデレーションするが、それはカタログ上で隠すだけで、リリースの書き換えや削除はしない。
- **権限を独立に与える**: プラグインは既定で自分専用のストレージにしか触れず、サイトのコンテンツ、メディア、ユーザー、秘密情報、環境変数、ファイルシステム、ネットワークには触れない。プラグインが宣言し、管理者が承認した能力だけが追加される。メディアへのアクセスを与えてもユーザー情報は見えず、一つのホストへの通信を許してもネットワーク全体は開かない。記事はこれを「モバイルアプリのインストールに近い」と表現する。

## 技術や手法の"キモ"はどこにある？

- **検証の連鎖**: atproto のリポジトリは署名つきの Merkle Search Tree なので、リリース記録が発行者の署名つきコミットに含まれることを包含証明で示せる。EmDash はカタログの写しを信用せず、この証明で記録を独立に検証する。そのうえで、チェックサム、パッケージ名、バージョン、要求する権限、必要ならビルドの来歴を確かめ、ダウンロードしたバンドルが署名済みの記録と一致することを確認する。
- **基盤のオープンソース化**: レジストリを動かすアグリゲーター、Workers AI で説明文をモデレーションするラベラー、プラグイン一覧を Astro サイトに読み込むローダーを公開した。
- **隔離の実装**: Cloudflare では各プラグインを Worker Loader 経由の Dynamic Worker として動かす。Node.js では `workerd` を別プロセスで起動し、その中で各プラグインを独立したサービスとして動かす。manifest と API はどちらでも同じである。
- **プラットフォーム向け**: Workers for Platforms で顧客ごとのサイトを Worker として動かし、EmDash をコンテンツ層に使う構成を想定している。プラットフォームは全レジストリを使うことも、承認済みのプラグインだけに絞ることもできる。
- **EmDash Build**: プロンプトから EmDash サイトを作る。プロジェクトごとに Cloudflare Sandbox のコンテナを用意し、エージェントが MCP 経由でコンテンツモデルを設計して中身を入れ、ページを書く。変更は git のコミットとして記録され、公開時に本番の EmDash サイトへ移される。生成後は人がページ上で直接編集できる点を、プロンプトに戻るしかない AI ビルダーとの違いとしている。

## どうやって有効だと検証した？

- 本番利用の事例として、Avulux が WordPress のマイクロサイトを Agent Skills を使って 1 日以内に移したこと、Cloudflare Blog の移行（[2026-jackson-cloudflare-blog-emdash.md](2026-jackson-cloudflare-blog-emdash.md)）を挙げている。
- 開発体制の指標として、175 人以上のコントリビューター、1,800 以上のコミット、25 言語への翻訳、800 人以上の Discord 参加者を示している。
- レジストリの安全性や性能についての定量的な評価は無い。

## 議論はあるか？

- **有料プラグインは未対応**: 現時点で扱えるのは無料プラグインだけで、有料化は Atproto Spaces を見ながら将来対応するとしている。
- **atproto への依存**: レジストリの分散性は atproto のエコシステムに依存する。発行者には Atmosphere アカウントが要る。
- **プラグインの成熟度**: コアは安定版だが、公式プラグインとプラグイン用ツールは 0.x で、マイナーリリースでも変わりうる。
- **Cloudflare 製品との結びつき**: KV キャッシュ、Hyperdrive、Workers Cache、Workers for Platforms、Cloudflare Sandbox など、1.0 の多くの機能は Cloudflare の製品を前提にしている。Node.js でも動くことと、Cloudflare 上で最もよく動くことの両方を、記事は同時に主張している。

## 次に読むべき論文はあるか？

1. Matt "TK" Taylor et al. (2026). [Introducing EmDash — the spiritual successor to WordPress that solves plugin security](https://blog.cloudflare.com/emdash-wordpress/). The Cloudflare Blog.
   - プラグインの隔離とマーケットプレイス依存の問題設定。
2. [AT Protocol](https://atproto.com/).
   - レジストリの基盤。署名つきの個人リポジトリという考え方は、ローカルを正本とするこのトピックの設計と比べられる。
3. EmDash. [Plugin sandbox](https://docs.emdashcms.com/deployment/plugin-sandbox/). EmDash Docs.
   - Cloudflare と Node.js での隔離の違い。
4. [Atproto Spaces Alpha](https://atproto.com/blog/atproto-spaces-alpha). AT Protocol Blog.
   - 有料プラグインを分散型で扱うために注目している仕組み。
