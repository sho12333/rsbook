# Blazor WebAssemblyでAPI通信を行う


# HttpClient を用いて API 通信を行う

## はじめに

筆者は現在 Blazor を勉強しており、登竜門である API 通信を行ってみます。

動作環境は以下の通りです。

| ツール   | 名称               |
| -------- | ------------------ |
| PC       | Windows11          |
| エディタ | Visual Studio 2022 |

## 1.Controller クラスの作成

本記事は書籍『ドメイン駆動設計入門 ボトムアップでわかる! ドメイン駆動設計の基本』に基づき、以下のフォルダ構成で開発を行います。

- Controller
  - エンドポイントを設定
- Service
  - 業務ロジックを定義
- Repository
  - DB アクセスロジックを定義
- Model
  - バインド対象のモデルを定義

<br/>

では、実際にコードを記述していきましょう。

```C#
namespace Sample.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class UserController : ControllerBase
    {
        public IUserService userService;

        public UserController(IUserService userService)
        {
            this.userService = userService;
        }

        [HttpGet]
        public async Task<ActionResult<IEnumerable<User>>> Get()
        {
            var results = await userService.GetUsers();
            return Ok(result);
        }

        [HttpPost, Route("{id}")]
        public async Task<ActionResult<User>> Post(string id)
        {
            var result = await userService.GetUser(id);
            return Ok(result);
        }
    }
}
```

ここで、**IUserService**クラスはまだ定義していませんが、処理の流れを決めるために、先に記述しておきます。

### Controller クラス

Controller クラスは API のエンドポイントの役割を果たします。
ApiController 属性を Controller クラスに適用することで、以下の API 固有の動作を提供できます。

- 属性ルーティング要件
- 自動的な HTTP 400 応答
- バインディングソース属性
- マルチパート/フォーム データ要求の推論

#### 属性ルーティング要件

属性ルーティング要件とは、コードの`[Route("api/[controller]")]`の部分です。
この場合だと、`[controller]`の部分は`UserController`の`User`の部分がマッピングされます。

#### 自動的な HTTP400 応答

モデル検証エラーが発生すると HTTP 400 応答が自動的にトリガーされます。 その結果、アクション メソッド内でステータスコード 400 を入れる処理が不要になります。

#### バインディングソース属性

バインディング ソース属性では、アクション パラメーターの値が存在する場所が定義されます。 次のバインディング ソース属性が存在します。

| 属性             | バインド ソース                                     |
| ---------------- | --------------------------------------------------- |
| `[FromBody]`     | 要求本文                                            |
| `[FromForm]`     | 要求本文内のフォーム データ                         |
| `[FromHeader]`   | 要求ヘッダー                                        |
| `[FromQuery]`    | 要求のクエリ文字列パラメーター                      |
| `[FromRoute]`    | 現在の要求からのルート データ                       |
| `[FromServices]` | アクション パラメーターとして挿入される要求サービス |
| `[AsParameters]` | メソッドのパラメーター                              |

今回の場合は、`Post`メソッド内で引数`id`を渡していますが、属性が省略されています。
この場合は、`ASP.NET Core ランタイム`により複合オブジェクト モデル バインダーの使用が試行されます。 複合オブジェクト モデル バインダーでは、値プロバイダーから定義された順序でデータを取得します。

#### マルチパート/フォーム データ要求の推論

[ApiController] 属性を設定すると、IFormFile および IFormFileCollection 型のアクション パラメーターに対して推論規則が適用されます。 multipart/form-data 要求コンテンツ タイプは、これらの種類に対して推論されます。

## 2.Service クラスの作成

### Service クラス

Service クラスはユースケースを実現するために使用されます。今回の場合ではユーザーのデータを操作するので、ユーザー取得、更新、削除といったユースケースを作成します。

同じように、コードを記述していきましょう。

```C#
namespace Sample.Services
{
    public class UserService : IUserService
    {
        public IUserRepository userRepository;

        public UserService(IUserRepository userRepository)
        {
            this.userRepository = userRepository;
        }

        public async Task<ActionResult<IEnumerable<User>>> Get()
        {
            return await userRepository.GetUsers();
        }

        public async Task<ActionResult<User>> Post(string id)
        {
            return await userService.GetUser(id);
        }
    }
}
```

先ほど Controller クラスで定義した`UserService`クラスとなります。
同じように、`IUserRepository`クラスはまだ定義していませんが、後に実装します。

本来であれば、ビジネスロジックや複雑なユースケースを実現するのですが、今回は簡単な CRUD 処理のため、簡潔な処理のみとなっています。

## 3.Repository クラスの作成

さいごに Repository クラスの作成です。
Repository クラスではドメインオブジェクトの永続化や再構築を行うクラスです。

簡単に説明すると、DB の処理をラップするクラスです。

同じように、コードを記述していきましょう。

```C#
namespace Sample.Repositories
{
    public class UserRepository : IUserRepository
    {
        public async Task<ActionResult<IEnumerable<User>>> Get()
        {
            using(var connection = new SqlConnection(ConnectionStrings))
            using(var command = connection.createCommand())
            {
                connection.Open();
                var sql = "SELECT * FROM M_USER";

                var results = connection.Query<User>(sql);
                return results;
            }
        }
    }
}
```

このように、DB から取得したデータを Service クラスに渡す役割を果たします。

以上の 3 クラスを利用することで、API 通信を行うための前段階が出来ました。

## 4. razor ファイルの実装

ここで、Blazor の Client 側である razor ファイルの記述を行っていきます。

Blazor はその特徴から、Client 側と Server 側の両方が C#で記述できます。

それではコードを記述していきましょう。

```C#
@inject HttpClient Http

@if(users is not null)
{
    <ul>
        @foreach (var user in users)
        {
            <li>@user.UserId</li>
        }
    </ul>
}

@code {
    private IEnumerable<User> users = Enumerable.Empty<User>;

    protected override async Task OnInitializedAsync(){
        users = await Http.GetFromJsonAsync<IEnumerable<User></User>>("api/user");
    }
}

```

Blazor では HttpClient がデフォルトで用意されているので、それを`injectディレクティブ`を用いてサービスをコンポーネントに挿入します。

あとはエンドポイントに合わせてメソッドを呼び出すだけです。今回は`GET`メソッドを用いてユーザー情報を取得しているので、`GetFromJsonAsync`メソッドを用います。

これで API 通信を用いてデータを画面に表示することができました。

## さいごに

Blazor を用いて API 通信を行ってみました。
まだまだ細かい理解ができていない部分が多いので、分かり次第記述していきます。

## 参考文献

https://learn.microsoft.com/ja-jp/aspnet/core/blazor/call-web-api?view=aspnetcore-7.0&pivots=webassembly

https://learn.microsoft.com/ja-jp/aspnet/core/mvc/controllers/routing?view=aspnetcore-7.0#conventional-routing

