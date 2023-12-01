<!-- headingDivider: 1 -->

# GraphQLとRelay

# Relay

- Facebookが開発したGraphQLのフレームワーク

# Relayのメリット

<!-- TODO: 書く -->

# Global Object Identification

- Relayで扱うデータは、オブジェクトタイプ `Node` を継承する。
- `Node` はオブジェクトタイプを跨ぐグローバルな識別子 (以降Global ID) をもつ。

# Global Object Identification

```graphql
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  name: String!
}

type Post implements Node {
  id: ID!
  title: String!
}
```

# Global Object Identification

- RelayクライアントではGlobal IDでデータのキャッシュを正規化して保持することで、Relayのあらゆるメリットを実現する。

# `node` クエリ

Relayでは、Global IDを引数に取りそれに対応するオブジェクトを返す `node` クエリを実装することが求められる。

```graphql
type Query {
  node(id: ID!): Node
}

# ...
```

# `node` クエリ

```graphql
query {
  node(id: $id) {
    id
    ... on User {
      name
    }
    ... on Post {
      title
    }
  }
}
```

# Global Object Identificationの実装

- UUIDはオブジェクトタイプを跨いで一意であるため、Global IDの要件を満たしている。
- しかし、実際にUUIDとGlobal IDとして `node` クエリを実装しようとすると、引数として受け取ったUUIDからどのテーブルを検索すればいいのか分からない。

# Global Object Identificationの実装

- graphql-java のRelay実装では、`<クラス名>:<エンティティID>` という形式の文字列をBase64エンコードしたものをGlobal IDとしている。
  - 例: `User:1` → `VXNlcjox`
  - 例: `Post:1` → `UG9zdDox`
- これにより `node` クエリの実装で、Global IDから検索すべきテーブルを特定できる。

# Global Object Identificationの実装

今回はクラス名の代わりにオブジェクトタイプを表すEnumを用いる。

```kotlin
enum class NodeType {
  USER,
  POST
}
```

# Global Object Identificationの実装

```kotlin
data class GlobalId(
  val nodeType: NodeType,
  val id: Long
)
```

# Global Object Identificationの実装

```kotlin
class Node(
  val nodeType: NodeType,
  val id: Long
) {
  fun toGlobalId(): GlobalId {
    return GlobalId(
      nodeType = nodeType,
      id = id
    )
  }
}
```

# Global Object Identificationの実装

```kotlin
class User(
  val id: Long,
  val name: String
) : Node(NodeType.USER, id)
```

```kotlin
class Post(
  val id: Long,
  val title: String
) : Node(NodeType.POST, id)
```

# Global Object Identificationの実装

```kotlin
// Base64エンコードされたGlobal IDをパースするextension
fun ID.toGlobalId(): GlobalId {
  val (nodeType, id) = Base64.getDecoder().decode(this).toString(Charsets.UTF_8).split(":")

  return GlobalId(
    nodeType = NodeType.valueOf(nodeType),
    id = id.toLong()
  )
}
```

# `node` クエリの実装

```kotlin
enum class NodeType {
  USER,
  POST
}
```

# `node` クエリの実装

```kotlin
data class GlobalId(
  val nodeType: NodeType,
  val id: Long
)
```

# `node` クエリの実装

```kotlin
class NodeQueryService(
  private val findUserByIdUsecase: FindUserByIdUsecase,
  private val findPostByIdUsecase: FindPostByIdUsecase
) : Query {
  fun node(id: ID): Node? {
    val globalId = id.toGlobalId()

    return when (globalId.nodeType) {
      NodeType.USER -> {
        val request = FindUserByIdUsecase.Request(id = globalId.id)
        findUserByIdUsecase.execute(request)
      }

      NodeType.POST -> {
        val request = FindPostByIdUsecase.Request(id = globalId.id)
        findPostByIdUsecase.execute(request)
      }
    }
  }
}
```

# ページングの実装

ページングには大まかに2つの実装方針がある。

# オフセット法

```sql
SELECT * FROM users LIMIT 10 OFFSET 20
```

- 実装がシンプル。
- 任意のページ数を指定して検索できる。
- 多くのDBMSで内部的に `offset+limit` 件分のデータを取得してから、`offset` 件分を捨てるよう実装されている。
- そのため、ページ数が大きくなるにつれパフォーマンスが低下する。

# シーク法

```sql
SELECT * FROM users WHERE id > ? ORDER BY id LIMIT 10
```

- 検索条件が増えるにつれ実装が煩雑に。
- 任意のページ数を指定して検索することはできない。
- ページ数に関わらず高速な検索ができる。
- 検索条件に含まれるカラムでソートする必要がある。

# シーク法

