# TanStack Router Fetch API 調査レポート

## 概要

TanStack Routerは、型安全なルーティングと組み込みのデータロード機能を提供するReact向けのルーターライブラリです。
従来のReact RouterやNext.jsのルーターとは異なり、ルートレベルでのデータフェッチ・キャッシュ制御・Stale-While-Revalidate（SWR）セマンティクスを標準機能として提供しています。

本レポートでは、TanStack Routerにおけるデータフェッチ関連のAPI・機能について調査した結果をまとめています。

---

## バージョン情報

| パッケージ | バージョン |
|---|---|
| `@tanstack/react-router` | 1.170.11（2026年6月時点の最新安定版） |
| `@tanstack/router-core` | 1.169.2 |
| `@tanstack/router-plugin` | 1.168.11 |

> 参考: [@tanstack/react-router - npm](https://www.npmjs.com/package/@tanstack/react-router)、[Releases · TanStack/router](https://github.com/TanStack/router/releases)

---

## Loaderとデータフェッチの仕組み

### Loaderの基本

TanStack RouterはルートオブジェクトにLoaderを定義することで、ナビゲーション時の自動データフェッチを実現します。
Loaderはルートがマッチした際に呼び出され、返した値は`useLoaderData()`フックを通じてコンポーネントから参照できます。

**全Loaderは並列で実行されます。** これにより、レイアウトのロードを待たずに下層ルートのLoaderも同時に処理が開始されます。

#### Loaderの基本構文

```tsx
// シンプル形式
export const Route = createFileRoute('/posts')({
  loader: () => fetchPosts(),
})

// オブジェクト形式（staleReloadModeなどの設定が必要な場合）
export const Route = createFileRoute('/posts')({
  loader: {
    handler: () => fetchPosts(),
    staleReloadMode: 'blocking',
  },
})
```

#### Loader関数のパラメータ

Loader関数は以下のプロパティを持つオブジェクトを受け取ります。

| パラメータ | 説明 |
|---|---|
| `params` | パスパラメータ（e.g. `/posts/$id` の `id`） |
| `deps` | `loaderDeps`関数から返される依存関係オブジェクト |
| `context` | 親ルートとのコンテキストをマージしたオブジェクト |
| `abortController` | ルートのアンロード時にキャンセルされるシグナル |
| `preload` | プリロード中かどうかを示すboolean |
| `cause` | Loader実行の原因（`'enter'` / `'preload'` / `'stay'`） |

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params, context, abortController, preload }) => {
    return fetchPost(params.postId, {
      signal: abortController.signal,
      maxAge: preload ? 10_000 : 0,
    })
  },
})
```

#### beforeLoadとLoaderの違い

TanStack Routerではデータロードのライフサイクルが2段階に分かれています。

```mermaid
flowchart LR
    A[ナビゲーション] --> B[beforeLoad]
    B --> C[loader]
    C --> D[コンポーネント描画]
