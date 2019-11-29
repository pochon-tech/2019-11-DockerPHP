Laravel + PHP-FPM + Nginx + Mysql
------------------------------

#### Project構成

```bash
project/
 ├ src/                 # Laravel Project
 ├ docker/              # Container
 │  ├  php/             # php Container 
 │  ├  nginx/           # Nginx Container
 │  ├  db/              # Mysql Container
 ├ logs/                # logs
 ├ docker-compose.yml
```

- php
  - Laravelをマウント（./src:/var/www）
- nginx
  - nginx:1.13.5-alpine
  - Proxy用
  - Laravelをマウント（./src:/var/www）
- db
  - mysql:5.7
  - 実データはここにマウント（TopVolumesのほうが良いかも。秘匿観点より）

setup (新規に始める場合)
-------------------------

- プロジェクトディレクトリの直下に、、、
  - `docker-compose.yml`を配置
  - `docker/`ディレクトリ一式を配置
  - 空ディレクトリ`src/`を作成 (cloneした場合はrm -rf src)
- 下記の実行

```bash
# コンテナ起動
$ docker-compose up -d

# コンテナの中に入る
$ docker-compose exec php bash

# 新規にLaravelプロジェクトを始める場合は下記を実行
$ laravel new
# open http://localhost/

# Version 確認
$ php artisan --version
Laravel Framework 6.6.0
```
#### .env修正

- composerを使ってプロジェクトを作成しているので`.env`が作られている
- DBの設定を変更する

```.env
DB_HOST=db
DB_PORT=3306
DB_DATABASE=database
DB_USERNAME=docker
DB_PASSWORD=docker
```

#### Migrationの実行

- プロジェクト作成時にUserのマイグレーションファイルが作られているので実行

```bash
$ php artisan migrate
Migration table created successfully.
Migrating: 2014_10_12_000000_create_users_table
Migrated:  2014_10_12_000000_create_users_table (0.1 seconds)
Migrating: 2014_10_12_100000_create_password_resets_table
Migrated:  2014_10_12_100000_create_password_resets_table (0.08 seconds)
Migrating: 2019_08_19_000000_create_failed_jobs_table
Migrated:  2019_08_19_000000_create_failed_jobs_table (0.06 seconds)
```

- ※ `docker/db`配下を削除したら実データが消える
- ※ DBを確認する
  ```bash
    $ docker-compose exec db bash
    $ mysql -udocker -pdocker
    $ use database;
    $ show tables;
  ```

簡易的な画面を作成する
-----------------------

#### モデルとマイグレーションを作成

```bash
$ php artisan make:model Article -m
```
- `php artisan make:model モデル名`でモデルファイルを作成
- `-m`オプションでマイグレーションファイルを同時に作成
- 生成されたモデルは`Eloquent`なモデル
  - EloquentはLaravelの標準ORM

#### マイグレーションファイルを編集

- `src\database\migrations\2019_11_29_051355_create_articles_table.php`にマイグレーションファイルが作られている
```php
public function up()
    {
        Schema::create('articles', function (Blueprint $table) {
            $table->increments('id');
            $table->string('title');
            $table->text('body');
            $table->timestamps();
        });
    }
```
- ここでは`string型のtitle`と`text型のbody`を追加した

#### マイグレーションファイルの実行

```bash
$ php artisan migrate
```
- 上記のコマンドを実行することでMysqlにテーブルが作成される

#### Controllerの作成

```bash
$ php artisan make:controller ArticlesController -r
```
- `php artisan make:controller コントローラー名`でコントローラーファイルを作成
- `-r`オプションでResourcefulなアクションを自動で作成

#### ルーティング設定

- `src\routes\web.php`にルーティング定義を追加する

```php
Route::get('/', function () {
    return view('welcome');
});
Route::resource('articles', 'ArticlesController');
```

- ※view('welcome')の正体 ( https://laraweb.net/knowledge/134/)
- ルーティングを確認したい時は下記コマンドで確認可能

```bash
$ php artisan route:list
```

#### seederを使って動作確認用ダミーデータの生成

```
$ php artisan make:seeder ArticlesTableSeeder
```
- 上記のコマンドで`Article`モデルにダミーデータ投入用ファイルが生成される
- `src\database\seeds\ArticlesTableSeeder.php`を編集してダミーデータを定義
  ```php
      public function run()
      {
          // articlesテーブルにデータをinsert
          DB::table('articles')->insert([
              [
              'title' => 'タイトル1',
              'body' => '内容1'
              ],
              [
              'title' => 'タイトル2',
              'body' => '内容2'
              ],
              [
              'title' => 'タイトル3',
              'body' => '内容3'
              ],
          ]);
      }
  ```
- `src\database\seeds\DatabaseSeeder.php`を編集して先ほど作成したSeedファイルを登録
  ```php
    $this->call(ArticlesTableSeeder::class);
  ```
- 下記コマンドでダミーデータをDBにInsertする
  ```bash
    $ php artisan db:seed
  ```
- Tinkerを使ってダミーデータが登録されたか確認
  ```bash
    php artisan tinker
    $a = App\Article::all();
    exit
  ```

#### Viewの作成

- Laravel標準テンプレートであるbladeを使う
- レイアウト用Viewを`src\resources\views\layouts\application.blade.php`として新たに作成する
  ```html
    <!DOCTYPE html>
    <html lang="ja">
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <meta http-equiv="X-UA-Compatible" content="ie=edge">
      <title>@yield('title')</title>
    </head>
    <body>
      @yield('content')
    </body>
    </html>
  ```
- Controllerがレンダリングする`src\resources\views\articles\index.blade.php`を作成する
  ```php:
    {{-- layoutsフォルダのapplication.blade.phpを継承 --}}
    @extends('layouts.application')

    {{-- @yield('title')にテンプレートごとの値を代入 --}}
    @section('title', '記事一覧')

    {{-- application.blade.phpの@yield('content')に以下のレイアウトを代入 --}}
    @section('content')
      @foreach ($articles as $article)
        <h4>{{$article->title}}</h4>
        <p>{{$article->body}}</p>
        <hr>
      @endforeach
    @endsection
  ```

#### Controllerを編集する
- 前述で自動生成した`src\app\Http\Controllers\ArticlesController.php`内に処理を実装する
```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Article; // 使用したいモデルがあれば随時追加

class ArticlesController extends Controller
{
    /**
     * Display a listing of the resource.
     *
     * @return \Illuminate\Http\Response
     */
    public function index()
    {
      // $articles変数にArticleモデルから全てのレコードを取得して、代入
      $articles = Article::all();
      return view('articles.index', ['articles' => $articles]);
    }
```
- http://localhost/articles


#### 所感

- API作るだけならNode(Express)が手軽で良い気がする
- LaravelはフルスタックすぎてAPIの作成だけのためには冗長感あり（ヘッドレスCMSには向かない？）
- 日本語の情報は多くてうれしい
- 規約重視な感じでチーム開発には向いてる
- 開発中のコード補完や型チェックがVSCodeでできないとは不便
  - ただし、開発ツールは優秀（Tinkerなど）
- Laravelの知見をいかせるMicroFWであるLumenも良さそう
  - PythonでゆうとこのFlask
  - silex,slimの競合らしい
- Swagger周りのエコシステムが少なめ？
  - いわゆる`TSOA`みたいなControllerからルーティングファイルやSwaggerを作るやつがない
  - `swagger-php`は後からSwaggerDocを作るのに便利
- ベンチマークだけ見るなら、Rust、Go、Java...いわゆる静的コンパイル言語が優秀？
  - https://www.techempower.com/benchmarks/