- ソートカラムの組み合わせは候補キー (テーブル内でレコードを一意に特定できるカラムの組み合わせ) である必要がある。
  - テーブル内の行に一意のキーが割り当てられている場合、そのキーをソートカラムの最後に含めておけばよい。
- ページの最初と最後のデータのソートカラムの値をクライアント側で保持しておく必要がある。

# オフセット法 vs シーク法

SQLパフォーマンス詳解の原著者のスライド Pagination Done the Right Way ではオフセット法とシーク法の詳細なパフォーマンス比較がなされている。
https://www.slideshare.net/MarkusWinand/p2d2-pagination-done-the-postgresql-way

# Connection

Relayでは、シーク法ページングにConnectionというGraphQLクエリインターフェイスを用いるよう推奨されている。

| 引数 | 説明 |
| --- | --- |
| `first` | (検索方向が正順の場合) 取得する件数 |
| `after` | ページングの開始位置 |
| `last` | (検索方向が逆順の場合) 取得する件数 |
| `before` | ページングの終了位置 |

# Connection

```graphql
type Query {
  users(first: Int, after: String, last: Int, before: String): UserConnection!
}
```

# Connection

- `first` が指定された場合は正順、`last` が指定された場合は逆順で検索する。
- `first` が指定された場合は `after` が指すデータから、`after` が指定されていない場合は先頭から検索を開始する。
- `last` が指定された場合は `before` が指すデータから、`before` が指定されていない場合は末尾から検索を開始する。

# Connection

```graphql
type UserConnection {
  pageInfo: PageInfo!
  edges: [UserEdge!]!
}
```

| フィールド | 説明 |
| --- | --- |
| `pageInfo` | ページに関する情報 |
| `edges` | ページに含まれるデータを指すEdge (後述) |

# PageInfo

```graphql
type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

# PageInfo

| フィールド | 型 | 説明 |
| --- | --- | --- |
| `hasNextPage` | `Boolean!` | 次のページが存在するか |
| `hasPreviousPage` | `Boolean!` | 前のページが存在するか |
| `startCursor` | `String` | ページの最初のデータを指すCursor (後述) |
| `endCursor` | `String` | ページの最後のデータを指すCursor (後述) |

# Edge

```graphql
type UserEdge {
  node: User!
  cursor: String!
}
```

- Relayではあらゆるデータをグラフ構造としてみなす。
- Edgeはグラフ構造における辺を表す。
  - 例えば、Connectionに含まれるEdgeはConnectionから取得したデータ (Node) への辺である。

# Edge

Edge (辺) には必要に応じて `node`, `cursor` 以外のフィールドを追加してもよい。

```graphql
type User {
  id: ID!
  name: String!
  # 関連するユーザー
  relatedUsers: [RelatedUserEdge!]!
}
```

# Edge

```graphql
enum UserRelation {
  FOLLOWEE
  FOLLOWER
}

type RelatedUserEdge {
  node: User!
  cursor: String!

  # 関連するユーザーとの関係
  relation: UserRelation!
}
```

# Cursor

Cursorはシーク法における検索開始 or 終了位置を示す。

# Cursor

CursorにエンティティIDを指定することは推奨しない。

- エンティティIDを指定すると、そのエンティティが削除された場合に検索位置が破綻する。
- 一度エンティティIDからソートカラムの値を取得する必要があるため、検索パフォーマンスにオーバーヘッドが生じる。

# Cursor

Cursorにはソートカラムの値を含めておくのがおすすめ。

# Cursor

例えば、User (ユーザー) を**名前の昇順 → IDの昇順**でソートするケースを考える。

```json
{
  "id": 1,
  "name": "Alice",
  "createdAt": "2019-01-01T00:00:00Z",
  "updatedAt": "2019-01-01T00:00:00Z"
}
```

# Cursor

ソートカラムの情報を含んだ文字列を作成する。

```json
{
  "id": 1,
  "name": "Alice",
}
```

# Cursor

後の実装のために、ソートカラムの情報に加えて `nodeType` (Nodeの実装オブジェクトタイプ) を含ませておく。

```json
{
  "nodeType": "USER",
  "id": 1,
  "name": "Alice"
}
```

# Cursor

ソートカラムの情報を含んだ文字列を必要に応じてBase64エンコードしたものをCursorとする。

```
eyJub2RlVHlwZSI6IlVTRVIiLCJpZCI6MSwibmFtZSI6IkFsaWNlIn0=
```

# PageInfo

Connectionから返る `pageInfo` はこのようになる。

```json
{
  "hasNextPage": true,
  "hasPreviousPage": false,
  "startCursor": "eyJub2RlVHlwZSI6IlVTRVIiLCJpZCI6MSwibmFtZSI6IkFsaWNlIn0=",
  "endCursor": "eyJub2RlVHlwZSI6IlVTRVIiLCJpZCI6MiwibmFtZSI6IkJvYiJ9",
}
```

# Edges

Connectionから返る `edges` はこのようになる。

```json
[
  {
    "node": {
      "id": "VXNlcjox",
      "name": "Alice"
    },
    "cursor": "eyJub2RlVHlwZSI6IlVTRVIiLCJpZCI6MSwibmFtZSI6IkFsaWNlIn0=="
  },
  {
    "node": {
      "id": "VXNlcjoy",
      "name": "Bob"
    },
    "cursor": "eyJub2RlVHlwZSI6IlVTRVIiLCJpZCI6MiwibmFtZSI6IkJvYiJ9"
  }
]
```

# Edgeの実装

`node` はData Loaderで取得することで

- `node` を一緒に取得したいユースケース
- `cursor` だけを保持しておき、必要なタイミングで別のクエリで `node` を遅延取得したいユースケース

の両方に対応できる。

# Edgeの実装

```kotlin
data class UserEdge(
  val cursor: Cursor
) {
  fun node(): CompletableFuture<User> {
    // Data Loaderでnodeを取得する処理
  }
}
```

# Cursorの実装

- CursorはJSON文字列であるため、型がない。
- ドメイン層では型安全を保ちたいため、Cursorをプレゼンテーション層でパースしてキャストする。

# Cursorの定義 (ドメイン層)

```kotlin
interface Cursor