```

| 機能 | `beforeLoad` | `loader` |
|---|---|---|
| 実行順序 | 最初に実行 | beforeLoadの後に実行 |
| SearchParamsへのアクセス | ✅ | ❌（型安全なアクセスはloaderDeps経由のみ） |
| loaderDepsへのアクセス | ❌ | ✅ |
| 用途 | 認証チェック・リダイレクト・コンテキスト注入 | データ取得 |

```tsx
export const Route = createFileRoute('/dashboard')({
  beforeLoad: async ({ context }) => {
    const user = await validateAuth()
    if (!user) throw redirect({ to: '/login' })
    return { user }
  },
  loader: async ({ context }) => {
    // context.user が利用可能
    return fetchDashboardData(context.user.id)
  },
})
```

> 参考: [Data Loading | TanStack Router Docs](https://tanstack.com/router/latest/docs/guide/data-loading)

---

### loaderDepsによる依存関係管理

検索パラメータなどの値をLoader関数に渡し、かつキャッシュキーとして機能させるためには`loaderDeps`を使用します。

**`loaderDeps`を経由せずにSearch Paramsにアクセスすると、キャッシュが正しく機能しません。**

```tsx
export const Route = createFileRoute('/posts')({
  validateSearch: z.object({
    offset: z.number().int().nonnegative().catch(0),
    limit: z.number().int().positive().catch(10),
  }),
  // loaderDeps: キャッシュキーに含める値を明示する
  loaderDeps: ({ search: { offset, limit } }) => ({ offset, limit }),
  // loader: depsからパラメータを受け取る
  loader: ({ deps: { offset, limit } }) =>
    fetchPosts({ offset, limit }),
})
```

キャッシュキーは以下の2つの情報から生成されます。

- ルートの完全に解析されたパス名
- `loaderDeps`関数が返したオブジェクト

> 参考: [Data Loading | TanStack Router Docs](https://tanstack.com/router/latest/docs/guide/data-loading)

---

### staleTime / gcTime / shouldReload

TanStack Routerは組み込みのSWRキャッシュを提供しており、以下のオプションでキャッシュの挙動を制御できます。

#### キャッシュ制御オプション一覧

| オプション | デフォルト値 | 説明 |
|---|---|---|
| `staleTime` | 0ms | ルートデータが「新鮮」と見なされる期間（ミリ秒）。この期間内は再フェッチをスキップ |
| `gcTime` | 1,800,000ms（30分） | ルートデータがキャッシュに保持される期間。期限後はGC対象になる |
| `preloadStaleTime` | 30,000ms（30秒） | プリロード後のstaleTime |
| `preloadGcTime` | 1,800,000ms（30分） | プリロード用のgcTime |
| `shouldReload` | — | ルートがリロードされるべきかを決定するbooleanまたは関数 |
| `staleReloadMode` | `'background'` | staleなデータをバックグラウンドで更新（`'background'`）か、ブロッキングで更新（`'blocking'`）か |

#### staleTimeの動作

デフォルトの`staleTime: 0`は「データは常にstale」を意味します。ルートへの再訪問時に毎回バックグラウンドで再フェッチが発生します。

```tsx
export const Route = createFileRoute('/posts')({
  loader: () => fetchPosts(),
  staleTime: 10_000, // 10秒間は再フェッチしない
})
```

#### gcTimeの動作

ルートがアンマウントされた後もデータをキャッシュに保持する期間を指定します。

```tsx
export const Route = createFileRoute('/posts')({
  loader: () => fetchPosts(),
  gcTime: 0, // アンマウント後は即座にキャッシュを削除
})
```

#### shouldReloadの使い方

`shouldReload`はstaleTimeやloaderDepsによる制御を超えた、カスタムのリロード判定を実装するためのオプションです。
boolean値、またはloaderと同じ引数型（`LoaderFnContext`）を受け取りbooleanを返す関数を指定できます。

```tsx
// shouldReload: false の場合、初回ナビゲーションか deps 変更時のみリロード
export const Route = createFileRoute('/posts')({
  loader: () => fetchPosts(),
  gcTime: 0,
  shouldReload: false,
})

// shouldReload: 関数の場合、条件に応じてリロードを制御
// 引数はloaderと同じ LoaderFnContext 型（params / deps / context / abortController など）
export const Route = createFileRoute('/dashboard')({
  loader: ({ context }) => fetchDashboard(context.user.id),
  shouldReload: ({ context }) => context.auth.isAuthenticated === false,
})
```

#### グローバル設定

ルーター全体でデフォルト値を設定することも可能です。

```tsx
const router = createRouter({
  routeTree,
  defaultStaleTime: 5_000,
  defaultGcTime: 600_000,
  defaultPreload: 'intent',
})
```

> 参考: [Data Loading | TanStack Router Docs](https://tanstack.com/router/latest/docs/guide/data-loading)、[Data Loading and Caching | DeepWiki](https://deepwiki.com/tanstack/router/2.3-data-loading-and-caching)

---

## refetch・再フェッチのAPI

### router.invalidate()

`router.invalidate()`は最も基本的な再フェッチのトリガーです。
現在マッチしているルートの`beforeLoad`と`loader`関数を強制的に再実行します。
`staleTime`の設定に関わらず、強制的に再フェッチが発生します。

#### 基本的な使い方

```tsx
const router = useRouter()

