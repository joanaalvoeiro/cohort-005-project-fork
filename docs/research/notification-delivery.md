# Getting new notifications to the browser (React Router 8 + SQLite)

Research for [#4](https://github.com/joanaalvoeiro/cohort-005-project-fork/issues/4), part of the in-app notifications map ([#1](https://github.com/joanaalvoeiro/cohort-005-project-fork/issues/1)). This doc compares the options and does not pick one. A later grilling ticket decides.

## Stack facts that constrain the answer

- React Router 8.3 in framework mode with `ssr: true` (`react-router.config.ts`). Production runs `react-router-serve ./build/server/index.js` (`package.json`).
- One SQLite file, `data.db`, opened lazily with better-sqlite3 in WAL mode (`app/db/index.ts`). better-sqlite3 is synchronous, so every query runs on the Node event loop.
- One Node process. Nothing pushes to the browser today. The only `api.*` resource routes are action-only JSON endpoints (`app/routes/api.video-tracking.ts`).
- `@react-router/serve` is Express with `compression()` at its defaults, plus `morgan` and `express.static`. It mounts `createRequestHandler` from `@react-router/express` ([cli.ts](https://github.com/remix-run/react-router/blob/main/packages/react-router-serve/cli.ts)).

## Option 1: Revalidate loaders on navigation (no extra mechanism)

**How it works.** A loader in the app layout (`routes/layout.app.tsx`) or in `root` returns the unread count or the latest notifications. React Router's docs say: "In framework mode with SSR, route loaders are automatically revalidated after all navigations and form submissions." Actions bring "automatic revalidation of all loader data on the page when called from `<Form>`, `useFetcher`, and `useSubmit`" ([Route Module](https://reactrouter.com/start/framework/route-module)). A route can opt out with `shouldRevalidate`, but "it does not skip parent or sibling loaders" (same page).

- **Latency.** Unbounded. A notification shows up only on the user's next navigation or mutation. A user who reads one lesson page for 20 minutes sees nothing new.
- **Server load.** Close to zero extra. The query runs inside requests that happen anyway, though the layout loader now runs on every navigation. An indexed `COUNT(*) WHERE user_id = ? AND read_at IS NULL` on better-sqlite3 is a sub-millisecond synchronous call.
- **Implementation.** Smallest. One query in an existing loader and one badge component. No new routes and no client timers.
- **Testing.** Easiest. The query is a plain function you can test against `createTestDb()` (`app/test/setup.ts`), like the existing `app/lib/*.test.ts`.
- **Single SQLite file and `@react-router/serve`.** Works with no caveats. Nothing is long-lived.
- **Multiple tabs.** Each tab updates only when that tab navigates. Marking a notification read in tab A leaves tab B's badge stale until tab B navigates.

## Option 2: Poll on an interval (`useRevalidator` or a fetcher)

**How it works.** Option 1 plus a client timer.

- *`useRevalidator`.* The docs describe it as a way to "revalidate the data on the page for reasons outside of normal data mutations like `Window` focus or polling on an interval". It returns `revalidate(): Promise<void>` and a `state` ([useRevalidator](https://reactrouter.com/api/hooks/useRevalidator)). Calling it re-runs **every** active loader on the page, not just the notification one, unless the other routes opt out with `shouldRevalidate`.
- *Fetcher against a resource route.* `fetcher.load("/api/notifications")` loads data with its "own, independent state" ([useFetcher](https://reactrouter.com/api/hooks/useFetcher)). A resource route is a module that "exports a loader or action but does not export a default component" ([Resource Routes](https://reactrouter.com/how-to/resource-routes)). Polling it runs only that one loader. That makes it much cheaper than `revalidate()` on heavy pages like the instructor analytics page. The fetcher docs reviewed here do not say whether `fetcher.load` triggers page revalidation; they describe revalidation only for submissions ([Using Fetchers](https://reactrouter.com/how-to/fetchers)).
- *Visibility.* Pause the timer when `document.visibilityState === "hidden"` and refresh once on `visibilitychange` ([Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)). Browsers already throttle background-tab timers. MDN says background windows are "subjected to throttling after 30 seconds… In Chrome, this value is 10 seconds", under a budget that "regenerates at a rate of 10 ms per second" (same page). So a hidden tab's poll rate is unreliable whether or not you pause it.

- **Latency.** Up to one interval (for example 15–60 s), plus a refresh on tab focus.
- **Server load.** One HTTP request per visible tab per interval. Each request runs the session lookup and an indexed query. At 30 s and 100 concurrent visible tabs that is about 3.3 req/s. That load is trivial for one Node process with better-sqlite3, and WAL mode lets these reads run alongside writes. With `useRevalidator`, multiply by the number of loaders on the page.
- **Implementation.** Small. One resource route (or none, with `useRevalidator`), plus a hook with `setInterval` and a visibility listener (about 30 lines).
- **Testing.** The loader is a pure server function and easy to test. The hook needs fake timers (`vi.useFakeTimers()`) and a stubbed `document.visibilityState`. That is moderate work, and the repo has no React component tests yet.
- **Single SQLite file and `@react-router/serve`.** Works with no caveats. Requests are ordinary and short-lived.
- **Multiple tabs.** Each tab polls on its own, so N tabs cost N times the load and drift apart by up to one interval. `BroadcastChannel` fixes this: it lets "different documents (in different windows, tabs, frames or iframes) of the same origin" exchange messages and is Baseline widely available since March 2022 ([BroadcastChannel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel)). A tab that marks a notification read can post `"notifications-changed"`, and the other tabs refetch right away. Electing one polling tab is possible but adds complexity.

## Option 3: Server-sent events (SSE) through a resource route

**How it works.** A resource-route loader (for example `api/notifications/stream`) returns `new Response(readableStream, { headers: { "Content-Type": "text/event-stream" } })`. The browser opens `new EventSource("/api/notifications/stream")`. Same-origin cookies are sent, so `getCurrentUserId(request)` works as it does in other routes.

Wire format and browser behaviour, from MDN ([Using SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)) and the WHATWG spec ([§9.2](https://html.spec.whatwg.org/multipage/server-sent-events.html)):

- The response must be `text/event-stream`, UTF-8, with messages separated by a blank line. Fields are `event:`, `data:`, `id:` and `retry:` (milliseconds).
- "By default, if the connection between the client and server closes, the connection is restarted." On reconnect the browser sends a `Last-Event-ID` header if an `id:` was seen. The server can use it to replay what was missed, for example `WHERE id > ?`.
- A non-200 status or the wrong content type makes the browser "fail the connection", and it "does not attempt to reconnect". An unauthenticated stream should therefore return 401 or 204 on purpose, not crash.
- Lines starting with `:` are comments. The spec suggests sending one "every 15 seconds or so" to keep proxies from dropping idle connections.
- **HTTP/1.1 connection cap.** "When not used over HTTP/2… the limit is *per browser* and is set to a very low number (6)… per browser + domain" (MDN). `react-router-serve` speaks plain HTTP/1.1. Without an HTTP/2 reverse proxy in front, each open tab holds one of those 6 slots. With 6 tabs open, the app's own fetches and navigations stall.

How the stack handles a streamed body:

- `@react-router/express` calls `res.flushHeaders()` when the response Content-Type matches `text/event-stream`. Its `writeReadableStreamToWritable` calls `writable.flush()` after every chunk when that method exists ([server.ts](https://github.com/remix-run/react-router/blob/main/packages/react-router-express/server.ts), [stream.ts](https://github.com/remix-run/react-router/blob/main/packages/react-router-node/stream.ts), checked on `main`, not on the 8.3.0 tag).
- The `compression` middleware's README says it "does not work out of the box with server-sent events" unless `res.flush()` is called ([expressjs/compression](https://github.com/expressjs/compression)). `text/event-stream` has no entry in mime-db, so `compressible` falls back to `/^text\//` and treats it as compressible ([compressible](https://github.com/jshttp/compressible/blob/master/index.js)). The adapter's per-chunk `flush()` covers this, so SSE should work behind `react-router-serve`. Confirm it with a quick spike before relying on it.
- The adapter aborts `request.signal` on `res` `close` when no response has been sent (`server.ts`). Once the stream has started, the loader has to clean up in the `ReadableStream`'s `cancel()` callback, or catch the write failure, so it can clear intervals and unsubscribe. Otherwise every closed tab leaks a timer.

Where "new notification" events come from:

- *In-process emitter.* Code that inserts a notification also calls `notificationBus.emit(userId)`, a Node `EventEmitter`. Each open stream subscribes for its user. This gives zero polling and near-instant delivery. It works **only because there is one Node process**. A second instance, or a write from `scripts/seed.ts` or another process, would never reach the emitter.
- *Server-side DB poll.* Each stream runs `SELECT … WHERE user_id = ? AND id > lastId` every few seconds. This works whichever process wrote the row. Each query is cheap, but better-sqlite3 runs them synchronously on the event loop, once per open connection.

- **Latency.** Sub-second with the emitter, or one server-poll interval. Reconnects after a deploy or restart take the browser's retry delay ("a few seconds" per the spec, or whatever `retry:` sets).
- **Server load.** One open socket per tab (memory, no CPU while idle) plus a heartbeat every ~15 s. CPU stays low with the emitter. With a server-side poll, load scales with connections.
- **Implementation.** The most work. A stream route, framing, heartbeat, cleanup on disconnect, auth failure handling, `Last-Event-ID` replay, and an emitter or poll loop. On the client, an `EventSource` hook that merges events into loader data or calls `revalidate()`.
- **Testing.** The hardest. Unit tests can call the loader and read the `Response` body stream, but cleanup and timing need careful fake-timer work. End-to-end behaviour (reconnects, compression, proxy buffering) needs a running server.
- **Single SQLite file and `@react-router/serve`.** Works, with the caveats above: the 6-connection HTTP/1.1 cap, a compression flush the adapter already does, and an emitter that is valid only for one process. Any reverse proxy in production must not buffer the response.
- **Multiple tabs.** Each tab opens its own `EventSource`, and every connection counts toward the cap of 6. Sharing one connection across tabs needs a SharedWorker, or a leader tab that relays over `BroadcastChannel`. Either adds a lot of complexity.

## Comparison

| | Navigation revalidation | Interval polling (fetcher / `useRevalidator`) | SSE resource route |
| --- | --- | --- | --- |
| Latency | Until next navigation or mutation (unbounded) | ≤ poll interval, plus refresh on focus | Sub-second (emitter) or server-poll interval |
| Server load | ~0 extra | 1 short request per visible tab per interval | 1 open socket per tab, plus heartbeats |
| Implementation | Trivial | Small | Large |
| Testing | Easy (pure query) | Moderate (fake timers, visibility) | Hard (streams, cleanup, real server) |
| Single SQLite file | Yes | Yes | Yes. An in-process emitter is single-process only |
| `@react-router/serve` | Yes | Yes | Works (adapter flushes headers and chunks). Verify with a spike. HTTP/1.1 caps it at 6 connections |
| Multiple tabs | Each tab stale until it navigates | N tabs = N times the polls; `BroadcastChannel` syncs read state | N connections against the cap of 6; sharing one needs SharedWorker or a leader tab |

The options stack. Navigation revalidation is the base layer either way. Polling or SSE only shortens the gap between navigations, and `BroadcastChannel` works with any of them for cross-tab read state. For a small single-server app on `react-router-serve`, plain HTTP/1.1 and one SQLite file, options 1 and 2 add no new failure modes. Option 3 buys lower latency at the cost of long-lived connections and per-process state.
