# garden.ooo の公開範囲、永続ページ、ダダ漏れ

baku89.com（Web ページ群）, 2026

- https://baku89.com/ja/garden/information-architecture
- https://baku89.com/ja/garden/visibility
- https://baku89.com/ja/garden-pages
- https://baku89.com/ja/permanent
- https://baku89.com/ja/ダダ漏れ
- https://baku89.com/ja/garden/frontmatter

著者:

- baku89（橋本 麦、個人）

## どんなもの？

- 一つの vault に公的な文章と私的なメモを混在させたまま公開するための、公開範囲の設計とその思想をまとめたページ群である。
- 公開範囲は、閲覧者がどこから辿り着けるかで段階を分ける。サイト上の説明（庭先公開ページ）では次の 4 段階である。

| 公開範囲 | 検索エンジン | サイト内の一覧 | リンクを知っている人 |
|---|:-:|:-:|:-:|
| `public`（みんなに公開） | ○ | ○ | ○ |
| `garden`（庭先公開） | × | ○ | ○ |
| `unlisted`（限定公開） | × | × | ○ |
| `private`（自分だけ） | × | × | × |

- これと別軸に `permanent` フラグがある。永続ページ（`permanent: true`）は URL をできる限り固定し、自由に引用してよいものとする。note page（`permanent: false`）は `noindex, nofollow` で、いつでも消せる下書きとする。

## 先行研究と比べてどこがすごいの？

- 公開と非公開の二値ではなく、「一覧に出るか」「検索エンジンに載るか」「リンクを知っていれば読めるか」を分けている。`unlisted` は一覧やバックリンクに出さず、言及を辿った人だけが到達できる。作者はこれを、タイムラインのような文脈を欠いた一覧で異なる文脈のコンテンツが衝突する「文脈崩壊」を構造的に避ける仕組みとして位置づけている。
- 作者の比喩では、鍵付きの部室（Discord のプライベートサーバー）にはせず、「すこし心もとない衝立」にとどめる。興味を持って辿ってきた人だけが読める細い私道に思考を置いておく、という態度である。

## 技術や手法の"キモ"はどこにある？

- **既定は非公開**: `visibility` キーが最優先される。キーが無い場合、他の静的サイトジェネレーターとの互換のため、`publish: true` または `published: true` のときだけ `public` とし、それ以外は `private` とする。
- **frontmatter の互換**: Obsidian / Jekyll の `title`、`aliases`、`tags`、`date`、Obsidian Publish の `publish`、Jekyll の `published` を受け付けたうえで、`title_locale`、`visibility`、`theme`、`layout` などの独自フィールドを足す。
- **`permanent` の推定**: 明示しなくても、`layout` が `page` 以外のときや、特定のタグ（`works`、`video` など）が付いたときは `permanent: true` と推定する。
- **通知の公開範囲**: 閲覧者への変更通知は public / garden / unlisted のページに限り、非公開化や削除は通知しない（[2026-baku89-garden-data-sync.md](2026-baku89-garden-data-sync.md)）。非公開になった事実そのものを漏らさないためである。

## どうやって有効だと検証した？

- 作者のサイトで実際に運用されている。庭先公開のページはグレーの見た目で表示し、冒頭に「半公開の庭先公開ページです」と明記している。注意書きとして、検索エンジンには載らないこと、URL や内容や公開状況は気まぐれで変わること、正確性は保証しないことを読者に示している。

## 議論はあるか？

- 公開範囲の種類はページによって数が違い、整理の途中にある。情報設計のページは `public` / `unlisted` / `protected` / `private` の 4 段階、ページスキーマは `contextual` を加えた 5 段階、庭先公開ページは `garden` を含む 4 段階を挙げる。「公開範囲を再考」するページでは、archival フラグは `public` のさらに上位ではないか、リンクバック・クロール・直リンクの可否を別フラグに分けるべきかと検討している。
- `protected`（認証つき公開）はメンバーシップや有料配信を想定しているが、決済機能は未実装である。ページ単位の `protected` とは別に、本文の一部だけを支援者に見せる `::gated` ブロックが GitHub Sponsors 連携で試験運用されている（[2026-baku89-garden-members-only.md](2026-baku89-garden-members-only.md)）。
- 作者は「ダダ漏れ」のページで、私的な会話から考えたことまで「許されるか許されないかのギリギリのところ」で公開していると書いている。公開範囲の仕組みが技術的に正しくても、何を `garden` に置くかの判断は書き手に残る。

## 次に読むべき論文はあるか？

1. Olia Lialina. [Vernacular Web](http://art.teleportacia.org/observation/vernacular/). art.teleportacia.org.
   - 文脈崩壊を避ける個人ホームページ文化の参照先として、作者が挙げている。
2. 折登 樹 (2026). [デジタルガーデンについて思うこと](https://strata.orito-itsuki.graphics/thoughts-on-digital-garden). strata.orito-itsuki.graphics.
   - 庭を「public と private の中間領域」とみる同じ問題意識を、別の作者が述べている（[2026-orito-hyperstrata.md](2026-orito-hyperstrata.md)）。