const addTodo = async (todo: Todo) => {
  try {
    await api.addTodo()
    router.invalidate() // バックグラウンドで非同期に再フェッチ
  } catch {
    // エラー処理
  }
}
```

再フェッチはバックグラウンドで実行されます。既存のデータは新しいデータが取得できるまで引き続き提供されます。

#### 同期的に完了を待つ場合

```tsx
const addTodo = async (todo: Todo) => {
  try {
    await api.addTodo()
    await router.invalidate({ sync: true }) // 全Loaderが完了するまで待機
  } catch {
    // エラー処理
  }
}
```

#### filterオプションで特定のルートのみ無効化

```tsx
router.invalidate({
  filter: (route) => {
    return (
      route.routeId === '/app/tasks/' ||
      (route.routeId === '/app/tasks/$taskId/' &&
        route.params.taskId === taskId)
    )
  },
})
```

#### invalidateのオプション一覧

| オプション | 型 | 説明 |
|---|---|---|
| `filter` | `(match) => boolean` | trueを返したマッチのみ無効化。未指定の場合は全マッチを無効化 |
| `sync` | `boolean` | trueの場合、全Loaderが完了するまでPromiseがresolveされない |
| `forcePending` | `boolean` | trueの場合、無効化されたマッチをエラー状態であっても`pending`状態に設定する |

> 参考: [Data Mutations | TanStack Router Docs](https://tanstack.com/router/v1/docs/guide/data-mutations)、[Router type | TanStack Router React Docs](https://tanstack.com/router/latest/docs/api/router/RouterType)

---

### router.load()

`router.load()`は現在マッチしている全ルートをロードし、全てレンダリング可能な状態になるまでresolveします。

**`router.invalidate()`との重要な違い:**  `router.load()`は`staleTime`を尊重します。データがまだ「新鮮」な場合、強制的な再フェッチは行いません。強制的に再フェッチしたい場合は`router.invalidate()`を使用してください。

```tsx
// ページ遷移後にデータがロード済みであることを確認する用途など
await router.load()
```

> 参考: [Router type | TanStack Router React Docs](https://tanstack.com/router/latest/docs/api/router/RouterType)

---

### useRouter()経由の操作

コンポーネント内から`useRouter()`フックでRouterインスタンスを取得し、`invalidate()`や`load()`を実行できます。

```tsx
import { useRouter } from '@tanstack/react-router'

function MutationButton() {
  const router = useRouter()

  const handleMutation = async () => {
    await performMutation()
    // 全アクティブルートのLoaderを無効化・再実行
    router.invalidate()
  }

  return <button onClick={handleMutation}>更新</button>
}
```

---

### navigate()による再フェッチトリガー

`navigate()`は直接の再フェッチAPIではありませんが、パスパラメータや検索パラメータが変化するナビゲーションを行うことで、`loaderDeps`の変化を通じてLoaderを再実行できます。

```tsx
import { useNavigate } from '@tanstack/react-router'

function Pagination() {
  const navigate = useNavigate()

  return (
    <button
      onClick={() =>
        navigate({
          to: '/posts',
          search: (prev) => ({ ...prev, offset: prev.offset + 10 }),
        })
      }
    >
      次のページ
    </button>
  )
}
```

> 参考: [Navigation | TanStack Router Docs](https://tanstack.com/router/v1/docs/guide/navigation)

---

## routerContextとloaderDataの仕組み

### routerContext

`routerContext`は依存性注入（Dependency Injection）の仕組みとして機能します。
`createRouter`にコンテキストオブジェクトを渡すことで、全ルートのLoader・beforeLoadから参照できるようになります。

```tsx
// ルートコンテキストの型定義
interface MyRouterContext {
  fetchPosts: () => Promise<Post[]>
  auth: AuthState
  queryClient: QueryClient
}

// ルーターの初期化時にコンテキストを注入
const router = createRouter({
  routeTree,
  context: {
    fetchPosts,
    auth: authState,
    queryClient,
  },
})

