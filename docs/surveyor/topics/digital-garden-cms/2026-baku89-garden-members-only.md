# garden.ooo のメンバー限定コンテンツ（GitHub Sponsors 連携の試験実装）

baku89.com（Web ページ群）, 2026

- https://baku89.com/ja/members-only
- https://baku89.com/ja/garden/manual/markdown-guide
- https://baku89.com/ja/garden/information-architecture
- https://baku89.com/ja/garden/devlog
- https://baku89.com/ja/garden/pricing

著者:

- baku89（橋本 麦、個人）

## どんなもの？

- garden.ooo に「メンバー限定コンテンツ」を試験的に実装したページ（「粗品の一覧」）と、その仕組みを説明するページ群である。作者は、友人のサイトの実装で必要になりそうだったので試したと書いている。
- 閲覧できるのは、過去に GitHub で公開スポンサーになった人、作者の展示準備の手伝いに応募した人、そのほか世話になった人である。作者に伝えたアカウントか、投げ銭をした GitHub アカウントでログインすると、各ページの「限定コンテンツ」ブロックの中身が見える。
- 作者は「オンラインサロンのようなことはしたくない」と明記し、支援者への粗品という位置づけにしている。

## 先行研究と比べてどこがすごいの？

- note や Substack のような有料配信は、記事単位かサイト全体で「払えば全部見える、払わなければ見えない」と区切る。garden.ooo は、公開ページの本文の中に限定ブロックを混ぜ込める。ページそのものは誰でも読め、その一部だけが支援者に開かれる。
- 支援の窓口を自前で持たず、GitHub Sponsors という既存の仕組みで支援した人を、GitHub の OAuth ログインで識別する。決済も会員管理も外部に任せている。

## 技術や手法の"キモ"はどこにある？

- **二つの粒度の閲覧制御**:
  - ページ単位: `visibility: protected`。認証（パスワードや OAuth）が必要なページで、メンバーシップ制の限定コンテンツや有料配信を想定している。
  - ブロック単位: 本文に `::gated` ブロックを書くと、メンバーにだけ中身を表示する。`::AdminOnly` ブロックは管理者にだけ表示し、メンバー限定としては扱わない。
- **配信経路の分岐**: 本文を単体の HTML として出す高速な経路（`layout: html`）が効くのは `public` と `unlisted` のページだけである。`private` と `protected` はアクセス判定がアプリ側にあるので、通常のサーバーサイドレンダリングを通る。
- **構想中の一般化**（devlog の TODO）: GitHub Sponsors に加えて Patreon、Gumroad、Stripe、Lemon Squeezy などを併用できるようにする。各サービスはアダプタとして実装し、結果をサービスに依存しない「Entitlement」（閲覧権）に正規化して、サーバー専用のコレクションに保存する。ページの frontmatter に `paywall: any_of: [...]` と書くと、複数サービスの閲覧権を OR で評価する。単品購入とサブスクリプションの両方を扱い、単品からサブスクへ段階的に関係を深める導線を想定している。各サービスの OAuth 認証情報は、別途構想している Secret Registry（サイトごとに暗号化して DB に保存する秘密情報の管理基盤）に載せる。
- **価格設定との関係**: Patreon などと連動した限定公開は Pro Account の機能とする案だが、GitHub Sponsors だけは無料プランでも使えるようにする方針がメモされている。

## どうやって有効だと検証した？

- 作者のサイトで試験運用している。作者は「見えるようになっていなかったら実装ミスなので連絡してほしい」と書いており、外部の利用者による動作確認を兼ねている段階である。

## 議論はあるか？

- 情報設計のページには「決済機能は未実装」とある。現状は「支援した事実」を GitHub ログインで確認するだけで、ページ単位の購入やサブスクリプションは構想段階である。
- 限定ブロックをページに混ぜる設計では、公開ページ用のキャッシュや静的出力に限定ブロックの中身を含めない必要がある。garden.ooo が `protected` を高速経路から外しているのはこのためとみられるが、`public` ページ内の `::gated` ブロックがどの経路で処理されるかは、今回の資料からは確認できなかった。

## 次に読むべき論文はあるか？

1. GitHub. [About GitHub Sponsors](https://docs.github.com/en/sponsors/getting-started-with-github-sponsors/about-github-sponsors). GitHub Docs.
   - 支援者の識別に使っている仕組みの公式説明。
2. baku89 (2026). [garden.ooo の公開範囲、永続ページ、ダダ漏れ](2026-baku89-garden-visibility.md).
   - ページ単位の公開範囲の全体像。
