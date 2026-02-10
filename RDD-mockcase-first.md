# 要件定義書（Laravel / バックエンド実装用）
- プロジェクト名：BookShelf
---
## 1. 概要
### 1.1 目的
- 本書は、Laravelでバックエンドを実装するために必要な仕様を定義する（フロントはBlade実装済み）。
- プログラミング初学者（学習時間300時間程度）が、Laravelの基本的な機能から応用的な機能までを網羅的に学習し、実務に近い形式でアプリケーション開発を体験することを目的とする。

### 1.2 前提・制約
- フレームワーク：Laravel 10.x
- フロント：Bladeは実装完了済みであり、本要件定義の対象外とする。
- 実装範囲：バックエンド（ルーティング、認証/認可、DB、ビジネスロジック、API等）

### 1.3 スコープ（対象 / 非対象）
- **対象**：
  - データベースの設計と構築（マイグレーション、シーディング）
  - ユーザー認証機能のバックエンド実装
  - 書籍、レビュー、ジャンルに関するCRUD機能のバックエンド実装
  - お気に入り、いいね機能のバックエンド実装
  - 応用機能（検索、ランキング、CSVエクスポート、外部API連携、レポート）のバックエンド実装
  - 上記機能を実現するためのAPI（Web, JSON）実装
- **非対象**：
  - BladeテンプレートのUI実装・修正
  - サーバーインフラの構築（Docker環境のセットアップ手順は「2. 環境構築」に記載）

---
## 2. 環境構築（必須）
### 2.1 動作要件
| コンポーネント | 技術/ツール | バージョン/種類 |
| :--- | :--- | :--- |
| OS | （Dockerが動作する任意のOS） | - |
| Webサーバー | Nginx | stable-alpine |
| アプリケーション | PHP | 8.2-fpm-alpine |
| | Laravel | 10.x |
| データベース | MySQL | 8.0 |

### 2.2 セットアップ手順
1. リポジトリ取得： `git clone` でリポジトリをクローン
2. 依存インストール：`composer install`
3. env作成：`.env.example` をコピーして `.env` を作成し、`php artisan key:generate` を実行
4. 起動：`docker-compose up -d --build`
5. migrate/seed：`docker-compose exec php php artisan migrate --seed` を実行
6. アクセス：ブラウザで `http://localhost` にアクセス

### 2.3 環境変数（最低限）
| Key | 必須 | 用途/意味 | 例 |
|---|---:|---|---|
| APP_NAME | ✅ | アプリケーション名 | BookShelf |
| APP_ENV | ✅ | 実行環境 | local |
| APP_KEY | ✅ | アプリケーション暗号化キー | `base64:...` |
| APP_DEBUG | ✅ | デバッグモード | true |
| APP_URL | ✅ | アプリケーションURL | http://localhost |
| DB_CONNECTION | ✅ | DB接続ドライバ | mysql |
| DB_HOST | ✅ | DBホスト | db |
| DB_PORT | ✅ | DBポート | 3306 |
| DB_DATABASE | ✅ | DB名 | bookshelf_db |
| DB_USERNAME | ✅ | DBユーザー名 | user |
| DB_PASSWORD | ✅ | DBパスワード | password |
| GOOGLE_BOOKS_API_KEY | △ | Google Books APIキー | (空でも動作可) |

---
## 3. 機能一覧
### 基本機能
| No | 機能名 | 詳細 |
|---|---|---|
| 1 | ユーザー認証 | ユーザーがシステムに登録・ログイン・ログアウトするための機能。 |
| 2 | 書籍管理 (CRUD) | ユーザーが書籍情報を登録、閲覧、更新、削除する機能。 |
| 3 | レビュー管理 (CRUD) | ユーザーが書籍に対してレビュー（評価・コメント）を投稿、編集、削除する機能。 |
| 4 | ジャンル管理 (CRUD) | 管理者が書籍に付与する「ジャンル」を作成、更新、削除できる機能。 |
| 5 | お気に入り機能 | ユーザーが気に入った書籍をブックマークとして登録・解除できる機能。 |
| 6 | いいね機能 | ユーザーが他のユーザーのレビューに対して「いいね」を付けたり外したりできる機能。 |

