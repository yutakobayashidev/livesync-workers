# 自作CMS仲間

baku89.com（Web ページ、リンク集）, 2026（最終更新 2026-11-02）

https://baku89.com/ja/自作CMS仲間

- baku89（橋本 麦、個人）

## どんなもの？

- CMS を自作している人とそのツールを集めたリンク集である。ここでの CMS は「Web ページ生成に限らない Content Management System」と緩く定義され、Markdown からの静的サイトジェネレーターやアウトラインツールも含む。
- 掲載されているのは次のツールである。
  - ドミニク・チェンさんらの「Pickles」: 既存の CMS に被せて走る bot 的な存在。現状は Notion API と連動する。
  - あずまさんの「Sate」: 書くことに集中するためのテキストエディタ（[2026-azuma-sate.md](2026-azuma-sate.md)）。
  - kdia さんの「loam」
  - 折登 樹さんの「Hyperstrata」（[2026-orito-hyperstrata.md](2026-orito-hyperstrata.md)）
  - 永良 新さんの「intertex」（[2026-nagara-intertex.md](2026-nagara-intertex.md)）
  - shivaduke さんの「bayou」（cairn に改名した可能性がある）
  - hashrock さんの「edane」
  - baku89 の「garden.ooo」
- 別枠として、CMS やエディタというより個人ホームページ的な運用をしているサイトを挙げる。加藤 淳さんの自作編集画面（[2026-kato-pagecms-and-my-cms.md](2026-kato-pagecms-and-my-cms.md)）、ひらのりょうさん、関口和希さん、菅 俊一さんのサイトなどである。

## 先行研究と比べてどこがすごいの？

- 個別のツール紹介ではなく、同時期に CMS やエディタを自作している人たちを一つの動向として並べている点に価値がある。作者は冒頭で「CMS の自作が今アツい」と書いている。

## 技術や手法の"キモ"はどこにある？

- リンク集なので、技術の詳細はリンク先にある。今回リンク先を辿った範囲では、各ツールに共通点が二つ見られる。
  - データの実体をプレーンテキスト（Markdown や JSON）に置き、ツールを乗り換えられるようにしている。
  - 書く・考える道具と、公開する仕組みを分けている。Sate は公開ボタンに任意のコマンドを割り当てるだけで、公開の中身は利用者が作る。intertex は共有用に PHP 1 枚と JSON ファイルのサーバーを持つ。

## どうやって有効だと検証した？

- 該当しない（リンク集）。

## 議論はあるか？

- loam は Scrapbox 上のページで、今回の取得では本文が得られなかった。bayou / cairn と edane もリンク先に詳細がなく、確認できていない。

## 次に読むべき論文はあるか？

1. あずま. [sngazm/sate](https://github.com/sngazm/sate). GitHub.
   - 書く道具と公開の仕組みを分ける設計の例。
2. 永良 新 (2026). [Intertex](https://aratanagara.com/intertex/). aratanagara.com.
   - 一つの木を三つの射影で編集するメタテキストエディタ。
3. 折登 樹 (2026). [Hyperstrata: 堆積するノートによるWebサイト形式](https://strata.orito-itsuki.graphics/introduction-to-hyperstrata). strata.orito-itsuki.graphics.
   - デジタルガーデンへの代案としての個人サイト形式。
4. 加藤 淳 (2026). [PX激変期、ブログを編集しやすくした](https://blog.junkato.jp/ja/posts/2026-09-28-pagecms-and-my-cms/). blog.junkato.jp.
   - Cloudflare Workers 上の自作編集画面。