// ルート側でコンテキストを利用
export const Route = createFileRoute('/posts')({
  loader: ({ context: { fetchPosts } }) => fetchPosts(),
})
```

#### 認証チェックでの活用例

```tsx
// root.tsx
export const Route = createRootRouteWithContext<MyRouterContext>()({
  beforeLoad: async ({ context }) => {
    const user = await context.auth.getUser()
    if (!user) throw redirect({ to: '/login' })
    return { user }
  },
})
```

> 参考: [Router Context | TanStack Router React Docs](https://tanstack.com/router/v1/docs/framework/react/guide/router-context)、[Context Inheritance in TanStack Router | tkdodo.eu](https://tkdodo.eu/blog/context-inheritance-in-tan-stack-router)

---

### loaderData

Loaderが返したデータは、各ルートの`useLoaderData()`フックで型安全に参照できます。

```tsx
// Routeオブジェクトから直接利用（推奨）
function PostsComponent() {
  const posts = Route.useLoaderData()
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}

// コンポーネントツリーの深い位置ではgetRouteApiを使用
import { getRouteApi } from '@tanstack/react-router'

const routeApi = getRouteApi('/posts')

function DeepComponent() {
  const posts = routeApi.useLoaderData()
  return <div>{posts.length} posts</div>
}
```

`useLoaderData()`は構造共有（Structural Sharing）をサポートしており、変更されていないネストされた値の参照等価性が維持されます。これにより不要な再レンダリングを防止できます。

> 参考: [useLoaderData hook | TanStack Router React Docs](https://tanstack.com/router/latest/docs/api/router/useLoaderDataHook)

---

## useLoaderData() / useSearch()とfetchの関係

### useLoaderData()

`useLoaderData()`はLoaderが返したデータを参照するためのhookです。フェッチ自体は行わず、Loaderがキャッシュしたデータを読み取ります。

**selectオプション**を使うことで、データの一部だけを購読し、不要な再レンダリングを抑制できます。

```tsx
function PostTitle() {
  const title = Route.useLoaderData({ select: (data) => data.post.title })
  return <h1>{title}</h1>
}
```

---

### useSearch()

`useSearch()`は現在のルートのSearch Paramsを返すhookです。
フェッチとの直接的な関係はありませんが、Search Paramsがloaderのトリガーになる場合は`loaderDeps`と組み合わせて使用します。

```tsx
export const Route = createFileRoute('/posts')({
  validateSearch: z.object({
    page: z.number().int().positive().catch(1),
    sort: z.enum(['asc', 'desc']).catch('asc'),
  }),
  loaderDeps: ({ search: { page, sort } }) => ({ page, sort }),
  loader: ({ deps }) => fetchPosts(deps),
  component: PostsList,
})

function PostsList() {
  const { page, sort } = Route.useSearch()
  const posts = Route.useLoaderData()

  return (
    <div>
      <p>Page: {page}, Sort: {sort}</p>
      <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
    </div>
  )
}
```

> 参考: [Search Params | TanStack Router React Docs](https://tanstack.com/router/v1/docs/framework/react/guide/search-params)、[useSearch hook | TanStack Router Docs](https://tanstack.com/router/latest/docs/framework/react/api/router/useSearchHook)

---

## TanStack Queryとの連携

### セットアップ

TanStack Queryを使う場合、`QueryClient`をルーターコンテキスト経由で全ルートに共有するのが推奨パターンです。
また、TanStack Router自体のキャッシュをバイパスするために、`defaultPreloadStaleTime: 0`を設定します。

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { createRouter } from '@tanstack/react-router'

const queryClient = new QueryClient()

const router = createRouter({
  routeTree,
  context: { queryClient },
  defaultPreloadStaleTime: 0, // Routerのキャッシュを無効化してQueryに委ねる
})

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <RouterProvider router={router} context={{ queryClient }} />
    </QueryClientProvider>
  )
}
```

