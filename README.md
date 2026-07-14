# tyweb

A small HTTP router for Tachyon over the built-in concurrent server, with
cookies and signed sessions. Routes are literal segments, `:name` captures, and
a trailing `*` wildcard; handlers are plain function values, so closures capture
your app state. Each worker isolates a panicking request as a 500 and keeps
serving.

## Install

tyweb depends on [`tachyon-crypto`](https://github.com/vahan-gev/tachyon-crypto)
for session signing; Tachyon fetches it transitively.

```toml
# Tachyon.toml
deps = ["git+https://github.com/vahan-gev/tachyon-tyweb#v0.1.0"]
```

```ts
import tyweb.router as web;
import tyweb.session as sess;
```

## Routing

```ts
function main(): void {
    let app = web.Router();
    app.get("/health", (req: web.Request) => web.Response(200, "text/plain", "ok"));
    app.get("/users/:id", (req: web.Request) =>
        web.Response(200, "text/plain", `user ${web.reqParam(&req, "id")}`));
    app.post("/users", (req: web.Request) => web.Response(201, "application/json", req.body));

    web.serve(app, 8080, 8);      // 8 worker threads
}
```

## Sessions

A session is a signed cookie (an HS256 JWT the client cannot forge). Call these
inside a handler — they read and write the live request:

```ts
sess.setSession("secret-key", `{"user":"ada"}`, 3600);   // sign + Set-Cookie
let who = sess.session("secret-key");                    // string? — the payload, or null
sess.endSession();                                       // log out
```

Also `sess.cookie(name)`, `sess.setCookie(name, value, maxAgeSecs)` (HttpOnly,
SameSite=Lax), and `sess.clearCookie(name)`.

## Behind a proxy

The built-in server speaks HTTP; terminate TLS at a reverse proxy (nginx/Caddy)
and set `Secure` cookies there. See the parent Tachyon project for the server's
keep-alive and request-limit behaviour.
