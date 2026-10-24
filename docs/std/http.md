# std/http

The REST projection: hand-written routes over the same procedures the derived RPC surface reflects on. A projection is opt-in, because a path like `GET /pastes/{id}` is API design that reflection cannot derive:

```vyrn
// server/api/pastes.http.vyrn -- base path `/pastes`, from the stem.
import { http, Route, GET, POST } from "std/http"
import { recent, byId, create } from http("./pastes")

export fn routes() -> Array<Route> {
    return [GET(recent("/")), GET(byId("/{id}")), POST(create("/"))]
}

```

The chain is value-level: every call takes a `Route` and returns a `Route`, so the type never widens. The pattern sits in the procedure's own parameter slot, whose generated type admits only the placeholders the input record has fields for, so `byId("/{ID}")` is a checker error at the call site. A generic `GET(pattern, proc)` could not do that: a type parameter cannot name a type to `fromJson`.

## Handler

`type`, at line 46.

```vyrn
type Handler = fn(Request, Map<String, String>) -> Option<Response>
```

What a mounted route runs: the request and the placeholder bindings. `None` declines, and `mount` tries the next group.

## Surface

`type`, at line 50.

```vyrn
type Surface = fn(Request) -> Option<Response>
```

A whole mounted subsystem, such as `rpcHandle` or a page router. A prefix binds no placeholders, so it takes only the request.

## IsMissing

`type`, at line 54.

```vyrn
type IsMissing = fn(String) -> Bool
```

Whether an `Err` payload is the absence this resource is named after. A `String`, because `Route` erases the procedure's error type.

## Route

`type`, at line 64.

```vyrn
type Route = {
    method: String,
    pattern: String,
    prefix: Bool,
    run: Handler,
    whole: Surface,
    derived: String,
    // `Cache-Control: max-age=N`; `-1` writes no header (`cacheFor`).
    maxAge: Int64,
    // Whether to compute an `ETag` and answer conditionals with it (`etag`).
    validator: Bool,
    // The response field carrying epoch millis for `Last-Modified`; `""` none.
    modified: String,
    // The `Vary` header this route forces; `""` keeps what the handler said.
    varyOn: String,
    // A success status that replaces the handler's; `0` keeps it (`status`).
    ok: Int64,
    // A `Location` template like `/pastes/{id}`; `""` none (`createdAt`).
    location: String,
    // Which `Err` payloads mean 404 (`notFoundWhen`).
    missing: IsMissing,
}
```

One route: a method, a full path pattern (base + sub-path), what runs, and the `derived` policy line the diagnostics quote. `prefix` marks a subsystem mounted under `pattern` and selects whether `mount` calls `run` or `whole`. `method` is `""` until a method constructor sets it; `mount` traps on that.

Two fn fields, not one plus an adapting closure: an adapter would allocate a capture block per subsystem and add a dispatch per request. The policy fields are flat rather than a nested record, so no `fn` value sits one record deeper.

## httpRoute

`fn`, at line 102.

```vyrn
fn httpRoute(pattern: String, run: Handler, derived: String) -> Route
```

A route with no method yet; `pattern` is already base-qualified. Generated constructors call it.

## GET

`fn`, at line 153.

```vyrn
fn GET(r: Route) -> Route
```

Sets the route's method; `POST`, `PUT`, `PATCH` and `DELETE` are the same with their verb. Uppercase, because the five verbs are HTTP's own spelling.

## POST

`fn`, at line 157.

```vyrn
fn POST(r: Route) -> Route
```

## PUT

`fn`, at line 161.

```vyrn
fn PUT(r: Route) -> Route
```

## PATCH

`fn`, at line 165.

```vyrn
fn PATCH(r: Route) -> Route
```

## DELETE

`fn`, at line 169.

```vyrn
fn DELETE(r: Route) -> Route
```

## surface

`fn`, at line 183.

```vyrn
fn surface(prefix: String, run: consume Surface) -> Route
```

A whole subsystem mounted under `prefix`, such as the RPC surface or a page router, in the `handle` shape.

## Policy

`protocol`, at line 207.

```vyrn
protocol Policy {
    /// `Cache-Control: max-age=N` on a successful answer. Neither `public`, which
    /// would let a shared cache store a credentialed response (RFC 9111 section
    /// 3.5), nor `private`, which would stop a CDN caching the anonymous GET.
    fn cacheFor(self, seconds: Int64) -> Route

    /// A strong `ETag`, and `If-None-Match` answered with 304. The tag is
    /// `FNV-1a-64(contentType + "\n" + body)` in hex: the representation, so two
    /// `Vary` variants differ, and content only, so every process emits the same
    /// tag. A 64-bit collision serves a 304 for content the client lacks.
    fn etag(self) -> Route

    /// `Last-Modified` from the named epoch-millis field, and `If-Modified-Since`
    /// answered with 304. A missing field writes no header rather than trapping.
    fn lastModified(self, field: String) -> Route

    /// The `Vary` header, written to `Response.vary`.
    fn vary(self, headers: String) -> Route

    /// The status a successful answer carries, such as 202. An error keeps its
    /// own status.
    fn status(self, code: Int64) -> Route

    /// `201 Created` with a `Location` built from the response: `"/pastes/{id}"`
    /// takes `{id}` from the created object's `id` field. A template, because a
    /// closure over the output type would make `Route` generic. An unknown
    /// `{name}` stays verbatim in the URL.
    fn createdAt(self, template: String) -> Route

    /// Which `Err` payloads are an absence: `.notFoundWhen(|why| why == "no such
    /// paste")` makes that error a 404, and every other `Err` stays a 200.
    fn notFoundWhen(self, isMissing: IsMissing) -> Route
}
```

The route policy. Every method is `Route -> Route`, so the chain stays one nominal value to the checker:

```vyrn
GET(byId("/{id}")).cacheFor(3600).etag().notFoundWhen(|why| why == "no such user")

