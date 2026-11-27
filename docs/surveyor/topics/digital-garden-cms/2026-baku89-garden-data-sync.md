# garden.ooo のデータアーキテクチャ、レプリケーション、ページスキーマ

baku89.com（Web ページ群、実装ドキュメント）, 2026

- https://baku89.com/ja/garden/data-architecture
- https://baku89.com/ja/garden/replication
- https://baku89.com/ja/garden/page-schema
- https://baku89.com/ja/garden/devlog

著者:

- baku89（橋本 麦、個人）

## どんなもの？

- garden.ooo の同期とデータモデルの実装ドキュメントである。ローカルのテキストファイルを SSOT とし、ブラウザ内の DB、サーバーの DB へ段階的にミラーする三層構造の動き方を、障害事例とあわせて記述している。
- 扱う問題は三つある。ファイルシステムとブラウザ DB の双方向同期でのエコーと競合、ブラウザ DB とサーバー DB のレプリケーションでの競合解決と取りこぼし防止、別端末や MCP による更新の通知である。

## 先行研究と比べてどこがすごいの？

- CouchDB 型の rev tree や HLC（ハイブリッド論理時計）を**採らない**判断を、理由つきで記している。単一ユーザーで共同同時編集を想定しないので、競合は `edited_at` の比較で決められ、因果順序を表す rev tree の豊かさは要らない、という論拠である。代わりにサーバーが受理した書き込みごとに単調増加の `seq` を振り、pull のカーソルを単一の整数にしている。
- クライアント時計に基づくカーソルの欠点も説明している。オフライン中に古い時計値で書かれた文書を、他端末の checkpoint が追い越して取りこぼす可能性がある。サーバーが受理時に `seq` を振れば、この取りこぼしは起きない。
- 変更判定に `mtime` も frontmatter の `updated` も使わず、コンテンツのハッシュを使う。`mtime` は Dropbox や iCloud による書き換え、エディタによる保持、NTP の補正で信頼できず、`updated` は著者が訪問者向けに付ける値だから、というのが理由である。

## 技術や手法の"キモ"はどこにある？

- **二本のレプリケーション**: `db.pages` に対して、Local File ⇆ Local DB と Local DB ⇆ Remote DB の二本の `replicateRxCollection` が同時に走る。`replicationIdentifier` で由来を区別するので、ファイルから取り込んだ文書はサーバーへ送るがファイルへは押し戻さない。
- **エコー防止**: ファイルごとに `lastSyncedHash` を持ち、これを唯一のフィルタにする。自分の書き込み直後、Dropbox による同内容の書き戻し、FileSystemObserver の連続発火は、どれもハッシュ一致で読み飛ばされる。
- **「無い」と「読めない」の区別**: ファイルが消えたときは `softDeleted: true` を流す。一方、ディレクトリハンドルが無い、権限が失効した、vault ごと消えたといった場合は `FsUnavailableError` として扱い、削除とはみなさない。
- **リネーム追跡**: 各ページは不変の `content_id`（ULID）を frontmatter に持ち、DB ではこれを主キーにする。編集とリネームを同時に行っても同じページとして追える。
- **競合解決**: push に `assumedMasterState` を付け、サーバーは `softDeleted`、`path`、`source_hash` の三つが一致したときだけ受理する。競合時は `edited_at` が新しい側を採る（Last-Writer-Wins）。例外は三つある。`source` を持たない側は勝てない。派生フィールドだけのずれは master の値を接ぎ木する。ユーザーが意図した削除は生きている master に勝つ。
- **変更通知**: サーバーは `GET /.garden/api/pages/events` で SSE を流す。イベントには見出しだけ（`seq`、`content_id`、`slug`、`source_hash`、`softDeleted`）を載せ、本文は載せない。SSE の `id:` を `seq` にしているので、再接続時に `Last-Event-ID` 以降を再送でき、切断中の取りこぼしがない。閲覧者には public / garden / unlisted のページの変更だけを通知し、非公開化や削除は通知しない。
- **マルチタブ協調**: ファイル同期を回すのは Web Lock を持つタブ一つだけにして、複数タブが同じファイルを同時に触る競合をなくしている。
- **ページスキーマ**: `PageDoc = {id, content, softDeleted}`。`content` は frontmatter、本文 AST、ソース、`source_hash`、`outlinks` を含む。frontmatter の一部（サイトタイトル、メニュー、テーマ、カスタム CSS など）はディレクトリ階層に沿って配下のページへ継承される。

## どうやって有効だと検証した？

- 本番運用で起きた障害と、その対策をドキュメントに残している。
  - slug が衝突した文書をサーバーが黙って拒否し、クライアントが同期済み扱いのまま再 push を永遠に繰り返した（2026-08-27）。対策として、サーバーが自動リネームした文書を競合として返し、クライアントに採用させるようにした。
  - DB を作り直した際に `seq` カウンタだけが 0 付近から再開し、生き残ったクライアントの checkpoint（50792）がカウンタ（25119）より先に進んでいた。その結果、pull が空になり push だけが通る片方向状態に陥った（2026-08）。対策として、checkpoint がカウンタより先にいたら古い世代とみなし、全件を再同期する。
  - 以前は常にサーバー側を勝たせていたが、古いサーバー状態がローカルを上書きし、ファイルまで巻き戻す事故が起きたため、`edited_at` による判定へ変えた。
- devlog（2026-03-17）には「同期実装がバイブス過ぎて、定期的に全ページの1割が勝手に消えるのを復元している」とあり、同期の不具合がデータ消失に直結した時期があったことがわかる。
- SSE は本番で `curl -N` により即座に届くことを確認している。

## 議論はあるか？

- 単一ユーザーかつ一度に一台で操作する前提なので、共同編集に広げるときは `edited_at` による Last-Writer-Wins では足りなくなる。
- 「同期インテント・ジャーナル」（vault 内 `.garden/` サイドカーで削除済みページの復活を防ぐ仕組み）は、索引ページに説明があるものの、本文は公開されていなかった。
- devlog の TODO には、管理者専用の `modified` フィールド、ブロック単位のバックリンクアンカー、ページ単位の changelog と RSS、サイトごとの秘密情報管理（Secret Registry）、複数の決済プロバイダを束ねるメンバーシップ基盤が並ぶ。

## 次に読むべき論文はあるか？

1. RxDB. [Replication](https://rxdb.info/replication.html). rxdb.info.
   - garden.ooo のレプリケーションの土台。checkpoint、`assumedMasterState`、conflict handler の仕様。
2. Ink & Switch (2019). [Local-first software: You own your data, in spite of the cloud](https://www.inkandswitch.com/local-first/). Ink & Switch.
   - ローカルを SSOT とする設計の背景。
3. baku89 (2026). [情報設計](https://baku89.com/ja/garden/information-architecture). baku89.com.
   - path / uri / slug の三層の識別子と、slug 衝突時の自動リネーム。公開範囲の定義もここにある（[2026-baku89-garden-visibility.md](2026-baku89-garden-visibility.md)）。
