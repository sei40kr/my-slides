<!-- headingDivider: 5 -->

## アジェンダ

- 前半: GraphQLの話
- 後半: GraphQLでDDDを再考する話

## GraphQLとは

- スキーマとクエリを定義するための言語
- 従来のWeb APIに代わってバックエンドとの通信を実現するためのFW
  - 厳密にはGraphQLはWeb API上で動作する。
- GraphQLエンドポイントと呼ばれる単一のWeb APIに対してGraphQLで書かれたクエリを送信する。

## GraphQLクラスの実装例

```kotlin
data class User(
  val id: Int,
  val name: String,
  val posts: List<Post>,
)

data class Post(
  val id: Int,
  val message: String,
)
```

## GraphQLクラスから生成されたGraphQLのスキーマの例

```graphql
type User {
  id: ID!
  name: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  message: String!
}
```

## GraphQLクエリResolverの実装例

```kotlin
class UserQueryResolver(private val findUserByIdUsecase: FindUserByIdUsecase) {
  fun user(id: ID): User? {
    val request = FindUserByIdUsecase.Request(id = id.toDomainId())

    return findUserByIdUsecase.execute(request)
  }
}
```

## GraphQLクエリの例

ここでは取得するユーザーのIDを指定していない。
`$id` はSQLのプレースホルダー `?` のようなもので、実行時に値を埋め込む。

```graphql
query {
  user(id: $id) {
    id
    name
    posts {
      id
      message
    }
  }

  # 同じリクエストに複数のクエリを含めることもできる
}
```

## GraphQLリクエストの例

クエリとは別に `$id` に渡す値を指定している。

```json
{
  "query": "query { user(id: $id) { id name posts { id title } } }",
  "variables": {
    "id": 1
  }
}
```

## GraphQLレスポンスの例

```json
{
  "data": {
    "user": {
      "id": 1,
      "name": "John",
      "posts": [
        {
          "id": 1,
          "message": "Hello, world!"
        },
        {
          "id": 2,
          "message": "Goodbye, world!"
        }
      ]
    }
  }
}
```

## GraphQLのメリット

- **静的型付け言語である。**
  - スキーマ**とクエリ**からTypeScriptの型定義を生成できるため、BEとFEが静的型付け言語であればBEからFEまで型安全に開発できる。
  - BEで定義したモデルからスキーマを生成し、スキーマとクエリからFEの型定義を生成するのが一般的。
- 単一のHTTPリクエストで**複数のクエリをバッチ実行できる**ため、HTTPリクエスト生成のオーバーヘッドを抑えられる。
- クエリごとに取得するフィールドを指定でき、**オーバーフェッチを防げる。**
- クエリをネストすることができる。(後述)
- スキーマにドキュメントを記述できる。Swaggerのようにドキュメントツールを併用しなくてよい。

## GraphQLのメリット

- Subscriptionを使ってBEとFE間で双方向通信ができる。(詳しくないので割愛)
- Tree-sitterパーサーやGraphQL Playground, LSP準拠のGraphQL言語サーバーなど、**開発環境が充実している。**
- **Data Loader** (後述)

## GraphQLのデメリット

- 一般的にGraphQLはWeb APIより通信量が多くなりがちである。
  - なぜならGraphQLがWeb APIの上に構築されるため。
  - APQ (Advanced Persistent Queries) というRPC機能で解決できるが、GraphQLではなくFW依存の機能である。
- 単一のHTTPリクエストで複数のクエリをバッチ実行できてしまうため、**BE負荷の制御が難しい。**
  - REST APIでは1リクエスト毎に1クエリであるため、L3でもリクエストレートを制限することで負荷を抑えることができる。
  - GraphQLではリクエストボディに含まれるクエリを解析しなければ負荷量が分からないため、L7で負荷を制御する必要がある。(後述)

## 取得するフィールドの指定

GraphQLではクエリごとに取得するフィールドを指定できる。

```graphql
type User {
  id: ID!
  name: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  message: String!
}
```

## 取得するフィールドの指定

```graphql
query {
  user(id: 1) {
    id
    name
  }
}
```