```

## Feed

`type`, at line 296.

```vyrn
type Feed = fn(Request, Map<String, String>, Int64) -> Stream<String>
```

What a mounted stream runs: the request, the bindings, and the cursor, which is the client's `Last-Event-ID` when the route is `resumable` and `0` otherwise. It yields encoded frames (see `event`): `std/stream`'s `map` is eager, so mapping records to frames would buffer the whole feed.

## Live

`type`, at line 301.

```vyrn
type Live = {
    pattern: String,
    feed: Feed,
    // `retry: N` in milliseconds, written before the first frame; `0` writes
    // none (`retryAfter`).
    retry: Int64,
    // Whether `Last-Event-ID` seeds the producer (`resumable`).
    resume: Bool,
    derived: String,
}
```

A mounted event stream. Not a `Route`: every `Policy` option describes a response that exists all at once, and an option meaningless to a transport is absent from it.

## sse

`fn`, at line 315.

```vyrn
fn sse(pattern: String, feed: consume Feed) -> Live
```

Mounts `feed` as an event stream at `pattern`. The pattern is an argument: a feed takes the request, not a record, so there are no fields to check the placeholders against.

## Wire

`protocol`, at line 327.

```vyrn
protocol Wire {
    /// `retry: N` before the first frame. A producer cannot say "nothing yet",
    /// so a feed that catches up ends and the client returns `ms` later: this is
    /// the poll interval.
    fn retryAfter(self, ms: Int64) -> Live

    /// `Last-Event-ID` becomes the producer's seed, so the feed resumes where the
    /// client left off. Write the cursor as each frame's `id`. Without
    /// it the seed is `0`.
    fn resumable(self) -> Live
}
```

A stream's policy. No `keepAlive`: a pull producer with nothing to say ends rather than idling.

## event

`fn`, at line 366.

```vyrn
fn event(id: String, name: String, data: String) -> String
```

One SSE frame: `id:`, `event:`, the `data:` lines and the closing blank line. An empty `id` or `name` writes no field. CR, LF and CRLF all end a line, so no argument may end one: `data` splits at every break into further `data:` lines, and `id` and `name` lose their CR and LF bytes, so an untrusted id cannot inject an event. A NUL cannot reach here; no `String` holds one.

## Socket

`type`, at line 395.

```vyrn
type Socket = {
    pattern: String,
    feed: Feed,
    // The close code the host sends when the feed ends (`closeCode`); 1000 is
    // normal closure.
    closing: Int64,
    // Echoed as `Sec-WebSocket-Protocol` only when the client offered it
    // (`subprotocol`); `""` negotiates none.
    subproto: String,
    // The largest payload per frame; longer ones fragment (`maxFrame`). `0`
    // never fragments.
    fragment: Int64,
    derived: String,
}
```

A mounted WebSocket; not a `Route`. Server-push only: the host parses inbound frames to honour a close and the masking rule, and delivers no message. No heartbeat: between frames the host is blocked in the producer, and a failed write already detects a vanished peer.

## ws

`fn`, at line 413.

```vyrn
fn ws(pattern: String, feed: consume Feed) -> Socket
```

Mounts `feed` as a WebSocket at `pattern`. The element is the message payload, not a frame: Vyrn owns what the user chooses (SSE's fields), and the host owns what the protocol fixes (opcode, length, mask).

## Frames

`protocol`, at line 425.

```vyrn
protocol Frames {
    fn closeCode(self, code: Int64) -> Socket
    fn subprotocol(self, name: String) -> Socket
    fn maxFrame(self, size: Int64) -> Socket
}
```

A socket's policy.

## mount

`fn`, at line 586.

```vyrn
fn mount(req: Request, groups: Array<Array<Route>>, live: Array<Live>, sockets: Array<Socket>) -> Option<Response>
```

Resolves `req` against the mounted groups, in order; the first match wins. A route that an earlier group, stream or socket already answers traps. The audit runs once per process, so `vyrn serve` reports it at startup; an app that changes its groups between requests is outside it. Streams and sockets are separate lists, because Vyrn has no sum over two record types.

## httpInput

`fn`, at line 1090.

```vyrn
fn httpInput(ps: Map<String, String>, body: String, numeric: Array<String>) -> String
```

The JSON a generated adapter hands `fromJson`: the bindings, then the body's fields. A bound name wins over a body field; a duplicate key would be a 422. Fields named in `numeric` bind as integers, so `/users/{id}` over an `Int64` gives `{"id":7}`; a value that does not parse stays a string, for the decoder to report. A `Float64` path field stays text: `parse` reads integers only.

## http

`fn`, at line 1280.

```vyrn
gen fn http(module: String) -> String
```

The REST projection of one procedure module: a placeholder-checked constructor per procedure, under the base path its stem derives.

## mountedRows

`fn`, at line 1332.

```vyrn
fn mountedRows(groups: Array<Array<Route>>, live: Array<Live>, sockets: Array<Socket>) -> String
```

Every `derived` row a `mount(..)` argument list carries, one per line, for `vyrn routes`. The first word is the kind: `SSE`, `WS`, `*` for a subsystem, or the HTTP method of a route, whose third word is its procedure.

---

From https://vyrn-lang.github.io/docs/std/http.html — generated by `site/export.vyrn` from the Vyrn repository.