### 応用機能
| No | 機能名 | 詳細 |
|---|---|---|
| 7 | 書籍検索・絞り込み・並び替え | キーワード、ジャンル、並び順を指定して書籍を多角的に検索できる機能。 |
| 8 | 評価ランキング | レビューの平均評価点が高い書籍のトップ10をランキング形式で表示する機能。 |
| 9 | CSVエクスポート | 書籍一覧の表示結果をCSVファイルとしてダウンロードできる機能。 |
| 10 | ISBN書籍検索 | 書籍登録時にISBNを入力すると、外部APIから書籍情報を自動で取得・入力する機能。 |
| 11 | マイ読書レポート | 自身の読書活動（レビュー数、評価分布など）をグラフで可視化して振り返る機能。 |

---
## 4. データ設計（DB）
### 4.1 ER図
```mermaid
erDiagram
    users ||--o{ books : "登録する"
    users ||--o{ reviews : "投稿する"
    users ||--o{ favorites : "お気に入り登録する"
    users ||--o{ review_likes : "いいねする"

    books ||--|{ book_genre : "属する"
    books ||--o{ reviews : "投稿される"
    books ||--o{ favorites : "お気に入り登録される"

    genres ||--|{ book_genre : "属する"

    reviews ||--o{ review_likes : "いいねされる"

    users {
        bigint id PK
        varchar(255) name
        varchar(255) email UK
        timestamp email_verified_at
        varchar(255) password
        varchar(100) remember_token
        timestamps
    }

    books {
        bigint id PK
        foreignId user_id FK
        varchar(255) title
        varchar(255) author
        varchar(13) isbn UK
        date published_date
        text description
        varchar(255) image_url
        timestamps
    }

    genres {
        bigint id PK
        varchar(255) name UK
        timestamps
    }

    reviews {
        bigint id PK
        foreignId user_id FK
        foreignId book_id FK
        tinyint rating
        text comment
        timestamps
    }

    book_genre {
        bigint id PK
        foreignId book_id FK
        foreignId genre_id FK
        timestamps
    }

    favorites {
        bigint id PK
        foreignId user_id FK
        foreignId book_id FK
        timestamps
    }

    review_likes {
        bigint id PK
        foreignId user_id FK
        foreignId review_id FK
        timestamps
    }
```

### 4.2 テーブル定義
#### `users` テーブル
- **マイグレーション**: Laravel標準の `create_users_table` を利用。
- **役割**: ユーザーアカウント情報を格納する。

#### `books` テーブル
- **マイグレーション**: `create_books_table`
- **役割**: ユーザーが登録した書籍情報を格納する。
- **スキーマ**:
  - `id()`: 主キー
  - `foreignId('user_id')->constrained()->cascadeOnDelete()`: 外部キー (users)
  - `string('title')`: タイトル
  - `string('author')`: 著者
  - `string('isbn', 13)->nullable()->unique()`: ISBN
  - `date('published_date')->nullable()`: 出版日
  - `text('description')->nullable()`: 書籍説明
  - `string('image_url')->nullable()`: 書影URL
  - `timestamps()`: 作成日時、更新日時

#### `genres` テーブル
- **マイグレーション**: `create_genres_table`
- **役割**: 書籍のカテゴリを管理するマスターテーブル。
- **スキーマ**:
  - `id()`: 主キー
  - `string('name')->unique()`: ジャンル名
  - `timestamps()`: 作成日時、更新日時

#### `reviews` テーブル
- **マイグレーション**: `create_reviews_table`
- **役割**: 書籍に対するレビュー情報を格納する。
- **スキーマ**:
  - `id()`: 主キー
  - `foreignId('user_id')->constrained()->cascadeOnDelete()`: 外部キー (users)
  - `foreignId('book_id')->constrained()->cascadeOnDelete()`: 外部キー (books)
  - `unsignedTinyInteger('rating')`: 評価 (1-5)
  - `text('comment')->nullable()`: コメント
  - `unique(['user_id', 'book_id'])`: 複合ユニーク制約
  - `timestamps()`: 作成日時、更新日時

