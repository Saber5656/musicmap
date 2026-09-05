# Review resolution contract

Repository: Saber5656/musicmap; PR #35

このファイルは既存のBot review findingに対する文書レベルの対応契約である。各節のresolutionは後続実装が満たすべき規範であり、focused verificationはresolve前に実装時点で実施する検証条件を示す。ここで実装・テスト・CI・実機検証を実行済みとは主張しない。Bot reviewの再triggerは行わず、repository full validationは後続の実装gateで実施する。

## Thread PRRT_kwDOTNkHUc6QBATV

### Preserve event ownership when day imports supersede

**Normative resolution**

日単位の再取込が既存eventを更新するときは、最新importをrowの所有者として原子的に移管するか、import-event associationを別表で保持する。旧import削除は最新importが供給するeventを削除してはならず、統計・enrichment対象も現行所有関係から再計算する。

**Focused verification before resolving this thread:**

同一dayの旧importと新importを取り込み、新importがplay_count/ms_playedを更新した後に旧importだけを削除する。eventが残り、import_id/associationが新importを指し、statsとenrichment対象が失われないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkHUc6QBATW

### Force floating-point coverage division

**Normative resolution**

durationCoverageは整数同士のSQLite除算を避け、分子または分母を1.0倍/castして実数として計算する。duration合計が0の場合の定義も固定し、mixed-duration bucketで小数値を保持する。

**Focused verification before resolving this thread:**

covered=1、total=2のbucketとtotal=0のbucketをfixtureで集計し、前者が0.5相当、後者が定義済みの安全な値になり、整数切捨てやdivision errorがないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkHUc6QBATa

### Add changeOrigin to the Vite API proxy

**Normative resolution**

開発用Vite proxyはobject形式でbackend targetとchangeOrigin=trueを明示する。これにより127.0.0.1:5173からbackend許可Hostへ転送し、Host allowlistと整合させる。

**Focused verification before resolving this thread:**

Vite dev server経由で/apiを呼び、Fastify側でbackend portのHostとして受理されること、直接の不許可Hostは従来どおり拒否されることをproxy integration testで確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkHUc6QBATd

### Remove schema_migrations from the 001 DDL block

**Normative resolution**

schema_migrationsの所有者をmigration runnerに一本化する。runnerが先に作るtableを001_init.sqlへ重複定義せず、DDL blockとmigration documentationの両方でrunner-ownedであることを明記する。

**Focused verification before resolving this thread:**

空DBに対して全migrationを実行し、schema_migrationsのtable-already-existsが発生しないこと、runnerがversion/name/applied_atを記録し、001の他tableが一度だけ作られることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkHUc6QBATh

### Query the schema_migrations version column

**Normative resolution**

status commandはschema_migrations.versionをMAXで取得し、bundled migration versionと比較する。appliedという存在しない列を参照せず、未適用・最新・欠損状態をread-onlyで明示する。

**Focused verification before resolving this thread:**

version/name/applied_atだけを持つfixture DBでstatusを実行し、MAX(version)と期待されるmigration状態が返ること、DBへの書込みがないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkHUc6QBATk

### Wait for enrichment to stop before deletion

**Normative resolution**

enrichment無効化だけをdelete-dataの前提にせず、runnerをquiesceしてin-flight requestの完了・cancelを待ち、書込みが停止した後に関連cacheを削除する。再開時は新しい世代だけを処理する。

**Focused verification before resolving this thread:**

enrichment requestを遅延させた状態でdisableとdeleteを同時に行い、削除後にartist_enrichment、artist_genres、artworkが再出現しないこと、runnerがquiesced/再開状態を正しく遷移することを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkHUc6QBATq

### Join the artwork cache path correctly

**Normative resolution**

artworkDirとbasename(file_path)を中央path helperとpath.join相当で結合する。file_pathのdirectory componentは常に破棄し、artworkDir外へのpath traversalやseparator欠落を許さない。

**Focused verification before resolving this thread:**

末尾separator有無、basenameのみ、../を含むfile_pathで期待ディレクトリ直下のpathになることを確認し、生成pathがartworkDir外へ出ないことをpath containment testで検証する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkHUc6QBATs

### Clean up duplicate CLI uploads

**Normative resolution**

CLI importでduplicateを検出した場合、DB rowを作らないだけでなく、その処理で作成したimports/<newId>のraw directoryを原子的に削除する。既存importのraw dataは変更しない。

**Focused verification before resolving this thread:**

同じexportを二度CLI投入し、二回目がduplicate結果を返してnewIdのraw directoryを残さないこと、既存importとPIIを含むraw exportが保持されることを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkHUc6QBATx

### Allow the test seam to dispatch to local HTTP mocks

**Normative resolution**

safeFetchのproduction URL policyはHTTPSを要求し、test seamが書き換えるloopbackのHTTP URLだけを明示的なtest mode/providerで許可する。rewrite前後の検証順序と、本番設定からtest exemptionへ到達できない境界を契約化する。

**Focused verification before resolving this thread:**

production設定でhttp://127.0.0.1以外とhttp gatewayを拒否し、test-only overrideで127.0.0.1:<port>のmockだけがdispatchされること、Authorizationが外部HTTPへ送られないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkHUc6QBATz

### Avoid writing settings from read-only servers

**Normative resolution**

read-only serveのGET /api/settingsは未作成のdisplay.timezoneを永続化しない。writable startupで既定値を先にseedするか、read-only側はdefaultを返して警告を維持する。GETがDB writeを発生させないことを保証する。

**Focused verification before resolving this thread:**

primary未起動でread-only instanceからsettingsを取得し、既定値が返りDBが変更されないことを確認する。writable instanceのseed後は同じ値を返し、並列GETでもwrite errorがないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkHUc6QBAT4

### Make the typecheck script check referenced projects

**Normative resolution**

typecheck scriptはsolution rootのfiles=[]に依存せず、参照されるwebとbuild projectを明示してtsc -bまたは同等のproject references buildを実行する。

**Focused verification before resolving this thread:**

各参照projectに意図的な型エラーを入れたfixtureでscriptが失敗し、エラー除去後に両projectを検査して成功することを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkHUc6QBAT8

### Garbage-collect identities after failed imports

**Normative resolution**

flush後にparserが失敗した場合、当該importのplay_eventsを削除した後、通常のimport deletionと同じidentity GCを実行する。失敗import由来の孤立artist/trackはenrichmentと集計へ流さない。

**Focused verification before resolving this thread:**

複数batchをflushしてからparser errorを注入し、play_events・未参照identity・enrichment queueが全てcleanになり、別importの共有identityは削除されないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。

## Thread PRRT_kwDOTNkHUc6QBAUB

### Remove partial events during interrupted-import recovery

**Normative resolution**

interrupted importのrecoveryはfailedへmarkするだけでなく、既にflushされた当該importのplay_eventsを削除し、identity GCとaggregate refreshを同一transaction/再実行安全な手順で行う。retryがpartial dataをduplicateと誤認しない状態に戻す。

**Focused verification before resolving this thread:**

process kill相当の状態を作ってrecoveryを実行し、partial eventsがstatsから消え、identity GC/aggregate refresh後に同一exportをretryできること、recoveryを二度実行しても壊れないことを確認する。

**Status boundary**

このaddendumは設計・受入契約の記録であり、実装または検証の完了を意味しない。実装後にrepository所定のfull validationと必要なsecurity/QA gateを実施する。