data class UserCursor(
  val id: Long,
  val name: String
) : Cursor
```

# Cursorの定義 (プレゼンテーション層)

```kotlin
enum class NodeType {
  USER,
  POST
}

// ドメイン層と名前衝突を避けるために接尾辞に `Object` を付ける
data class CursorObject(
  // どのNodeクラスのオブジェクトへのCursorかを特定するためにNodeタイプを含める
  val nodeType: NodeType,
  val payload: Map<String, Any>
)
```

# Cursorの生成

```kotlin
fun User.toCursor(): UserCursor {
  return UserCursor(
    id = id,
    name = name
  )
}
```

# Cursorの変換 (ドメイン層 → プレゼンテーション層)

```kotlin
fun UserCursor.toCursorObject(): CursorObject {
  return CursorObject(
    nodeType = NodeType.USER,
    payload = mapOf(
      "id" to id,
      "name" to name
    )
  )
}
```

# Cursorの変換 (プレゼンテーション層 → ドメイン層)

```kotlin
fun CursorObject.toDomainCursor(): Cursor {
  return when (nodeType) {
    NodeType.USER -> UserCursor(
      id = payload["id"] as Long,
      name = payload["name"] as String
    )

    NodeType.POST -> PostCursor(
      id = payload["id"] as Long,
      title = payload["title"] as String
    )
  }
}
```

# Cursorカスタムスカラーの定義

```kotlin
object CursorCoercing : Coercing<Cursor, String> {
  override fun parseValue(input: Any, graphQLContext: GraphQLContext, locale: Locale) = runCatching {
    val json = Base64.getDecoder().decode(input as String).toString(Charsets.UTF_8)
    val payload = jacksonObjectMapper().readValue<Map<String, Any>>(json)
    val nodeType = NodeType.valueOf(payload.nodeType)

    Cursor(
      nodeType = nodeType,
      payload = payload.filterKeys { it != "nodeType" }
    )
  }.getOrElse {
    throw CoercingParseValueException("Invalid cursor: $input")
  }

  // ...
}
```

# Cursorカスタムスカラーの定義

```kotlin
object CursorCoercing : Coercing<Cursor, String> {
  // ...