#### `book_genre` テーブル
- **マイグレーション**: `create_book_genre_table`
- **役割**: `books`と`genres`の多対多リレーションを定義する中間テーブル。
- **スキーマ**:
  - `foreignId('book_id')->constrained()->cascadeOnDelete()`: 外部キー (books)
  - `foreignId('genre_id')->constrained()->cascadeOnDelete()`: 外部キー (genres)
  - `primary(['book_id', 'genre_id'])`: 複合主キー

#### `favorites` テーブル
- **マイグレーション**: `create_favorites_table`
- **役割**: `users`と`books`の多対多リレーション（お気に入り）を定義する中間テーブル。
- **スキーマ**:
  - `foreignId('user_id')->constrained()->cascadeOnDelete()`: 外部キー (users)
  - `foreignId('book_id')->constrained()->cascadeOnDelete()`: 外部キー (books)
  - `primary(['user_id', 'book_id'])`: 複合主キー

#### `review_likes` テーブル
- **マイグレーション**: `create_review_likes_table`
- **役割**: `users`と`reviews`の多対多リレーション（いいね）を定義する中間テーブル。
- **スキーマ**:
  - `foreignId('user_id')->constrained()->cascadeOnDelete()`: 外部キー (users)
  - `foreignId('review_id')->constrained()->cascadeOnDelete()`: 外部キー (reviews)
  - `primary(['user_id', 'review_id'])`: 複合主キー

### 4.3 モデル定義
#### `User` モデル
- `books()`: `hasMany(Book::class)`
- `reviews()`: `hasMany(Review::class)`
- `favoriteBooks()`: `belongsToMany(Book::class, 'favorites')`
- `likedReviews()`: `belongsToMany(Review::class, 'review_likes')`

#### `Book` モデル
- `protected $fillable = ['user_id', 'title', 'author', 'isbn', 'published_date', 'description', 'image_url'];`
- `user()`: `belongsTo(User::class)`
- `reviews()`: `hasMany(Review::class)`
- `genres()`: `belongsToMany(Genre::class, 'book_genre')`
- `favoritedByUsers()`: `belongsToMany(User::class, 'favorites')`

#### `Genre` モデル
- `protected $fillable = ['name'];`
- `books()`: `belongsToMany(Book::class, 'book_genre')`

#### `Review` モデル
- `protected $fillable = ['user_id', 'book_id', 'rating', 'comment'];`
- `user()`: `belongsTo(User::class)`
- `book()`: `belongsTo(Book::class)`
- `likedByUsers()`: `belongsToMany(User::class, 'review_likes')`

---
## 5. 初期データ（シーディング）
#### `UserSeeder`
- `User::factory()->count(10)->create()` を使用し、テストユーザーを10件作成する。

#### `GenreSeeder`
- `Genre::create()` を使用し、以下の10件のジャンルを登録する。
  - '文学・小説', 'ビジネス・経済', 'コンピュータ・IT', '趣味・実用', '漫画', 'ライトノベル', '絵本・児童書', '雑誌', '学術・専門書', 'その他'

#### `BookSeeder`
- `Book::factory()->count(50)->create()` を使用して、ダミーデータを50件作成する。
- 各書籍には、`afterCreating` state を利用して、1〜3個のランダムなジャンルを紐付ける。

#### `ReviewSeeder`
- `Review::factory()->count(200)->create()` を使用して、ダミーデータを200件作成する。

#### `DatabaseSeeder`
- `run` メソッド内で `$this->call()` を使用し、`UserSeeder`, `GenreSeeder`, `BookSeeder`, `ReviewSeeder` の順で呼び出す。