## 取得するフィールドの指定

```json
{
  "data": {
    "user": {
      "id": 1,
      "name": "John"
      // postsは取得されない
    }
  }
}
```

## フィールドの指定

```graphql
query {
  user(id: 1) {
    id
    name
    # レスポンスに posts を含める
    posts {
      id
      message
    }
  }
}
```

## フィールドの指定

```json
{
  "data": {
    "user": {
      "id": 1,
      "name": "John",
      // postsが取得される
      "posts": [
        {
          "id": 1,
          "message": "Hello, world!"
        },
        {
          "id": 2,
          "message": "Goodbye, world!"
        }
      ]
    }
  }
}
```

## クエリのネスト

GraphQLはクエリをネストすることができる。

```kotlin
data class User(
  val id: ID,
  val name: String,
) {
  fun posts(dfe: DataFetchingEnvironment, limit: Int): CompletableFuture<List<Post>> {
    // posts の取得処理
  }
}
```

## クエリのネスト

```graphql
query {
  user(id: 1) {
    id
    name
    posts(limit: 10) {
      id
      message
    }
  }
}
```

## クエリのネスト

GraphQLの構造体型が同じ型をフィールドにもつとクエリを無限にネストできる。

```graphql
type Post {
  id: ID!
  content: String!
  replies: [Post!]!
}
```

## クエリのネスト

`curl` などで直接GraphQLエンドポイントに深いネストを含むクエリを送信することで、クライアントは高負荷のHTTPリクエストを作成できてしまう。

```graphql
query {
  post(id: $id) {
    id
    content
    replies {
      id
      content
      replies {
        id
        content
        replies {
          # ...
        }
      }
    }
  }
}
```

## エラー処理

- クエリの実行中にエラーが発生してもGraphQLエンドポイントは `200 OK` を返す。
  - なぜならGraphQLのクエリはバッチ実行するため、一部のクエリでエラーが発生しても他のクエリは正常終了する可能性があるため。
- エラーを返す場合は `errors` フィールドにエラー情報を格納する。
- GraphQLクエリに構文エラーがあった場合も `errors` フィールドにエラー情報を格納する。

## エラー処理

```json
{
  "errors": [
    {
      "message": "Syntax Error: Expected Name, found <EOF>.",
      "locations": [
        {
          "line": 1,
          "column": 26
        }
      ]
    }
  ]
}
```

## エラー処理

- ドメイン例外の処理にGraphQL標準のエラー処理を使うのは好ましくないと考えている
  - `errors` フィールドはランタイムエラーも格納されるため、FEで処理すべきエラーとその他を区別する必要がありコードが冗長になる。
  - ドメインエラーをランタイムエラーと同じ `errors` フィールドで返すと、FE開発者にクエリがドメインエラーを吐きうるという情報が型情報から伝わりにくく、最悪ドメインエラーが捨てられる。

## エラー処理

- 静的型でHaskellの `Either`, あるいはRustの `Result` のようなtyped errorを実装して使うのがよい。
  - しかし、投げうるドメイン例外全てをUnionで列挙するのは非常に面倒。
  - 最低限のエラーメッセージをラップしただけのtyped errorでも、無意識にドメイン例外が捨てられる事態を軽減できる (はず)。

## エラー処理

```kotlin
data class Error(val message: String)

data class Result<out T>(val value: T?, val error: Error?) {
  companion object {
    fun <T> success(value: T): Result<T> = Result(value, null)
    fun <T> failure(error: Error): Result<T> = Result(null, error)
  }
}
```

```kotlin
Result.success(user)
Result<User>.failure(Error("User not found"))
```

## エラー処理

GraphQLスキーマにはジェネリクス型がないため、型引数ごとに型が定義される。

```graphql
type Error {
  message: String!
}

type UserResult {
  value: User
  error: Error
}
```

## セキュリティ

攻撃者は次の方法でGraphQLリクエスト当たりのBEの負荷量を大きくできる。

- バッチ実行するクエリを多くする
- ネストを深くする
- など...

そのため、リクエストレートを制限するだけの負荷対策は不十分であり、次のような負荷対策が必要である。

