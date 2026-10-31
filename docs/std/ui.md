# std/ui

File-based routing as a generator: `pages(dir)` scans a directory of page modules at generation time and returns a router module.

```vyrn
import { pages } from "std/ui"
import { route } from pages("./pages")
fn handle(req: Request) -> Response { return route(req) }

```

```vyrn
pages/index.vyrn        -> GET /
pages/items/index.vyrn  -> GET /items
pages/items/[id].vyrn   -> GET /items/:id

```

A page module exports what the `Page` contract below allows. A page with `[bracket]` segments exports `type Params`, whose field names must equal the segments; a segment that does not parse as its type is a 404 before user code runs. The router module exports `route(req: Request) -> Response`, `type RoutePath` (the regex of every route), and a typed URL helper per route. A Params mismatch, an unsupported param type or a route collision fails the load with a report at the page module.

## PageError

`type`, at line 34.

```vyrn
type PageError = { status: Int64, message: String }
```

A page-load failure. The router renders the nearest `error.vyx`, or a built-in error body, at `status`.

## pageError

`fn`, at line 36.

```vyrn
fn pageError(status: Int64, message: String) -> PageError
```

## notFound

`fn`, at line 40.

```vyrn
fn notFound(message: String) -> PageError
```

## badRequest

`fn`, at line 44.

```vyrn
fn badRequest(message: String) -> PageError
```

## PageData

`type`, at line 51.

```vyrn
type PageData<T> = enum { Loading, Ready(T) }
```

A lazy page's view data. `Loading` exists only on a client soft nav; the server always renders `Ready`. Defined once here, because each constructor may register only once.

## Meta

`type`, at line 56.

```vyrn
type Meta = { name: String, content: String }
```

## Head

`type`, at line 64.

```vyrn
type Head = {
    title: Option<String>,
    stylesheets: Array<String>,
    modules: Array<String>,
    scripts: Array<String>,
    meta: Array<Meta>,
}
```

What a page contributes to the document head, built from `noHead()` and the `with*` combinators:

```vyrn
export fn head() -> Head {
    return withModule(noHead(), "/app.js")
}

```

## noHead

`fn`, at line 72.

```vyrn
fn noHead() -> Head
```

## withTitle

`fn`, at line 77.

```vyrn
fn withTitle(h: Head, title: String) -> Head
```

The page's title wins over its layouts'.

## withStylesheet

`fn`, at line 81.

```vyrn
fn withStylesheet(h: Head, href: String) -> Head
```

## withModule

`fn`, at line 87.

```vyrn
fn withModule(h: Head, src: String) -> Head
```

## withScript

`fn`, at line 93.

```vyrn
fn withScript(h: Head, src: String) -> Head
```

## withMeta

`fn`, at line 99.

```vyrn
fn withMeta(h: Head, name: String, content: String) -> Head
```

## headHtml

`fn`, at line 106.

```vyrn
fn headHtml(h: Head) -> Array<Html>
```

Stylesheets, then modules, then scripts, then meta, each in the order added.

## headTitleOf

`fn`, at line 124.

```vyrn
fn headTitleOf(h: Head) -> String
```

`""` when the head declares no title.

## Query

`type`, at line 137.

```vyrn
type Query<T> = { run: fn() -> T }
```

A page's data, produced by a deferred call. The page blocks until the data lands; return a [`Lazy`](#e-Lazy) to render first and fill in later.

```vyrn
export fn data() -> Query<Array<Paste>> {
    return query(|| listPastes().pastes)
}

```

## Lazy

`type`, at line 143.

```vyrn
type Lazy<T> = { run: fn() -> T }
```

A page's data when the page renders lazily: on a client soft nav the shell paints first and the data region fills in. A distinct type, not a flag, because laziness decides the view's type (`PageData<T>`), and the generator reads types, not function bodies.

## ParamQuery

`type`, at line 151.

```vyrn
type ParamQuery<P, T> = { run: fn(P) -> T }
```

A page's data when it depends on the route parameters. `P` is the page's own `Params`, which `std/ui` cannot name.

```vyrn
export fn data() -> ParamQuery<Params, Result<Paste, PageError>> {
    return paramQuery(|p: Params| fetch(p.id))
}

```

## ParamLazy

`type`, at line 153.

```vyrn
type ParamLazy<P, T> = { run: fn(P) -> T }
```

## query

`fn`, at line 155.

```vyrn
fn query<T>(run: consume fn() -> T) -> Query<T>
```

## lazy

`fn`, at line 160.

```vyrn
fn lazy<T>(q: consume Query<T>) -> Lazy<T>
```

The server always has the data, so SSR is unaffected.

## paramQuery

`fn`, at line 164.

```vyrn
fn paramQuery<P, T>(run: consume fn(P) -> T) -> ParamQuery<P, T>
```

## paramLazy

`fn`, at line 168.

```vyrn
fn paramLazy<P, T>(q: consume ParamQuery<P, T>) -> ParamLazy<P, T>
```

## noQuery

`fn`, at line 194.

```vyrn
fn noQuery() -> Query<Unit>
```

The default `data` for a page that declares none.

## pages

`fn`, at line 2178.

```vyrn
gen fn pages(dir: String) -> Code
```

Scans `dir` for `.vyrn` and `.vyx` pages and returns the router module.

## pagesThemed

`fn`, at line 2184.

```vyrn
gen fn pagesThemed(dir: String, theme: String) -> Code
```

[`pages`](#e-pages), with every `.vyx` page's classes checked against `theme`. `theme` resolves relative to the importing module.

## pagesClient

`fn`, at line 2453.

```vyrn
gen fn pagesClient(dir: String) -> Code
```

The client page bundle for `dir`.

## pagesClientThemed

`fn`, at line 2458.

```vyrn
gen fn pagesClientThemed(dir: String, theme: String) -> Code
```

[`pagesClient`](#e-pagesClient) with a theme; the server side is [`pagesThemed`](#e-pagesThemed).

---

From https://vyrn-lang.github.io/docs/std/ui.html — generated by `site/export.vyrn` from the Vyrn repository.