> 参考: [TanStack Router and Query | tkdodo.eu](https://tkdodo.eu/blog/tan-stack-router-and-query)

---

### ensureQueryData / prefetchQuery

#### ensureQueryData（推奨パターン）

`queryClient.ensureQueryData()`はLoaderの中でQueryのキャッシュを「プライム」するために使用します。
データがキャッシュ済みの場合は即座に返し、未ロードの場合はフェッチします。

```tsx
const postsQueryOptions = queryOptions({
  queryKey: ['posts'],
  queryFn: () => fetchPosts(),
})

export const Route = createFileRoute('/posts')({
  loader: ({ context: { queryClient } }) =>
    queryClient.ensureQueryData(postsQueryOptions),
  component: PostsPage,
})

function PostsPage() {
  // useLoaderDataではなく、useSuspenseQueryを使用する
  const { data: posts } = useSuspenseQuery(postsQueryOptions)
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

**重要:** TanStack Queryと組み合わせる場合は`Route.useLoaderData()`ではなく`useSuspenseQuery()`または`useQuery()`を使用してください。
`useLoaderData()`を使うと、TanStack QueryのQuery Observer（自動再フェッチ、ウィンドウフォーカス時の再フェッチなど）が機能しません。

> 参考: [TanStack Router and Query | tkdodo.eu](https://tkdodo.eu/blog/tan-stack-router-and-query)

#### prefetchQuery

`queryClient.prefetchQuery()`はLoaderの返り値をawaitしない場合に使用します。
データ取得を開始しつつ、コンポーネント描画をブロックしない非同期なプリフェッチを実現します。

```tsx
export const Route = createFileRoute('/posts')({
  loader: async ({ context: { queryClient }, deps }) => {
    // awaitしないため、ページ描画をブロックしない
    queryClient.prefetchQuery(postsQueryOptions(deps.page))
  },
})
```

`prefetchQuery`と`ensureQueryData`の違い：

| | `prefetchQuery` | `ensureQueryData` |
|---|---|---|
| 返り値 | `Promise<void>` | `Promise<TData>` |
| await効果 | ブロックしない | ブロックする |
| staleTimeの扱い | staleTimeを尊重 | キャッシュにデータがあれば返す |
| 用途 | バックグラウンドプリフェッチ | ブロッキングなデータ取得 |

> 参考: [Prefetching & Router Integration | TanStack Query React Docs](https://tanstack.com/query/v5/docs/framework/react/guides/prefetching)

---

### ルーターとクエリキャッシュの統合

#### router.invalidate()とqueryClient.invalidateQueries()の違い

| | `router.invalidate()` | `queryClient.invalidateQueries()` |
|---|---|---|
| 対象 | TanStack RouterのLoaderキャッシュ | TanStack QueryのQueryキャッシュ |
| 粒度 | ルート単位（粗い） | クエリキー単位（細かい） |
| 推奨用途 | TanStack Router単体での使用時 | TanStack Queryと組み合わせる場合 |

TanStack QueryをLoaderと組み合わせている場合、ミューテーション後の無効化は`queryClient.invalidateQueries()`で行うのが基本です。
`router.invalidate()`を実行すると、Loaderが再実行されて`ensureQueryData`が呼ばれますが、すでにキャッシュに新鮮なデータがある場合はネットワークリクエストが発生しません。

```tsx
// ミューテーション後の無効化パターン（推奨）
const router = useRouter()
const queryClient = useQueryClient()

const handleAddPost = async (data: NewPost) => {
  await api.createPost(data)

  // Queryのキャッシュを無効化（細粒度）
  await queryClient.invalidateQueries({ queryKey: ['posts'] })

  // Routerのキャッシュも無効化する場合（Loaderを再実行したい場合）
  router.invalidate()
}
```

> 参考: [TanStack Query Integration | TanStack Router Docs](https://tanstack.com/router/v1/docs/integrations/query)、[Loading Data with TanStack Router: react-query | Frontend Masters Blog](https://frontendmasters.com/blog/tanstack-router-data-loading-2/)

---

## コードサンプル

### 基本的なデータフェッチとキャッシュ制御

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { z } from 'zod'

export const Route = createFileRoute('/posts')({
  validateSearch: z.object({
    offset: z.number().int().nonnegative().catch(0),
    limit: z.number().int().positive().catch(10),
  }),
  loaderDeps: ({ search: { offset, limit } }) => ({ offset, limit }),
  loader: async ({ deps: { offset, limit }, abortController }) => {
    return fetchPosts({
      offset,
      limit,
      signal: abortController.signal,
    })
  },
  staleTime: 30_000,    // 30秒間はキャッシュを新鮮と判断
  gcTime: 5 * 60_000,   // 5分間キャッシュを保持
  component: PostsList,
})

function PostsList() {
  const posts = Route.useLoaderData()
  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

---

### ミューテーション後のrouter.invalidate()

```tsx
import { useRouter } from '@tanstack/react-router'

function CreatePostForm() {
  const router = useRouter()

  const handleSubmit = async (data: NewPost) => {
    await api.createPost(data)
    // 全アクティブルートのLoaderを再実行（バックグラウンド）
    router.invalidate()
  }

  const handleSubmitSync = async (data: NewPost) => {
    await api.createPost(data)
    // Loaderの完了を待つ（ブロッキング）
    await router.invalidate({ sync: true })
    // この行はLoaderが完了してから実行される
    console.log('データ更新完了')
  }

  return <form onSubmit={handleSubmit}>{/* フォームの内容 */}</form>
}
```

---

### TanStack Queryとの統合

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { queryOptions, useSuspenseQuery } from '@tanstack/react-query'

const postsQueryOptions = (page: number) =>
  queryOptions({
    queryKey: ['posts', page],
    queryFn: () => fetchPosts({ page }),
  })

export const Route = createFileRoute('/posts')({
  validateSearch: z.object({
    page: z.number().int().positive().catch(1),
  }),
  loaderDeps: ({ search: { page } }) => ({ page }),
  loader: ({ context: { queryClient }, deps: { page } }) =>
    queryClient.ensureQueryData(postsQueryOptions(page)),
  component: PostsPage,
})

function PostsPage() {
  const { page } = Route.useSearch()
  // useLoaderDataではなくuseSuspenseQueryを使用する
  const { data: posts } = useSuspenseQuery(postsQueryOptions(page))

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

---

### selectを使った最適化

```tsx
// selectで購読するデータを絞り込み、不要な再レンダリングを防止
function PostTitle({ postId }: { postId: string }) {
  const title = Route.useLoaderData({
    select: (data) => data.posts.find((p) => p.id === postId)?.title,
  })
  return <h2>{title}</h2>
}
```

---

## アンチパターン

### 1. loaderDepsで全検索パラメータを返す

```tsx
// ❌ アンチパターン: search全体を返すと使用しないパラメータの変化でも再フェッチが発生する
export const Route = createFileRoute('/posts')({
  loaderDeps: ({ search }) => search, // 全パラメータが依存関係になる
  loader: ({ deps }) => fetchPosts({ page: deps.page }), // pageしか使っていない
})

// ✅ 推奨: 実際に使用するパラメータのみを含める
export const Route = createFileRoute('/posts')({
  loaderDeps: ({ search: { page } }) => ({ page }),
  loader: ({ deps: { page } }) => fetchPosts({ page }),
})
```

> 参考: [Data Loading | TanStack Router Docs](https://tanstack.com/router/latest/docs/guide/data-loading)

---

### 2. TanStack Query併用時にuseLoaderDataを使う

```tsx
// ❌ アンチパターン: useLoaderDataを使うとQuery Observerが作成されない
export const Route = createFileRoute('/posts')({
  loader: ({ context: { queryClient } }) =>
    queryClient.ensureQueryData(postsQueryOptions),
  component: () => {
    const data = Route.useLoaderData() // ウィンドウフォーカス時の再フェッチが機能しない
    return <div>{data.posts.length}</div>
  },
})

// ✅ 推奨: useSuspenseQueryを使う
export const Route = createFileRoute('/posts')({
  loader: ({ context: { queryClient } }) =>
    queryClient.ensureQueryData(postsQueryOptions),
  component: () => {
    const { data } = useSuspenseQuery(postsQueryOptions) // Query Observerが正しく機能する
    return <div>{data.posts.length}</div>
  },
})
```

> 参考: [TanStack Router and Query | tkdodo.eu](https://tkdodo.eu/blog/tan-stack-router-and-query)

---

### 3. グローバル変数によるキャッシュ実装

公式ドキュメントでも「明らかに欠陥のある（obviously flawed）」実装として例示されているアンチパターンです。

```tsx
// ❌ アンチパターン: グローバル変数でのキャッシュは型安全性・整合性に問題がある
let postsCache: Post[] = []

export const Route = createFileRoute('/posts')({
  loader: async () => {
    if (!postsCache.length) {
      postsCache = await fetchPosts()
    }
    return postsCache
  },
})

// ✅ 推奨: staleTime/gcTimeを使ったRouter組み込みキャッシュ、またはTanStack Queryを使う
```

> 参考: [Data Loading | TanStack Router Docs](https://tanstack.com/router/latest/docs/guide/data-loading)

---

### 4. router.invalidate()後にstaleTimeが機能しない問題（既知バグ）

`router.invalidate()`を呼び出した後、`staleTime`が無視され続けLoaderが繰り返し実行されるというバグが報告されています（Issue #2474、Issue #2915）。

`defaultPreload: "intent"`設定時に特に顕著です。調査時点（2026/06/03）での修正状況は未確認のため、最新のリリースノートを参照してください。

**ワークアラウンド:** TanStack Queryを組み合わせ、`queryClient.invalidateQueries()`で細粒度の無効化を行う方法が有効です。

> 参考: [loader staleTime is NOT respected AFTER router invalidation · Issue #2474](https://github.com/TanStack/router/issues/2474)、[After router.invalidate() data is not refetched · Issue #2915](https://github.com/TanStack/router/issues/2915)

---

### 5. コードスプリットコンポーネントでのRoute直接インポート

```tsx
// ❌ アンチパターン: 循環依存が発生する可能性がある
import { Route } from '../routes/posts'

function DeepComponent() {
  const data = Route.useLoaderData()
  return <div>{data.title}</div>
}

// ✅ 推奨: getRouteApiを使用する
import { getRouteApi } from '@tanstack/react-router'

const routeApi = getRouteApi('/posts')

function DeepComponent() {
  const data = routeApi.useLoaderData()
  return <div>{data.title}</div>
}
```

> 参考: [useLoaderData hook | TanStack Router React Docs](https://tanstack.com/router/latest/docs/api/router/useLoaderDataHook)

---

## 参考情報

### 公式ドキュメント

- [Data Loading | TanStack Router Docs](https://tanstack.com/router/latest/docs/guide/data-loading)
- [Data Mutations | TanStack Router Docs](https://tanstack.com/router/v1/docs/guide/data-mutations)
- [External Data Loading | TanStack Router Docs](https://tanstack.com/router/latest/docs/guide/external-data-loading)
- [Router Context | TanStack Router React Docs](https://tanstack.com/router/v1/docs/framework/react/guide/router-context)
- [Search Params | TanStack Router React Docs](https://tanstack.com/router/v1/docs/framework/react/guide/search-params)
- [Router type | TanStack Router React Docs](https://tanstack.com/router/latest/docs/api/router/RouterType)
- [useLoaderData hook | TanStack Router React Docs](https://tanstack.com/router/latest/docs/api/router/useLoaderDataHook)
- [useSearch hook | TanStack Router React Docs](https://tanstack.com/router/latest/docs/framework/react/api/router/useSearchHook)
- [TanStack Query Integration | TanStack Router Docs](https://tanstack.com/router/v1/docs/integrations/query)
- [Prefetching & Router Integration | TanStack Query React Docs](https://tanstack.com/query/v5/docs/framework/react/guides/prefetching)

### 解説記事・ブログ

- [TanStack Router and Query | tkdodo.eu](https://tkdodo.eu/blog/tan-stack-router-and-query)
- [Context Inheritance in TanStack Router | tkdodo.eu](https://tkdodo.eu/blog/context-inheritance-in-tan-stack-router)
- [Loading Data with TanStack Router: Getting Going | Frontend Masters Blog](https://frontendmasters.com/blog/tanstack-router-data-loading-1/)
- [Loading Data with TanStack Router: react-query | Frontend Masters Blog](https://frontendmasters.com/blog/tanstack-router-data-loading-2/)
- [Data Loading and Caching | TanStack Router | DeepWiki](https://deepwiki.com/tanstack/router/2.3-data-loading-and-caching)
- [Data Loading | TanStack Router | DeepWiki](https://deepwiki.com/TanStack/router/3.3-data-loading)

### GitHubイシュー・ディスカッション

- [After router.invalidate() data is not refetched · Issue #2915](https://github.com/TanStack/router/issues/2915)
- [loader staleTime is NOT respected AFTER router invalidation · Issue #2474](https://github.com/TanStack/router/issues/2474)
- [Invalidate all routes · Discussion #2567](https://github.com/TanStack/router/discussions/2567)

---

最終更新日: 2026/06/03