- バッチ実行するクエリの数を制限する
- ネストの深さを制限する
- クエリの数とネストの深さから算出されるcomplexity (複雑性) を制限する (ハイブリッド)

# Rethink DDD

- 基本的にGraphQLはプレゼンテーション層の関心事
- プレゼンテーション層の関心事はドメインモデルに影響を与えない

# Rethink DDD

- 「プレゼンテーション層の関心事はドメインモデルに影響を与えない」
- そう思ってたいた時期が俺にもありました。
- 現実は異なる
  - ドメイン層はデータフローの中間に位置するため、プレゼンテーション層のビューモデル設計はドメイン層に影響を与えうる。
    - よく見るドメイン層が最上位だったり同心円の中心に位置する図はあくまで依存の方向を表したもの。
  - 抽象化とパフォーマンスのトレードオフ
    - 例: 複数テーブルに跨るデータをどこで結合するか

## 複数テーブルに跨るデータの取得

1. BEでモデルを結合
1. SQLでテーブル結合
    - 結合パターンごとにエンティティクラスを使い分ける
    - 全ての結合パターンに対応できるエンティティクラスを定義する
        - 常に全てのテーブル結合を行う
        - テーブル結合を行わずに取得しなかったフィールドを `null` とする
1. CQRSによる読み込み系/更新系でモデルの分離

### BEでモデルを結合

- one to manyで結合する場合はほぼBEでモデルを結合することになる。
- 実装するとすればドメイン層 or ユースケース層
  - ドメイン操作に関係のない、ただプレゼンテーション層に流したいだけのデータの結合処理がドメイン層 or ユースケース層に入り込むことになり冗長に。
  - あるいは取得〜結合までをやってくれる `Service` が乱立し、モデルベースから遠ざかってゆく。
- N+1問題を意識して書く必要がある

### BEでモデルを結合

```kotlin
data class User(
  val id: Int,
  val name: String,
)

data class Profile(
  val id: Int,
  val userId: Int,
  val bio: String,
)
```

### BEでモデルを結合

```kotlin
val users = userRepository.findByIds(userIds)
val profileByUserId = profileRepository.findByUserIds(userIds).associateBy { it.userId }

users.map { user ->
  // ハッシュマップからの取得は O(1) の償却コストで行える
  val profile = profileByUserId[user.id]

  checkNotNull(profile) { "ユーザーのプロフィールが存在しません" }

  // 何らかの処理
}
```

### 結合パターンごとにエンティティクラスを使い分ける

結合パターン数に依ってはエンティティクラスの数が組み合わせ爆発を起こす。

```kotlin
data class User(val id: Int, val name: String)

data class Profile(val id: Int, val userId: Int, val bio: String)

data class Post(val id: Int, val userId: Int, val message: String)
```

```kotlin
data class UserWithProfile(val user: User, val profile: Profile)
data class UserWithPosts(val user: User, val posts: List<Post>)
data class UserWithProfileAndPosts(val user: User, val profile: Profile, val posts: List<Post>)
// ...
```

### 全ての結合パターンに対応できるエンティティクラスを定義する

1. 常に全てのテーブル結合を行う。
    - ユースケースに依らず全てのテーブル結合を行うため、パフォーマンスは最悪。
    - `Repository` の更新系メソッドを呼び出すときには全てのテーブルを更新する必要がある。
        - 読み込み系と更新系でモデルを分ける方法 (CQRS) もあるが、銀の弾丸ではない。(後述)
1. テーブル結合を行わずに取得しなかったフィールドを `null` とする
    - 論理設計レベルでnullableであるフィールドと区別しにくい。
    - 静的なnull安全でないため、使う側のコードで `null` チェックやエラー処理などの防御的プログラミングが必要になり、コードが冗長に。

### CQRSによる読み込み系/更新系でモデルの分離

- 参照系/更新系でモデルを分離するのはモデルベースであることを犠牲にしている。
  - もっとも、モデルベースで実装することが現実的に難しい処理は確実に存在する。
    - 例: モデルベースでユーザーIDの重複チェックを実装するために全ユーザーをDBからフェッチするのか、など。
  - CQRSはモデルベースでの実装が難しい箇所に限定して使うのがよい。
  - CQRSパターンを採用したからと言って、全ての実装がCQRSである必要は全くない。

