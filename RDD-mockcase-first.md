# 要件定義書（Laravel / バックエンド実装用） - BookShelf
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
  - 応用機能（検索、ランキング）のバックエンド実装
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

---
## 3. 機能要件詳細

| No | 機能名 | ID | ワークフロー | 概要 | 仕様・条件 | 期待結果 | 実装方針 |
|:---|:---|:---|:---|:---|:---|:---|:---|
| 1 | **ユーザー認証** | BF01 | 登録画面表示 | 新規ユーザー登録フォームを表示する。 | ・URL: `/register` (GET)<br>・認証: 不要（未ログイン時のみ） | ・登録フォームが表示される | ・Laravel Breezeの標準機能を利用<br>・`routes/auth.php` で定義 | 
| | | BF02 | ユーザー登録処理 | 入力された情報で新規ユーザーを登録する。登録後は書籍一覧へリダイレクトする。 | ・URL: `/register` (POST)<br>・パラメータ: `name`, `email`, `password`, `password_confirmation`<br>・バリデーション: `name` (必須, string, max:255), `email` (必須, string, email, max:255, unique:users), `password` (必須, 8文字以上, 確認一致) | ・`users`テーブルにデータが作成される<br>・パスワードはハッシュ化される<br>・自動的にログイン状態になる<br>・書籍一覧 (`/`) へリダイレクトされる | ・Breezeの`RegisteredUserController@store`が処理<br>・`User`モデルの`create`メソッドで保存 | 
| | | BF03 | ログイン画面表示 | ログインフォームを表示する。 | ・URL: `/login` (GET)<br>・認証: 不要（未ログイン時のみ） | ・ログインフォームが表示される | ・Laravel Breezeの標準機能を利用<br>・`routes/auth.php` で定義 | 
| | | BF04 | ログイン処理 | メールアドレスとパスワードで認証を行う。認証成功時は書籍一覧へリダイレクトする。 | ・URL: `/login` (POST)<br>・パラメータ: `email`, `password`, `remember`<br>・バリデーション: `email` (必須, email), `password` (必須) | ・認証成功: セッションが開始され、書籍一覧 (`/`) へリダイレクト<br>・認証失敗: エラーメッセージが表示され、ログイン画面に戻る | ・Breezeの`AuthenticatedSessionController@store`が処理<br>・リダイレクト先は`RouteServiceProvider`で設定 | 
| | | BF05 | ログアウト処理 | ログアウトしてセッションを破棄する。ログアウト後はトップページにリダイレクトする。 | ・URL: `/logout` (POST)<br>・認証: 必要 | ・セッションが破棄される<br>・トップページ (`/`) にリダイレクトされる | ・Breezeの`AuthenticatedSessionController@destroy`が処理<br>・`Auth::logout()` と `session()->invalidate()` を実行 | 
| 2 | **書籍管理** | BF06 | 一覧表示 | 登録されている書籍を一覧表示する。10件ごとにページネーションを実装する。 | ・URL: `/` または `/books` (GET)<br>・認証: 不要<br>・クエリパラメータ: `page` | ・書籍一覧が登録の新しい順に表示される<br>・10件ごとにページネーションリンクが表示される | ・`BookController@index`<br>・`Book::with("genres")->latest()->paginate(10)`<br>・ビューに`$books`を渡す | 
| | | BF07 | 検索機能 | キーワードで書籍のタイトルと著者を横断して部分一致検索する。 | ・URL: `/books/search` (GET)<br>・認証: 不要<br>・クエリパラメータ: `query` | ・キーワードに一致する書籍のみ表示される<br>・検索キーワードがフォームに維持される<br>・ページネーションは維持される | ・`BookController@search`<br>・`where("title", "like", ...)->orWhere("author", "like", ...)`<br>・`paginate(10)->withQueryString()` | 
| | | BF08 | 登録フォーム表示 | 新規書籍を登録するためのフォームを表示する。ジャンル一覧をDBから取得してチェックボックスで表示する。 | ・URL: `/books/create` (GET)<br>・認証: 必要 | ・書籍登録フォームが表示される<br>・全ジャンルがチェックボックスで表示される | ・`BookController@create`<br>・`Genre::all()`で全ジャンルを取得<br>・ビューに`$genres`を渡す | 
| | | BF09 | 登録処理 | 入力された情報で新規書籍を登録する。登録者情報も記録する。 | ・URL: `/books` (POST)<br>・認証: 必要<br>・パラメータ: `title`, `author`, `isbn`, `published_date`, `description`, `image_url`, `genres[]` | ・`books`テーブルにデータが保存される<br>・`book_genre`テーブルにジャンル情報が保存される<br>・登録した書籍の詳細ページにリダイレクトされる | ・`BookController@store`<br>・`StoreBookRequest`でバリデーション<br>・`$request->user()->books()->create()`で保存<br>・`$book->genres()->attach()`で紐付け | 
| | | BF10 | 詳細表示 | 指定された書籍の詳細情報を表示する。関連するレビューやジャンルも表示する。 | ・URL: `/books/{book}` (GET)<br>・認証: 不要 | ・書籍の詳細情報が表示される<br>・投稿されたレビューの一覧が表示される<br>・紐づくジャンルが表示される | ・`BookController@show`<br>・ルートモデルバインディングを利用<br>・`load(["reviews.user", "genres"])`でEager Loading | 
| | | BF11 | 編集フォーム表示 | 既存の書籍情報を編集するためのフォームを表示する。 | ・URL: `/books/{book}/edit` (GET)<br>・認証: 必要<br>・認可: 登録者本人のみ | ・書籍情報が入力された編集フォームが表示される<br>・紐づくジャンルがチェックされた状態で表示される | ・`BookController@edit`<br>・`authorize("update", $book)`で認可<br>・ビューに`$book`と`$genres`を渡す | 
| | | BF12 | 更新処理 | 入力された情報で書籍情報を更新する。 | ・URL: `/books/{book}` (PUT)<br>・認証: 必要<br>・認可: 登録者本人のみ<br>・パラメータ: (登録処理と同様) | ・`books`テーブルのデータが更新される<br>・`book_genre`テーブルの紐付けが更新される<br>・更新した書籍の詳細ページにリダイレクトされる | ・`BookController@update`<br>・`UpdateBookRequest`でバリデーション<br>・`authorize("update", $book)`で認可<br>・`$book->update()`で更新<br>・`$book->genres()->sync()`で紐付け更新 | 
| | | BF13 | 削除処理 | 既存の書籍を削除する。関連するレビューなどもカスケード削除される。 | ・URL: `/books/{book}` (DELETE)<br>・認証: 必要<br>・認可: 登録者本人のみ | ・対象の書籍が`books`テーブルから削除される<br>・書籍一覧ページにリダイレクトされる<br>・削除完了メッセージが表示される | ・`BookController@destroy`<br>・`authorize("delete", $book)`で認可<br>・`$book->delete()`で削除 | 
| 3 | **レビュー管理** | BF14 | 投稿処理 | 書籍詳細ページからレビュー（評価とコメント）を投稿する。 | ・URL: `/books/{book}/reviews` (POST)<br>・認証: 必要<br>・パラメータ: `rating`, `comment` | ・`reviews`テーブルにデータが保存される<br>・投稿した書籍の詳細ページにリダイレクトされる<br>・投稿完了メッセージが表示される | ・`ReviewController@store`<br>・`StoreReviewRequest`でバリデーション<br>・`$book->reviews()->create()`で保存 | 
| | | BF15 | 編集フォーム表示 | 自身のレビューを編集するためのフォームを表示する。 | ・URL: `/reviews/{review}/edit` (GET)<br>・認証: 必要<br>・認可: 投稿者本人のみ | ・レビュー内容が入力された編集フォームが表示される | ・`ReviewController@edit`<br>・`authorize("update", $review)`で認可<br>・ビューに`$review`を渡す | 
| | | BF16 | 更新処理 | 入力された情報でレビューを更新する。 | ・URL: `/reviews/{review}` (PUT)<br>・認証: 必要<br>・認可: 投稿者本人のみ<br>・パラメータ: `rating`, `comment` | ・`reviews`テーブルのデータが更新される<br>・レビュー対象の書籍詳細ページにリダイレクトされる<br>・更新完了メッセージが表示される | ・`ReviewController@update`<br>・`UpdateReviewRequest`でバリデーション<br>・`authorize("update", $review)`で認可<br>・`$review->update()`で更新 | 
| | | BF17 | 削除処理 | 自身のレビューを削除する。 | ・URL: `/reviews/{review}` (DELETE)<br>・認証: 必要<br>・認可: 投稿者本人のみ | ・対象のレビューが`reviews`テーブルから削除される<br>・レビュー対象の書籍詳細ページにリダイレクトされる<br>・削除完了メッセージが表示される | ・`ReviewController@destroy`<br>・`authorize("delete", $review)`で認可<br>・`$review->delete()`で削除 | 
| 4 | **ジャンル管理** | BF18 | 一覧・登録フォーム | ジャンルの一覧と新規登録フォームを同じ画面に表示する。 | ・URL: `/genres` (GET)<br>・認証: 必要 | ・登録済みジャンルの一覧が表示される<br>・新規登録フォームが表示される | ・`GenreController@index`<br>・`Genre::withCount(\'books\')->get()`で書籍数を取得<br>・ビューに`$genres`を渡す | 
| | | BF19 | 登録処理 | 新しいジャンルを登録する。 | ・URL: `/genres` (POST)<br>・認証: 必要<br>・パラメータ: `name` | ・`genres`テーブルにデータが保存される<br>・ジャンル管理ページにリダイレクトされる<br>・登録完了メッセージが表示される | ・`GenreController@store`<br>・`StoreGenreRequest`でバリデーション<br>・`Genre::create()`で保存 | 
| | | BF20 | 更新処理 | 既存のジャンル名を更新する。 | ・URL: `/genres/{genre}` (PUT)<br>・認証: 必要<br>・パラメータ: `name` | ・`genres`テーブルのデータが更新される<br>・ジャンル管理ページにリダイレクトされる<br>・更新完了メッセージが表示される | ・`GenreController@update`<br>・`UpdateGenreRequest`でバリデーション<br>・`$genre->update()`で更新 | 
| | | BF21 | 削除処理 | 既存のジャンルを削除する。ただし、そのジャンルに紐づく書籍が存在する場合は削除しない。 | ・URL: `/genres/{genre}` (DELETE)<br>・認証: 必要 | ・対象のジャンルが削除される<br>・ジャンル管理ページにリダイレクトされる<br>・削除完了メッセージが表示される<br>・(エラー時) エラーメッセージと共にリダイレクト | ・`GenreController@destroy`<br>・`$genre->books()->count()`で紐付けチェック<br>・`$genre->delete()`で削除 | 
| 5 | **お気に入り** | BF22 | 一覧表示 | 自身がお気に入りに登録した書籍を一覧表示する。 | ・URL: `/favorites` (GET)<br>・認証: 必要 | ・お気に入り登録した書籍が一覧表示される<br>・10件ごとにページネーション | ・`FavoriteController@index`<br>・`Auth::user()->favoriteBooks()->paginate(10)`<br>・ビューに`$books`を渡す | 
| | | BF23 | 登録・解除 | 書籍をお気に入り登録、または解除する（トグル動作）。 | ・URL: `/books/{book}/favorites` (POST)<br>・認証: 必要 | ・`favorites`テーブルのデータが作成または削除される<br>・直前のページにリダイレクトされる | ・`FavoriteController@toggle`<br>・`Auth::user()->favoriteBooks()->toggle($book->id)` | 
| 6 | **いいね** | BF24 | 登録・解除 | レビューに「いいね」を付ける、または外す（トグル動作）。 | ・URL: `/reviews/{review}/like` (POST)<br>・認証: 必要 | ・`review_likes`テーブルのデータが作成または削除される<br>・直前のページにリダイレクトされる | ・`ReviewLikeController@toggle`<br>・`Auth::user()->likedReviews()->toggle($review->id)` | 
| 7 | **評価ランキング** | AF01 | ランキング表示 | レビューの平均評価点が高い書籍トップ10をランキング形式で表示する。 | ・URL: `/ranking` (GET)<br>・認証: 不要 | ・平均評価が高い順に書籍が10件表示される<br>・書籍ごとに平均評価点が表示される | ・`RankingController@index`<br>・`Book::select(..., DB::raw(\'AVG(reviews.rating) as average_rating\'))`<br>・`join(\'reviews\', ...)`<br>・`groupBy(\'books.id\')`<br>・`orderByDesc(\'average_rating\')`<br>・`take(10)->get()` | 
| 8 | **ジャンル別一覧** | AF02 | ジャンル別表示 | 指定されたジャンルに属する書籍を一覧表示する。 | ・URL: `/genres/{genre}` (GET)<br>・認証: 不要 | ・指定されたジャンルの書籍一覧が表示される<br>・10件ごとにページネーション | ・`GenreController@show`<br>・`$genre->books()->with(\'genres\')->paginate(10)`<br>・ビューに`$genre`と`$books`を渡す | 

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
        foreignId book_id FK
        foreignId genre_id FK
        PK(book_id, genre_id)
    }

    favorites {
        foreignId user_id FK
        foreignId book_id FK
        PK(user_id, book_id)
    }

    review_likes {
        foreignId user_id FK
        foreignId review_id FK
        PK(user_id, review_id)
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
  - `foreignId(\'user_id\')->constrained()->cascadeOnDelete()`: 外部キー (users)
  - `string(\'title\')`: タイトル
  - `string(\'author\')`: 著者
  - `string(\'isbn\', 13)->unique()`: ISBN
  - `date(\'published_date\')`: 出版日
  - `text(\'description\')->nullable()`: 書籍説明
  - `string(\'image_url\')->nullable()`: 書影URL
  - `timestamps()`: 作成日時、更新日時

#### `genres` テーブル
- **マイグレーション**: `create_genres_table`
- **役割**: 書籍のカテゴリを管理するマスターテーブル。
- **スキーマ**:
  - `id()`: 主キー
  - `string(\'name\')->unique()`: ジャンル名
  - `timestamps()`: 作成日時、更新日時

#### `reviews` テーブル
- **マイグレーション**: `create_reviews_table`
- **役割**: 書籍に対するレビュー情報を格納する。
- **スキーマ**:
  - `id()`: 主キー
  - `foreignId(\'user_id\')->constrained()->cascadeOnDelete()`: 外部キー (users)
  - `foreignId(\'book_id\')->constrained()->cascadeOnDelete()`: 外部キー (books)
  - `unsignedTinyInteger(\'rating\')`: 評価 (1-5)
  - `text(\'comment\')`: コメント
  - `timestamps()`: 作成日時、更新日時

#### `book_genre` テーブル
- **マイグレーション**: `create_book_genre_table`
- **役割**: `books`と`genres`の多対多リレーションを定義する中間テーブル。
- **スキーマ**:
  - `foreignId(\'book_id\')->constrained()->cascadeOnDelete()`: 外部キー (books)
  - `foreignId(\'genre_id\')->constrained()->cascadeOnDelete()`: 外部キー (genres)
  - `primary([\'book_id\', \'genre_id\'])`: 複合主キー

#### `favorites` テーブル
- **マイグレーション**: `create_favorites_table`
- **役割**: `users`と`books`の多対多リレーション（お気に入り）を定義する中間テーブル。
- **スキーマ**:
  - `foreignId(\'user_id\')->constrained()->cascadeOnDelete()`: 外部キー (users)
  - `foreignId(\'book_id\')->constrained()->cascadeOnDelete()`: 外部キー (books)
  - `primary([\'user_id\', \'book_id\'])`: 複合主キー

#### `review_likes` テーブル
- **マイグレーション**: `create_review_likes_table`
- **役割**: `users`と`reviews`の多対多リレーション（いいね）を定義する中間テーブル。
- **スキーマ**:
  - `foreignId(\'user_id\')->constrained()->cascadeOnDelete()`: 外部キー (users)
  - `foreignId(\'review_id\')->constrained()->cascadeOnDelete()`: 外部キー (reviews)
  - `primary([\'user_id\', \'review_id\'])`: 複合主キー

### 4.3 モデル定義
#### `User` モデル
- `books()`: `hasMany(Book::class)`
- `reviews()`: `hasMany(Review::class)`
- `favoriteBooks()`: `belongsToMany(Book::class, \'favorites\')`
- `likedReviews()`: `belongsToMany(Review::class, \'review_likes\')`

#### `Book` モデル
- `protected $fillable = [\'user_id\', \'title\', \'author\', \'isbn\', \'published_date\', \'description\', \'image_url\'];`
- `user()`: `belongsTo(User::class)`
- `reviews()`: `hasMany(Review::class)`
- `genres()`: `belongsToMany(Genre::class, \'book_genre\')`
- `favoritedByUsers()`: `belongsToMany(User::class, \'favorites\')`

#### `Genre` モデル
- `protected $fillable = [\'name\'];`
- `books()`: `belongsToMany(Book::class, \'book_genre\')`

#### `Review` モデル
- `protected $fillable = [\'user_id\', \'book_id\', \'rating\', \'comment\'];`
- `user()`: `belongsTo(User::class)`
- `book()`: `belongsTo(Book::class)`
- `likedByUsers()`: `belongsToMany(User::class, \'review_likes\')`

---
## 5. 初期データ（シーディング）
#### `UserSeeder`
- テストユーザーを5件作成する。

#### `GenreSeeder`
- 以下の10件のジャンルを登録する。
  - '小説', 'ビジネス', '技術書', '自己啓発', 'エッセイ', '歴史', '科学', '芸術', '料理', '旅行'

#### `BookSeeder`
- ダミーデータを11件作成し、各書籍に1〜2個のジャンルを紐付ける。

#### `ReviewSeeder`
- 各書籍に対して、複数のユーザーからダミーレビューを登録する。

#### `FavoriteSeeder`
- 各ユーザーに複数のお気に入り書籍を紐付ける。

#### `ReviewLikeSeeder`
- 各レビューにランダムで0〜3件の「いいね」を紐付ける。

#### `DatabaseSeeder`
- `run` メソッド内で `$this->call()` を使用し、`UserSeeder`, `GenreSeeder`, `BookSeeder`, `ReviewSeeder`, `FavoriteSeeder`, `ReviewLikeSeeder` の順で呼び出す。

---
## 6. ルーティング (`routes/web.php`)
| Method | Path | Controller@Action | Route Name | Middleware |
|---|---|---|---|---|
| GET | `/` | `BookController@index` | `home` | - |
| GET | `/books` | `BookController@index` | `books.index` | - |
| GET | `/books/search` | `BookController@search` | `books.search` | - |
| GET | `/books/create` | `BookController@create` | `books.create` | `auth` |
| POST | `/books` | `BookController@store` | `books.store` | `auth` |
| GET | `/books/{book}` | `BookController@show` | `books.show` | - |
| GET | `/books/{book}/edit` | `BookController@edit` | `books.edit` | `auth` |
| PUT | `/books/{book}` | `BookController@update` | `books.update` | `auth` |
| DELETE | `/books/{book}` | `BookController@destroy` | `books.destroy` | `auth` |
| GET | `/genres` | `GenreController@index` | `genres.index` | `auth` |
| POST | `/genres` | `GenreController@store` | `genres.store` | `auth` |
| GET | `/genres/{genre}/edit` | `GenreController@edit` | `genres.edit` | `auth` |
| PUT | `/genres/{genre}` | `GenreController@update` | `genres.update` | `auth` |
| DELETE | `/genres/{genre}` | `GenreController@destroy` | `genres.destroy` | `auth` |
| GET | `/genres/{genre}` | `GenreController@show` | `genres.show` | - |
| POST | `/books/{book}/reviews` | `ReviewController@store` | `reviews.store` | `auth` |
| GET | `/reviews/{review}/edit` | `ReviewController@edit` | `reviews.edit` | `auth` |
| PUT | `/reviews/{review}` | `ReviewController@update` | `reviews.update` | `auth` |
| DELETE | `/reviews/{review}` | `ReviewController@destroy` | `reviews.destroy` | `auth` |
| GET | `/favorites` | `FavoriteController@index` | `favorites.index` | `auth` |
| POST | `/books/{book}/favorites` | `FavoriteController@toggle` | `favorites.toggle` | `auth` |
| POST | `/reviews/{review}/like` | `ReviewLikeController@toggle` | `reviews.like` | `auth` |
| GET | `/ranking` | `RankingController@index` | `ranking.index` | - |

*その他、認証関連のルートが `routes/auth.php` に定義される (Laravel Breeze)*

---
## 7. バリデーション (FormRequest)
#### `StoreBookRequest`
- `title`, `author`: `required|string|max:255`
- `isbn`: `required|string|size:13|unique:books`
- `published_date`: `required|date`
- `description`, `image_url`: `nullable|string`
- `genres`: `required|array`
- `genres.*`: `exists:genres,id`

#### `UpdateBookRequest`
- `isbn` のユニークチェックを `Rule::unique(\'books\')->ignore($this->book)` に変更する。

#### `StoreGenreRequest`
- `name`: `required|string|max:255|unique:genres`

#### `UpdateGenreRequest`
- `name` のユニークチェックを `Rule::unique(\'genres\')->ignore($this->genre)` に変更する。

#### `StoreReviewRequest` / `UpdateReviewRequest`
- `rating`: `required|integer|min:1|max:5`
- `comment`: `nullable|string|max:1000`

---
## 8. 認証・認可
- **認証方式**: セッションベース認証（Laravel Breezeを利用）。
- **アクセス制御**: ルーティング定義にて、認証が必要なルートに`auth`ミドルウェアを適用する。
- **認可方式**: ポリシー（Policy）を利用する。
  - `BookPolicy`: 書籍の`update`, `delete`は、書籍を登録したユーザー本人にのみ許可する。
  - `ReviewPolicy`: レビューの`update`, `delete`は、レビューを投稿したユーザー本人にのみ許可する。
- **権限NG時の挙動**: `auth`ミドルウェアにより、未認証ユーザーはログインページへリダイレクトされる。ポリシーによる認可失敗時は403エラーページが表示される。

---
## 9. 受け入れ条件
- **主要なユーザーフロー**：
  1. ユーザーが会員登録・ログインできる。
  2. ログイン後、書籍を登録・編集・削除できる。また、他人の書籍は編集・削除できない。
  3. 書籍詳細ページでレビューを投稿・編集・削除できる。また、他人のレビューは編集・削除できない。
  4. 書籍一覧ページで、キーワード検索が機能する。
  5. 管理者としてジャンルをCRUDできる。
  6. 応用機能（ランキング、ジャンル別一覧）が仕様通りに動作する。
- **成功条件**：
  - 上記フローがエラーなく完了し、DBの状態が期待通りに更新されていること。
  - 各種バリデーションが正しく機能し、不正なデータは登録されないこと。
- **エラー条件**：
  - 必須項目未入力でフォームを送信した場合、バリデーションエラーが返されること。
  - 未ログイン状態で認証必須ページにアクセスしようとした場合、ログインページにリダイレクトされること。
  - 他人のリソースを不正に操作しようとした場合、403エラーが表示されること。
