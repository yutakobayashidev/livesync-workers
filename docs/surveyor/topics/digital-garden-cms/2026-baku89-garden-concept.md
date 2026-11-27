# garden.ooo（概要、設計思想、思想的な背景、やらないこと、利用モード、価格設定）

baku89.com（Web ページ群）, 2026

- https://baku89.com/ja/garden
- https://baku89.com/ja/garden/design-principles
- https://baku89.com/ja/garden/references
- https://baku89.com/ja/garden/what-it-is-not
- https://baku89.com/ja/garden/use-mode
- https://baku89.com/ja/garden/pricing
- https://baku89.com/ja/garden/memo

著者:

- baku89（橋本 麦、個人）

## どんなもの？

- baku89 が開発している Local-First な CMS「garden.ooo」の構想と設計原則をまとめたページ群である。旧名は Morion。
- 実績サイト、ナレッジベース、日記、有料コンテンツ配信を一つのシステムで賄うことを狙う。作者のポートフォリオ（baku89.com）と友人のサイトで実際に使いながら開発している。
- 設計原則は次の 5 つである。
  1. **階層性と線形性を信じない**: 投稿タイプもカテゴリの木構造も持たない。コンテンツ同士の関係は「A が B に属する」という包含ではなく、「A が B を指し示す」というリンクだけで決める。
  2. **ローカル・ファースト**: ローカルのテキストファイルを SSOT とし、クラウドはそのミラーとする。
  3. **プレーンテキストは普遍的なインターフェース**: ソースは Obsidian 互換の Markdown と YAML frontmatter で、メモ帳でも `sed` でも AI でも扱える。
  4. **ダダ漏れをアフォードする**: 公的な文章から私的なメモまでを同じ場所に置き、多段階の公開範囲で成熟度のグラデーションを表す。
  5. **愚行権を認める**: パーサーをユーザーが差し替えられるようにし、サーバーは最低限のサニタイズしかしない。
- 「やらないこと」として 3 原則を掲げる。タイムラインを作らない（RSS リーダーに任せる）、通知をしない（フェッチのみ）、評価やリアクションの機能を自分で持たない（Webmention、はてブ、既存 SNS に外部化する）。

## 先行研究と比べてどこがすごいの？

- 先行例として Cosense（ページとタグを区別しない設計）、Obsidian（ローカル Markdown を SSOT とする思想）、Neocities（個人ホームページ文化の再興）を挙げ、それぞれから引き継いだ点を明示している。
- 一般的な CMS が「まず分類の箱を設計し、そこにコンテンツを入れる」のに対し、garden.ooo は全ページを同一スキーマで保存し、分類を tag link と body link で表す。作者は、`work` でもあり `event` でもあるコンテンツが現れた時点で投稿タイプの設計は破綻すると述べ、WordPress のタクソノミーにプロフィールを足せない例を挙げている。
- 利用モードを 3 段階に分け、アカウントなしで使い始められるようにしている。In-Browser mode はブラウザの IndexedDB だけで動き、Local mode はローカルの `.md` と同期し（そのフォルダを Obsidian の vault にもできる）、Server mode で初めてアカウントを作る。サーバーが必要になるのは、モバイル同期、Web 公開、共同編集のときだけという整理である。

## 技術や手法の"キモ"はどこにある？

- データは Local File（SSOT）↔ Local DB（RxDB / IndexedDB）↔ Remote DB（RxDB / MongoDB）の三層で持つ。ブラウザとローカルファイルの双方向同期には File System Access API と FileSystemObserver を使う（詳細は [2026-baku89-garden-data-sync.md](2026-baku89-garden-data-sync.md)）。
- コンテンツパイプラインは `source → parser → AST` である。サーバーには `{source, parsed_AST, ...metadata}` を送り、AST はクライアント側で生成する。
- Server mode の構想として、OAuth のみの認証、`foobar.garden.ooo` 形式のサブドメイン、元の `.md` をエンベロープ暗号で E2E 保管する案が挙がっている。
- 価格設定のメモでは、Browser / Local mode は無料、Pro Account（月 18 ドル）で独自ドメイン、Headless CMS 利用、`index.md` の非公開化、Patreon 等と連動した限定公開を提供する案がある。サービスへの貢献者には Pro と同等の権限を無償で与える。

## どうやって有効だと検証した？

- 定量評価はない。作者自身のサイトと友人のサイトで運用し続けることが検証にあたる（dogfooding）。
- 作者のサイトは、ポートフォリオ、Cosense、中学時代の FC2 ブログ、15 年分の WordPress サイトを統合したもので、移行メモには WordPress からの Markdown 書き出し手順が残っている。複数の既存メディアを一つのスキーマに収容できたことが、汎用スキーマ設計の実例になっている。

## 議論はあるか？

- Server mode の E2E 暗号化と Web 公開が両立するかは、ページ内で説明されていない。サーバーが公開用 HTML を出すには平文か AST が必要なので、何を暗号化し何を平文で持つかは未確定とみられる。
- カスタムパーサーは構想段階で、リッチテキスト編集（WYSIWYG）とは相性が悪く、ソースとプレビューの 2 ペイン編集に固定される可能性があると作者自身が書いている。
- 価格や法人向けの売り方はメモ段階で、製品として外部提供する段階には達していない。

## 次に読むべき論文はあるか？

1. Ink & Switch (2019). [Local-first software: You own your data, in spite of the cloud](https://www.inkandswitch.com/local-first/). Ink & Switch.
   - garden.ooo の「ローカル・ファースト」の出典。作者の Local-First Software ページからリンクされている。
2. Steph Ango. [File over app](https://stephango.com/file-over-app). stephango.com.
   - プレーンテキストを SSOT とする思想の参照先として、作者が挙げている。
3. Olia Lialina. [Vernacular Web](http://art.teleportacia.org/observation/vernacular/). art.teleportacia.org.
   - 個人ホームページ文化の美学をアーカイブした仕事。garden.ooo が再興を目指す「自分の場所を自分で作る」文化の参照先。
4. 橋本 翔. [階層整理型WiKiはスケールしない](https://scrapbox.io/shokai/%E9%9A%8E%E5%B1%A4%E6%95%B4%E7%90%86%E5%9E%8BWiKi%E3%81%AF%E3%82%B9%E3%82%B1%E3%83%BC%E3%83%AB%E3%81%97%E3%81%AA%E3%81%84). Cosense.
   - 「階層性と線形性を信じない」の出典。Sate の作者も同じ文章を参照している。