---
## 6. ルーティング
### `routes/web.php`
| Method | Path | Controller@Action | Route Name | Middleware |
|---|---|---|---|---|
| GET | `/` | `BookController@index` | `home` | - |
| GET | `/books` | `BookController@index` | `books.index` | - |
| GET | `/books/create` | `BookController@create` | `books.create` | `auth` |
| POST | `/books` | `BookController@store` | `books.store` | `auth` |
| GET | `/books/{book}` | `BookController@show` | `books.show` | - |
| GET | `/books/{book}/edit` | `BookController@edit` | `books.edit` | `auth` |
| PUT | `/books/{book}` | `BookController@update` | `books.update` | `auth` |
| DELETE | `/books/{book}` | `BookController@destroy` | `books.destroy` | `auth` |
| GET | `/books/export/csv` | `BookController@exportCsv` | `books.export` | `auth` |
| GET | `/books/fetch` | `BookController@fetch` | `books.fetch` | `auth` |
| GET | `/genres` | `GenreController@index` | `genres.index` | `auth` |
| POST | `/genres` | `GenreController@store` | `genres.store` | `auth` |
| PUT | `/genres/{genre}` | `GenreController@update` | `genres.update` | `auth` |
| DELETE | `/genres/{genre}` | `GenreController@destroy` | `genres.destroy` | `auth` |
| POST | `/books/{book}/reviews` | `ReviewController@store` | `reviews.store` | `auth` |
| GET | `/reviews/{review}/edit` | `ReviewController@edit` | `reviews.edit` | `auth` |
| PUT | `/reviews/{review}` | `ReviewController@update` | `reviews.update` | `auth` |
| DELETE | `/reviews/{review}` | `ReviewController@destroy` | `reviews.destroy` | `auth` |
| GET | `/favorites` | `FavoriteController@index` | `favorites.index` | `auth` |
| POST | `/books/{book}/favorite` | `FavoriteController@toggle` | `favorites.toggle` | `auth` |
| POST | `/reviews/{review}/like` | `ReviewLikeController@toggle` | `reviews.like` | `auth` |
| GET | `/ranking` | `RankingController@index` | `ranking.index` | - |
| GET | `/reports` | `ReportController@index` | `reports.index` | `auth` |

*その他、認証関連のルートが `routes/auth.php` に定義される (Laravel Breeze)*

### `routes/api.php`
| Method | Path | Controller@Action | Route Name | Middleware |
|---|---|---|---|---|
| GET | `/v1/books` | `Api\V1\BookController@index` | `api.v1.books.index` | - |
| GET | `/v1/books/{book}` | `Api\V1\BookController@show` | `api.v1.books.show` | - |

---
## 7. バリデーション (FormRequest)
#### `StoreBookRequest`
- **rules()**:
  - `title`, `author`: `required|string|max:255`
  - `isbn`: `nullable|string|size:13|unique:books`
  - `published_date`: `nullable|date`
  - `description`: `nullable|string`
  - `image_url`: `nullable|url|max:255`
  - `genres`: `required|array|min:1`
  - `genres.*`: `exists:genres,id`
- **messages()**: 各ルールに対応する日本語のエラーメッセージを定義する。

#### `UpdateBookRequest`
- `StoreBookRequest` を継承。
- `isbn` のユニークチェックを `Rule::unique(\'books\')->ignore($this->book->id)` に変更する。

#### `StoreGenreRequest`
- `name`: `required|string|max:255|unique:genres`

#### `UpdateGenreRequest`
- `name`: `required|string|max:255|unique:genres,name,{$this->genre->id}`

#### `StoreReviewRequest`
- `rating`: `required|integer|min:1|max:5`
- `comment`: `nullable|string|max:1000`

#### `UpdateReviewRequest`
- `StoreReviewRequest` と同様のルールを定義する。

---
## 8. コントローラーロジック詳細
### `BookController@index` (書籍一覧)
- クエリパラメータ `keyword`, `genre`, `sort` を受け取る。
- `Book::with(\'genres\', \'reviews\')` でクエリを開始する。
- 各クエリパラメータが存在する場合、`if`文で動的に検索条件（`where`句、`orderBy`句）を構築する。
  - `keyword`: `title`と`author`カラムを対象に`orWhere`で部分一致検索を行う。
  - `genre`: `whereHas`を使用してジャンルで絞り込む。
  - `sort`: `match`式で`latest`, `oldest`, `title`, `rating`の各ケースに応じた`orderBy`を適用する。`rating`の場合は`withAvg`と`orderBy`を組み合わせる。
- `paginate(10)->withQueryString()`で結果を取得し、ビューに渡す。

