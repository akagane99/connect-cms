# リポジトリガイドライン

## 基本方針

- Connect-CMS は Laravel 8 を基盤とするプラグイン型CMSです。このファイルをAIコーディングエージェント向けルールの正本とします。
- 回答は日本語で行います。コミットやPRに、特定のAI製品による生成を示す署名を付けません。
- 変更前に対象コード、同種の既存実装、設定、テストを確認し、既存の基底クラス、ヘルパー、Blade部品、実装パターンを優先して再利用します。
- 依頼された目的に必要な範囲だけを変更し、無関係な整形、命名変更、依存更新、リファクタリングを混在させません。
- PHPは `composer.json` の対応範囲である7.4～8.1で動作する構文を使い、サポート範囲の変更を伴わずにPHP 8専用構文を導入しません。
- 依頼がない限り、commit、push、PR作成、ファイルやworktreeの削除、DB初期化などの破壊的操作を行いません。

## プロジェクト構成

- `app/`: Laravelアプリ本体
- `app/Plugins/User/`, `Manage/`, `Api/`, `Mypage/`: 一般、管理、API、マイページの各プラグイン
- `app/Models/Common/`, `Core/`, `User/`: 共通、コア、ユーザプラグインのモデル
- `config/`, `routes/`, `resources/`, `public/`: 設定、ルート、Blade・ソースアセット、Webルート・ビルド済みアセット
- `database/migrations/`, `seeders/`, `factories/`: DB関連
- `tests/Unit/`, `Feature/`, `Browser/`: PHPUnitとLaravel Dusk
- `docker/`, `docker-compose.yml`: ローカル開発環境
- `storage/`: 実行時ファイル。ログ、キャッシュ、セッション、アップロード生成物、秘密情報をコミットしません。

## Connect-CMSプラグイン

- ユーザプラグインは原則として `app/Plugins/User/{Plugin}/{Plugin}Plugin.php` に置き、`UserPluginBase` を継承します。モデルは `app/Models/User/{Plugin}/`、Bladeは `resources/views/plugins/user/{plugin}/`、表示名は `plugin.ini` の既存構成に従います。
- 管理、API、マイページでは、それぞれ `ManagePluginBase`、`ApiPluginBase`、`MypagePluginBase` と同種の既存実装を確認します。
- ユーザプラグインのアクションはURLから動的に呼び出されます。アクション追加・変更時は、HTTPメソッドを定義する `config/cc_role.php` または `getPublicFunctions()`、権限を定義する `declareRole()`、対象投稿を取得する `getPost()` の必要性を確認します。
- `page_id`、`frame_id`、`bucket_id`、plugin名、投稿ID、upload IDをリクエスト値だけで信用せず、現在のpage/frame/plugin/bucketや所有者に結び付く条件をDBクエリへ含めます。
- プラグイン追加時は、プラグインクラス、`plugin.ini`、モデル、Blade、migration、seeder、factory、テストの必要性を確認します。
- ユーザプラグイン直下のサブディレクトリはテンプレート候補になります。テンプレート選択に出さない共通Bladeは `resources/views/plugins/user/{plugin}` 直下に置き、新しい `common/` ディレクトリを作りません。

## 実装・セキュリティ

- 認証・認可はUI表示だけに依存せず、直接リクエストでも保護されるようアクションとDBクエリの両方で確認します。GETで状態を変更しません。
- 入力は型、長さ、範囲、nullable、存在、所有関係を検証し、リクエスト全体を無条件にモデルへ渡しません。
- SQLはEloquentまたはバインドされたQuery Builderを優先し、入力値をSQL、列名、並び順へ文字列連結しません。
- Bladeは `{{ }}` によるエスケープを基本とし、`{!! !!}` は信頼済みまたはHTMLPurifier等で処理済みのHTMLに限定します。POSTフォームではCSRFを維持します。
- ファイル操作では、MIME、拡張子、サイズ、保存先、パストラバーサル、公開範囲、upload IDの所有関係、DBと実ファイルの整合性を確認します。
- URL、form action、asset、Ajaxのパスは `url()`、`route()`、既存の `UrlUtils` 等を使い、サブディレクトリ設置を壊すルート固定のパスを追加しません。
- 秘密情報、個人情報、token、未公開の脆弱性情報をログ、テストデータ、Issue、PRへ含めません。脆弱性の報告は `.github/CONTRIBUTING.md` から参照される公式手順に従います。

## DB・設定変更

- スキーマと既存データの変更は新しいmigrationで行い、共有済みのmigrationを後から書き換えません。
- migrationでは既存データ、`up()` / `down()`、nullable/default、index、論理削除、`created_id`、`updated_id` の既存パターンを確認します。不可逆な変更や長時間ロックの可能性がある処理は明示します。
- 複数テーブル、またはDBとファイルを一体で更新する場合は、transactionと途中失敗時の整合性を確認します。
- `configs` テーブルの設定追加時は、画面保存時の `updateOrCreate()` だけでよいか、新規インストール用seederや既存インストール用migrationも必要か確認します。
- 環境変数を追加・変更した場合は `.env.example` と関連設定を更新し、実環境の値は記録しません。