### 取得方法の比較

|取得方法|モデルベース|パフォーマンス|コードの簡潔さ|
|---|---|---|---|
|BEでモデルを結合|○|○|✗|
|テーブル結合パターンごとにエンティティクラスを定義|✗|◎|○|
|常に全てのテーブル結合を行う|◎|✗|○|
|テーブル結合を行わずに取得しなかったフィールドを `null` とする|○|◎|✗|
|CQRSによる読み込み系/更新系でモデルの分離|✗|◎|○|

## 抽象化とパフォーマンスはしばしばトレードオフ

## Data Loader

- BEの**プレゼンテーション層**でデータを結合するためのGraphQLの機能
  - Data LoaderはCQRSにおけるQueryモデルをプレゼンテーション層に書くためのフレームワークと言える。
- スケジューラーによる**遅延バッチ処理**と**並列処理**
  - 遅延バッチ処理によりN+1を意識せずに直感的に書ける。
  - 並列処理によりレスポンスタイムの劣化を最小限に抑える。

## Data Loader

- プレゼンテーション層から呼び出せるため、ビュー都合でドメイン層を汚染しない。
- FEが必要としないデータはフィールドを省けばData Loaderは実行されない。
  - FEの型定義はスキーマ**とクエリ**から生成されるため、静的なnull安全が手に入る。

### Data Loaderを呼び出すフィールドの実装例

```kotlin
data class User(
  val id: ID,
  val name: String,
) {
  fun posts(dfe: DataFetchEnvironment): CompletableFuture<List<Post>> {
    return dfe.getValueFromDataLoader("PostsByUserIdDataLoader", id)
  }
}
```

### Data Loaderの実装例

- `User#posts` では単一のユーザーIDをData Loaderに渡しているのに対し、Data Loaderは複数のユーザーIDを処理しているのに注目。
  - Data Loaderのスケジューラーが遅延バッチ処理により複数のユーザーIDをまとめて処理しているため。これによりN+1を意識せずに直感的に書ける。
  - そのため、IDからエンティティの取得を行う `Repository` はバッチ取得に対応させておくのがおすすめ。

```kotlin
class PostsByUserIdDataLoader(private val postRepository: PostRepository) : KotlinDataLoader<ID, List<Post>> {
  override val dataLoaderName = "PostsByUserIdDataLoader"
  override suspend fun getDataLoader() = DataLoader<ID, List<Post>> { userIds ->
    CompletableFuture.supplyAsync {
      val postsByUserId = postRepository.findByUserIds(userIds).associateBy { it.userId }

      userIds.map { userId -> postsByUserId[userId] ?: listOf() }
    }
  }
}
```

### Data Loaderを呼び出す導出フィールドの実装例

- 非同期処理をチェーンするだけで導出フィールドを実装できる。
- `posts` と `postCount` の両方を取得しても、`posts` の取得処理は1回しか実行されない。

```kotlin
data class User(
  // ...
) {
  fun posts(dfe: DataFetchEnvironment): CompletableFuture<List<Post>> {
    return dfe.getValueFromDataLoader("PostsByUserIdDataLoader", id)
  }

  fun postCount(dfe: DataFetchEnvironment): CompletableFuture<Int> {
    return posts.thenApply { it.size }
  }
}
```

### Data Loaderもまた銀の弾丸ではない

- プレゼンテーション層でデータを結合するため、ドメインロジックでも結合データを参照したい場合は使えない。
- 結局BE側でデータを結合していることに変わりはないため、SQLのテーブル結合のパフォーマンスには及ばない。(と予測)

## Data Loaderを使う条件まとめ

以下の条件を全て満たす場合はData Loaderを使うのがよい。

- 結合したいデータはユースケースに依っては結合が不要である。
- 結合したいデータはドメインロジックで参照されない。

## 次回予告

- GraphQL KotlinでRelay準拠のConnectionの実装をする話
- NixとNixOSの原理

のいずれか。

## End