### `BookController@store` (書籍登録)
- `StoreBookRequest`でバリデーションを行う。
- `DB::transaction()`を使用して、書籍登録とジャンル紐付けをアトミックに行う。
- `Auth::user()->books()->create()`で認証ユーザーに紐づけて書籍を保存する。
- `$book->genres()->attach($request->genres)`でジャンルを紐付ける。
- 成功後、`books.show`へリダイレクトする。

### `BookController@exportCsv` (CSVエクスポート)
- `BookController@index`と同様のロジックで検索クエリを構築する。
- `latest()->get()`で全件取得する。
- `response()->streamDownload()`を使用してCSVをストリーム出力する。
  - ファイル名のフォーマットは`books_{Ymd_His}.csv`とする。
  - `fwrite($handle, "\xEF\xBB\xBF");`でBOMを先頭に書き込む。
  - ヘッダー行を書き込んだ後、`foreach`で1行ずつデータを`fputcsv()`で書き込む。ジャンルは`implode`で連結する。

### `BookController@fetch` (ISBN書籍検索)
- クエリパラメータ`isbn`を受け取る。
- `Http::get()`でGoogle Books API (`https://www.googleapis.com/books/v1/volumes`) を呼び出す。
- レスポンスをチェックし、書籍が見つかれば必要な情報（タイトル、著者、出版日、説明、書影URL）を抽出し、JSONで返す。
- API通信失敗時や書籍が見つからない場合は、適切なエラーステータスコードとメッセージを返す。

### `ReportController@index` (マイ読書レポート)
- `Auth::user()->reviews()->with(\'book.genres\')->get()`でログインユーザーの全レビューを取得する。
- Laravel Collectionのメソッドを駆使して以下の統計データを算出する。
  1. **基本サマリー**: `count`, `pluck`+`unique`+`count`, `avg`
  2. **評価分布**: `countBy(\'rating\')`
  3. **高評価書籍TOP5**: `where(\'rating\', \'>=\', 4)`+`sortByDesc(\'created_at\')`+`take(5)`
  4. **ジャンル別評価傾向TOP5**: `flatMap`で全ジャンルを抽出し、`countBy`で集計後、`sortByDesc`+`take(5)`
- 算出したデータをビューに渡す。

---
## 9. 認証・認可
- **認証方式**: セッションベース認証（Laravel Breezeを利用）。
- **アクセス制御**: ルーティング定義 (`routes/web.php`) にて、認証が必要なルートに`auth`ミドルウェアを適用する。
- **認可方式**: ポリシー（Policy）を利用する。
  - `BookPolicy`: 書籍の`update`, `delete`は、書籍を登録したユーザー本人にのみ許可する。
  - `ReviewPolicy`: レビューの`update`, `delete`は、レビューを投稿したユーザー本人にのみ許可する。
- **権限NG時の挙動**: `auth`ミドルウェアにより、未認証ユーザーはログインページへリダイレクトされる。ポリシーによる認可失敗時は403エラーページが表示される。

---
## 10. 受け入れ条件
- **主要なユーザーフロー**：
  1. ユーザーが会員登録・ログインできる。
  2. ログイン後、書籍を登録・編集・削除できる。また、他人の書籍は編集・削除できない。
  3. 書籍詳細ページでレビューを投稿・編集・削除できる。また、他人のレビューは編集・削除できない。
  4. 書籍一覧ページで、キーワード検索、ジャンル絞り込み、並び替えが機能し、結果がCSVでエクスポートできる。
  5. 管理者としてジャンルをCRUDできる。
  6. 応用機能（ランキング、ISBN検索、読書レポート）が仕様通りに動作する。
- **成功条件**：
  - 上記フローがエラーなく完了し、DBの状態が期待通りに更新されていること。
  - 各種バリデーションが正しく機能し、不正なデータは登録されないこと。
- **エラー条件**：
  - 必須項目未入力でフォームを送信した場合、バリデーションエラーが返されること。
  - 未ログイン状態で認証必須ページにアクセスしようとした場合、ログインページにリダイレクトされること。
  - 他人のリソースを不正に操作しようとした場合、403エラーが表示されること。