## Blade・JavaScript・CSS・依存関係

- JavaScript/Sassは `webpack.mix.js` のエントリと出力先を確認し、原則として `resources/js`、`resources/sass` を変更してビルドします。ビルド済みファイルを直接編集しません。
- アセット変更時は `npm run dev` または `npm run prod` を実行し、`public/mix-manifest.json` と変更されたビルド済みアセットをコミット対象に含めます。
- `.styleci.yml` と既存コードのスタイルに従います。`package.json` にローカルlintスクリプトはないため、存在しないコマンドを推測しません。
- PHPの本番用依存は `composer.json` / `composer.lock`、開発・テスト用依存は `composer-dev.json` / `composer-dev.lock` です。runtime依存は意図的な差を除き両方へ反映します。
- JavaScript依存変更では `package.json` と `package-lock.json` を整合させます。目的に無関係な一括更新やバージョン制約の緩和を行いません。

## セットアップ・検証

- ホスト環境の初期設定: `.env.example` を `.env` にコピーし、`php artisan key:generate`
- PHP依存: 本番用は `composer install`、開発・テスト用は環境変数 `COMPOSER` に `composer-dev.json` を指定して `composer install`
- フロントエンド依存・ビルド: `npm ci`、`npm run dev` / `npm run prod`
- PHPコード規約: `composer phpcs`、対象限定は `composer phpcs-any -- path/to/file`
- Unit・Feature: `composer phpunit`、対象限定は `composer phpunit -- tests/Feature/SomeTest.php`
- Dusk: `php artisan dusk`。PHPUnitを直接使う場合は `phpunit.dusk.xml` を指定します。
- `composer phpcs` は `phpcs.xml` により `config/`、`database/`、`public/`、`routes/`、`resources/` 等を除外します。除外されたPHP変更は、少なくとも `php -l path/to/file.php` とレビューで確認します。
- PHPUnitはMySQLと `RefreshDatabase` を使用するテストを含みます。専用テストDB以外では実行しません。現在、`phpunit.xml` のDB名 `test` とDocker初期化SQLの `connect_testing` が一致しないため、接続先を確認してから実行します。

Docker Composeでは、同じコマンドを `webapp` コンテナ内で実行します。Composeプロジェクト名が不要な環境では `--project-name <task-name>` を省略できます。

- 起動: `docker compose --project-name <task-name> up -d --build webapp mailhog`
- 初回セットアップ: `docker compose --project-name <task-name> exec -T webapp bash docker/webapp/setup.sh`
- 稼働確認: `docker compose --project-name <task-name> ps`
- 対象テスト例: `docker compose --project-name <task-name> exec -T webapp composer phpunit -- tests/Feature/SomeTest.php`
- Duskでは `--profile dusk` を付けて `dusk` サービスも起動します。

## テスト方針

- Unitは `tests/Unit`、Featureは `tests/Feature`、Duskは `tests/Browser` に置き、ファイル名を `*Test.php` とします。
- 新機能・不具合修正では変更した公開動作のテストを追加・更新し、不具合修正では可能な限り修正前に失敗する回帰テストを追加します。
- 認証・認可、page/frame/plugin/bucket境界、入力境界、既存データ更新、ファイル公開範囲を重点的に検証します。セキュリティ修正では許可される正常系と拒否される異常系をテストします。
- private実装を直接検査するより、route、publicメソッド、画面操作など公開された境界から検証します。
- テストクラスのコメントには対象と検証方針、テストメソッドのコメントには利用者から見た仕様や回帰させたくない動作を簡潔に書きます。privateな補助メソッドには補助内容と必要な理由だけを書きます。

## Git・PR

- デフォルトブランチは `master` です。依頼により作業ブランチを作る場合は目的が分かるケバブケース名を使い、`master` へ直接コミットしません。
- コミットは `fix:`、`feat:`、`refactor:`、`docs:` 等のConventional Commits形式とし、対象ファイルだけをステージングします。
- PR本文は `.github/PULL_REQUEST_TEMPLATE.md` の見出しと順序を維持し、概要、関連Issue、DB変更の有無、確認手順を記載します。
- 人手で作成するPRタイトルは、近年の既存PRに合わせて `[プラグイン名または対象領域] <利用者向けの過去形要約>` とします。UI変更時はスクリーンショットまたは操作手順を付けます。

## git worktreeを利用する場合

- 既にタスク用worktree内にいる場合は、さらにworktreeを作りません。新規作成が依頼された場合だけ、タスクごとのブランチとworktreeを用意します。
- 並行起動時は `APP_URL`、ホスト側ポート、Composeプロジェクト名を重複させません。コンテナ内の `DB_HOST=db` と `DB_PORT=3306` は変更しません。
- `storage/` と `vendor/` はworktree間で共有されません。DBデータは `docker/db/mysql_data` に残るため、初期化・削除・worktree削除前に必要性とバックアップを確認します。
- 他の開発環境にも影響する `docker system prune -a` は使用しません。