  override fun serialize(dataFetcherResult: Any): String {
    val cursor = dataFetcherResult as Cursor
    val json = jackonObjectMapper().writeValueAsString(
      cursor.payload + ("nodeType" to cursor.nodeType.name)
    )

    return Base64.getEncoder().encodeToString(json.toByteArray(Charsets.UTF_8))
  }
}
```

# CursorカスタムスカラーをInputで使う

```graphql
input SearchUsersInput {
  first: Int
  after: Cursor
  last: Int
  before: Cursor
}
```

# CursorカスタムスカラーをInputで使う

```kotlin
val request = SearchUsersUsecase.Request(
  first = input.first,
  after = input.after?.toDomainCursor() as UserCursor?,
  last = input.last,
  before = input.before?.toDomainCursor() as UserCursor?
)
```

# CursorカスタムスカラーをInputで使う

```json
{
  "query": "query ($input: SearchUsersInput!) { users(input: $input) { ... } }",
  "variables": {
    "input": {
      "first": 10,
      "after": "eyJpZCI6MSwibmFtZSI6IkFsaWNlIn0=",
      "last": null,
      "before": null
    }
  }
}
```

# Connectionの実装

以下の要件を満たすConnectionの実装を考える。

- Relay Connectionの仕様に準拠する正順 or 逆順ページングを実装する。
- ソート順序を指定できる。

# Connectionの実装

- Relay Connectionの仕様には `first` と `last` のどちらも指定しない場合について仕様の記述がない。
- 仕様通りだと、データを全件取得することになる。
- しかし、そのような処理を実装する訳にはいかないので、今回は `first` or `last` のどちらか一方は必須とする。

# Connectionの実装

つまりConnectionの引数には以下があれば良い。

- 検索方向 (direction)
- 検索開始 or 終了位置 (cursor)
- 取得件数 (count)

# Connectionの実装

- SQLの `LIMIT` 句は先頭からの件数を指定するため、逆順に検索したい場合は工夫が必要。
- 検索方向が逆順の場合は、ソート順序を逆にして `SELECT` して `LIMIT` 句を適用した後に、結果を再度逆順にする。

# Connectionの実装

| ソート順序 | 検索方向 | `ORDER BY` 句に指定する順序 |
| --- | --- | --- |
| 昇順 | 正順 | `ASC` |
| 昇順 | 逆順 | `DESC` |
| 降順 | 正順 | `DESC` |
| 降順 | 逆順 | `ASC` |

# ConnectionのSQL

| 引数 | 値 |
| --- | --- |
| ソート順序 | 名前の昇順 → IDの昇順 |
| 検索方向 | 正順 |
| 検索開始位置 | `null` |
| 取得件数 | `10` |

```sql
SELECT * FROM users
ORDER BY name ASC, id ASC
LIMIT 10
```

# ConnectionのSQL

| 引数 | 値 |
| --- | --- |
| ソート順序 | 名前の昇順 → IDの昇順 |
| 検索方向 | 正順 |
| 検索開始位置 | non-null |
| 取得件数 | `10` |

```sql
SELECT * FROM users
WHERE (name > ?) OR (name = ? AND id > ?)
ORDER BY name ASC, id ASC
LIMIT 10
```

# ConnectionのSQL

| 引数 | 値 |
| --- | --- |
| ソート順序 | 名前の降順 → IDの降順 |
| 検索方向 | 正順 |
| 検索開始位置 | non-null |
| 取得件数 | `10` |

```sql
SELECT * FROM users
WHERE (name < ?) OR (name = ? AND id < ?)
ORDER BY name DESC, id DESC
LIMIT 10
```

# ConnectionのSQL

| 引数 | 値 |
| --- | --- |
| ソート順序 | 名前の降順 → IDの降順 |
| 検索方向 | 逆順 |
| 検索開始位置 | non-null |
| 取得件数 | `10` |

# ConnectionのSQL

```sql
SELECT t.*
FROM (
  SELECT
    *,
    row_number() OVER () AS rownum
  FROM users
  WHERE (name > ?) OR (name = ? AND id > ?)
  -- 降順ソートを逆順に検索するので、ORDER BY句はASC
  ORDER BY name ASC, id ASC
  LIMIT 10
) t
-- 逆順になるように検索結果を反転
ORDER BY t.rownum DESC
```

# Connectionの実装

検索結果の反転はアプリケーション側で行ってもよい。

# PageInfoの取得

`hasNextPage` と `hasPreviousPage` を取得する。

# PageInfoの取得

- 検索方向が正順の場合の `hasNextPage` と検索方向が逆順の場合の `hasPreviousPage` は、検索と同じSQLで取得できる。
- 具体的には `LIMIT` 句に取得件数より1多い値を指定して検索を行い、検索結果の件数が `取得件数+1` であれば `hasNextPage` or `hasPreviousPage` は `true` である。

# PageInfoの取得

- 検索方向が正順の場合の `hasPreviousPage` と検索方向が逆順の場合の `hasNextPage` は、検索と別のSQLで取得する必要がある。
- `WHERE` 句には検索に使用した条件を `NOT` で否定したものを指定する。
- `ORDER BY` 句は検索と逆の順序を指定する。
- `LIMIT` 句には1を指定する。
- 検索結果の件数が1以上であれば `hasPreviousPage` or `hasNextPage` は `true` である。

# PageInfoの取得

```sql
SELECT count(1)
FROM users
WHERE NOT ((name > ?) OR (name = ? AND id > ?))
ORDER BY name DESC, id DESC
LIMIT 1
```

# PageInfoの取得

検索方向が逆順の場合にアプリケーション側で検索結果を反転するように実装している場合は、そこで一緒に `hasNextPage` と `hasPreviousPage` をスワップすればよい。

```kotlin
if (results.size > count) {
  hasNextPage = true
  results = results.dropLast(1)
}

if (direction == Direction.BACKWARD) {
  results = results.reversed()

  hasNextPage = hasPreviousPage.also { hasPreviousPage = hasNextPage }
}
```

# End
