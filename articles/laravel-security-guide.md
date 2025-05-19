---
title: "Laravel セキュリティ対策 完全ガイド"
emoji: "🛡️"
type: "tech"
topics: ["Laravel", "セキュリティ", "PHP", "XSS", "SQLインジェクション"]
published: true
---

# はじめに
本記事は、 Laravel を使った開発におけるセキュリティ対策について、自分自身の備忘録としてまとめたものです。
特に、実務で頻出する脅威（ XSS ・ CSRF ・ SQLインジェクション など）とその具体的な対策を整理し、後から見返しやすいように構成しています。
同じように Laravel で開発を行う方にも参考になる内容になっていれば幸いです。

# 脅威一覧
下記、IPA の「[ウェブアプリケーションのセキュリティ実装](https://www.ipa.go.jp/security/vuln/websecurity/about.html)」から抜粋
* SQLインジェクション
* OSコマンド・インジェクション
* パス名パラメータの未チェック／ディレクトリ・トラバーサル
* セッション管理の不備
* クロスサイト・スクリプティング
* CSRF（クロスサイト・リクエスト・フォージェリ）
* HTTPヘッダ・インジェクション
* メールヘッダ・インジェクション
* クリックジャッキング
* バッファオーバーフロー
* アクセス制御や認可制御の欠落

# 脅威の概要と対策方法
## SQLインジェクション
### 概要
利用者からの入力情報を基にSQL文（データベースへの命令文）を組み立てる際に、攻撃によってデータベースの不正利用をまねく可能性があります。前述の問題を悪用した攻撃を、「SQLインジェクション攻撃」と呼びます。
### 対策
LaravelのEloquent ORMは、SQLインジェクション攻撃を防ぐための機能を標準で提供しています。下記がSQLインジェクション対策として搭載されている機能です。
* プリペアドステートメントの自動使用
* パラメータのバインド
* クエリビルダによる安全なSQL生成

Laravelでは、基本的にSQLインジェクションへの対策はされています。しかし、素のSQLを叩く場合は注意が必要です。

## OSコマンド・インジェクション
### 概要
外部からの攻撃により、ウェブサーバのOSコマンドを不正に実行されてしまう問題を持つものがあります。前述の問題を悪用した攻撃手法を、「OSコマンド・インジェクション攻撃」と呼びます。
### 対策
以下の対策があります。
* 入力値のバリデーション
    * 特殊文字を入力させない
* Laravelのコマンド実行方法を変更
    * シェル経由ではなく、`Artisan::call`や`Symfony\Component\Process\Process`で実行する
    * `shell_exec()`や`exec()`を使用しない

## パス名パラメータの未チェック／ディレクトリ・トラバーサル
### 概要
ウェブアプリケーションが外部からのパラメータでファイル名を指定する場合、実装に問題があると攻撃者に任意のファイルを指定され、意図しない処理が実行されるリスクがあります。この問題を悪用した攻撃手法の一つに、「ディレクトリ・トラバーサル攻撃」があります。
### 対策
以下の対策があります。
* ユーザー入力を直接パスに使わない
* basename関数を使用する
    * ファイル名のみ抜き出すため、階層移動などの対策ができる
* 正規表現で入力を検証
    * ディレクトリ操作に関わる文字列がないかチェックする

## セッション管理の不備
### 概要
セッションIDの発行や管理に不備がある場合、悪意のある人にログイン中の利用者のセッションIDを不正に取得され、その利用者になりすましてアクセスされてしまう可能性があります。この問題を悪用した攻撃手法を、「セッション・ハイジャック」と呼びます。
### 対策
* セッションIDの再生成
    * 例：ログイン後に再生成する
```php
Session::regenerate()
```
* HTTPS通信の強制化
```php:config/session.php
// 環境変数でHTTPS通信を強制化する
'secure' => env('SESSION_SECURE_COOKIE', false)
```
* セッションタイムアウト
    * 例：時間が経ったらログアウトさせる
```php:config/session.php
// 環境変数でセッションの永続期間を指定
'lifetime' => env('SESSION_LIFETIME', 120)
```

## クロスサイト・スクリプティング（XSS）
### 概要
利用者からの入力内容やHTTPヘッダの情報を処理し、ウェブページとして出力するものがあります。ここでウェブページへの出力処理に問題がある場合、そのウェブページにスクリプト等を埋め込まれてしまいます。
この問題を悪用した攻撃手法を、「クロスサイト・スクリプティング攻撃」と呼びます。
### 対策
LaravelのBladeでは、デフォルトでXSS対策が行われています。`{{}}`を使用することで、[`htmlspecialchars`関数](https://qiita.com/pito555/items/a2f649a050a0b7a46cf1)を通され自動的に送信されます。その他の対策として、下記があります。
* Bladeのエスケープ付き出力構文を使用する
```php
{{ $message }}
```
* 入力値のバリデーションを行う
* `e`関数を使用する

## CSRF（クロスサイト・リクエスト・フォージェリ）
### 概要
ログイン機能を持つウェブサイトで、リクエストの正当性を確認する仕組みがないと、外部サイト経由で悪意あるリクエストを受け入れ、利用者が意図しない操作を実行させられる可能性があります。
このような問題を「CSRF（Cross-Site Request Forgeries／クロスサイト・リクエスト・フォージェリ）の脆弱性」と呼び、これを悪用した攻撃を、「CSRF攻撃」と呼びます。
### 対策
Laravelでは、CSRF対策としてトークン生成が採用されています。サーバ側でトークン検証を行うために、「_token」というname属性を持つ非表示のinputタグをformタグの中に含めて送信する必要があります。
「@csrfディレクティブ」を用いると、非表示のトークン付きinputタグを丸ごと生成してくれます。
```html:example.blade.php
<form method="POST" action="{{route('/exemple')}}">
    @csrf
    <input type="text" name="txt"/>
    <input type="submit" value="送信"/>
</form>
```

## HTTPヘッダ・インジェクション
### 概要
外部パラメータをもとにHTTPレスポンスヘッダを動的生成するウェブアプリで、処理に問題があると、攻撃者に任意のヘッダやボディを挿入され、不正なレスポンスを作られる可能性があります。
この問題を悪用した攻撃手法は「HTTPヘッダ・インジェクション攻撃」と呼びます。特に、複数のレスポンスを作り出す攻撃は、「HTTPレスポンス分割（HTTP Response Splitting）攻撃」と呼びます。
### 対策
Laravelでは、標準のresponseメソッドを使う場合は、自動的に`\r, \n`を削除するなど対策が行われている。

* responseメソッドを使用する
```php
response()->redirectTo();
```
* Locationヘッダなどに外部パラメータを使用する場合はバリデーションを行う
* URLの検証（正当性チェック）を行う
    * 基本的に相対パスが安全

## メールヘッダ・インジェクション
### 概要
ウェブアプリケーションには、ユーザーが入力した情報を特定のメールアドレスに送信する機能があります。このメールアドレスは通常管理者のみが変更可能ですが、実装次第では外部の利用者が送信先アドレスを自由に指定できてしまう場合があります。
このような問題を悪用した攻撃を、「メールヘッダ・インジェクション攻撃」と呼びます。
### 対策
Laravelの標準メール送信機能（MailファサードやMailableクラス）は、デフォルトでメールヘッダを適切に処理しており、メールヘッダ・インジェクションを防ぐために安全な方法でヘッダを設定します。

* Mail::to()やMailableを利用する
```php
use Illuminate\Support\Facades\Mail;
use App\Mail\ContactFormMail;

Mail::to($request->input('email'))->send(new ContactFormMail($data));
```
* 入力のエスケープ
```php
// メールアドレスのサニタイズ
$email = filter_var($request->input('email'), FILTER_SANITIZE_EMAIL);

// 件名のサニタイズ
$subject = str_replace(["\r", "\n", "%0A", "%0D"], "", $request->input('subject'));
```
* 入力値のバリデーションを行う

## クリックジャッキング
### 概要
ウェブサイトのログイン機能を利用する際、マウス操作のみで利用可能な機能がある場合、外部サイトを通じて誤操作を引き起こし、意図しない機能が実行されるリスクがあります。
このような問題を「クリックジャッキングの脆弱性」と呼び、問題を悪用した攻撃を、「クリックジャッキング攻撃」と呼びます。
### 対策
iflameなどの埋め込みを許可しないことで対策できます。
```php:AddXFrameOptions.php
<?php

namespace App\Http\Middleware;

use Closure;

class AddXFrameOptions
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);
        $response->headers->set('X-Frame-Options', 'deny'); //他のWebページでそのページを<iframe>で表示することを禁止する
        return $response;
    }
}
```
> 参考：[Laravelセキュリティ：クリックジャッキング対策](https://qiita.com/teru0x1/items/c622fa814809061bfcad)

## バッファオーバーフロー
### 概要
プログラムはメモリ上に領域を確保しますが、入力データを適切に扱わないとメモリ領域外が上書きされ、意図しないコードが実行されることがあります。
この問題を悪用した攻撃を「バッファオーバーフロー攻撃」と呼びます。
### 対策
* 入力値のバリデーションを行う
    * 意図しない値が渡されることを防ぐ
```php
$validated = $request->validate([
    'name' => 'required|string|max:255',
]);
```
* サーバー設定でパラメータを制限する
```ini
# PHP が処理できる最大 POST データのサイズを制限
post_max_size = 8M
# PHP が許可する最大のアップロードファイルサイズを制限
upload_max_filesize = 2M
# 1回のリクエストで処理する最大の入力変数を設定
max_input_vars = 1000
# 入力データのネストの最大深さを制限
max_input_nesting_level = 64
# PHP スクリプトが使用できる最大メモリ量を制限
memory_limit = 128M
# PHP スクリプトの実行最大時間を設定
max_execution_time = 30
# セキュリティ上危険な PHP 関数を無効
disable_functions = exec, shell_exec, system, passthru
```
* 依存関係を最新に保つ
```sh
composer update
```

## アクセス制御の欠落
### 概要（根本解決）
アクセス制御機能による防御措置が必要とされるウェブサイトには、パスワード等の秘密情報の入力を必要とする認証機能を設ける。
### 対策
1. Laravel認証システムの利用
Laravelは、認証機能を簡単に実装できる「Laravel Breeze」や「Laravel Jetstream」などのパッケージを提供しています。これらを利用すると、ユーザーの登録、ログイン、パスワードリセットなどの機能を容易に設定できます。

```bash:使用例
composer require laravel/breeze --dev
php artisan breeze:install
npm install && npm run dev
php artisan migrate
```

2. パスワードの暗号化
Laravelはデフォルトでパスワードを暗号化するため、bcrypt（または argon2）アルゴリズムを使用して安全に保存します。ユーザーのパスワードは平文で保存されず、常にハッシュ化されます。
```php
use Illuminate\Support\Facades\Hash;

$password = Hash::make('your-password');
```

ログイン時に、入力されたパスワードと保存されたハッシュを比較して認証を行う。

```php
if (Hash::check('your-password', $storedPassword)) {
    // パスワードが一致
}
```

## 認可制御の欠落
### 概要（根本解決）
認証機能に加えて認可制御の処理を実装し、ログイン中の利用者が他人になりすましてアクセスできないようにする。
### 対策
#### ポリシー（Policy）を使った認可制御
Laravelでは、ポリシー（Policy）を使って、特定のユーザーがリソースにアクセスできるかどうかを判断します。ポリシーを使用することで、アプリケーション内の認可ロジックを整理して管理することができます。
1. ポリシーの作成
```bash
php artisan make:policy PostPolicy --model=Post
```
2. ポリシーの定義
```php
namespace App\Policies;

use App\Models\User;
use App\Models\Post;

class PostPolicy
{
    /**
     * 投稿を編集できるかどうかを判断する
     *
     * @param  \App\Models\User  $user
     * @param  \App\Models\Post  $post
     * @return bool
     */
    public function update(User $user, Post $post)
    {
        // ユーザーが投稿のオーナーである場合のみ編集可能
        return $user->id === $post->user_id;
    }
}
```
3. ポリシーの登録
```php
namespace App\Providers;

use App\Models\Post;
use App\Policies\PostPolicy;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * ポリシーのマッピング
     *
     * @var array
     */
    protected $policies = [
        Post::class => PostPolicy::class,
    ];

    /**
     * アプリケーションのサービス提供をブートする
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();
    }
}
```
4. ポリシーを使用する
```php
namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    /**
     * 投稿を編集する
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \App\Models\Post  $post
     * @return \Illuminate\Http\Response
     */
    public function update(Request $request, Post $post)
    {
        // ポリシーを使って認可を確認
        $this->authorize('update', $post);

        // 認可された場合、投稿の更新処理
        $post->update($request->all());

        return redirect()->route('posts.index');
    }
}
```

#### ゲート（Gate）を使った認可制御
ポリシーが便利ですが、リソースが複数のモデルに関連しない場合や、シンプルな認可制御が必要な場合は、ゲート（Gate）を使うとより簡潔に認可を実装できます。
1. ゲートの定義
```php
namespace App\Providers;

use App\Models\User;
use App\Models\Post;
use Illuminate\Support\Facades\Gate;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * アプリケーションのサービス提供をブートする
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        // ゲートを定義
        Gate::define('update-post', function (User $user, Post $post) {
            return $user->id === $post->user_id;
        });
    }
}
```
2. ゲートの使用
```php
public function update(Request $request, Post $post)
{
    // ゲートを使って認可を確認
    $this->authorize('update-post', $post);

    // 認可された場合、投稿の更新処理
    $post->update($request->all());

    return redirect()->route('posts.index');
}
```

#### ユーザーの認可確認
認可に関しては、特定のリソースに対する権限を持っているかどうかを事前に確認する必要があります。authorize メソッドを使うことで、簡単に認可制御ができます。
1. なりすまし防止対策
```php
Gate::define('edit-post', function ($user, $post) {
    return $user->id === $post->user_id || $user->is_admin;
});
```

# まとめ
今回は、IPAの安全なウェブサイトの作り方のセキュリティ実装をLaravelに落とし込んでみました。セキュリティ対策についてフレームワークレベルで掘り下げることはあまりしなかったこともあり、フレームワーク特有の対策方法を再確認することができました。本記事をぜひ参考にしていただけたら嬉しいです。
