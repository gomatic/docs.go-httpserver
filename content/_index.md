---
title: go-httpserver
---

`go-httpserver` is the gomatic ecosystem's **HTTP server lifecycle wrapper** for Go. It wraps an [`*http.Server`](https://pkg.go.dev/net/http#Server) and owns the start/stop machinery — listen, context-cancellation shutdown, and the start/shutdown error contract — and nothing else. The caller supplies the [`http.Handler`](https://pkg.go.dev/net/http#Handler) and wires the cancellation (typically via [`signal.NotifyContext`](https://pkg.go.dev/os/signal#NotifyContext)); a single [`Serve`](https://pkg.go.dev/github.com/gomatic/go-httpserver#Server.Serve) call then blocks until the context is cancelled or startup fails, shutting the server down gracefully within a supplied timeout. It is stdlib-only and reusable by any service that needs to serve HTTP and stop cleanly.

- **Source:** [`gomatic/go-httpserver`](https://github.com/gomatic/go-httpserver)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-httpserver](https://pkg.go.dev/github.com/gomatic/go-httpserver)

## Install

```sh
go get github.com/gomatic/go-httpserver
```

## Why a lifecycle wrapper

Serving HTTP cleanly is more than `ListenAndServe`: a real service must shut down gracefully on a signal, bound that shutdown by a deadline, and report a startup failure without letting a clean shutdown mask it. Hand-rolling that dance — a goroutine, an error channel, a `select` on context cancellation, and the right error precedence — is repetitive and easy to get subtly wrong. `go-httpserver` owns that machinery once, so every gomatic service starts and stops identically.

The library owns the **lifecycle only** — it ships no routing, middleware, or handler logic. The caller supplies the handler and the cancellation context:

```go
import "github.com/gomatic/go-httpserver"

// Bind host:port, serve a handler, and stop when ctx is cancelled.
srv := httpserver.New(slog.Default(), "127.0.0.1", 8080, handler)
err := srv.Serve(ctx, 5*time.Second)
```

## Usage

### Serve until a signal

The caller wires cancellation — typically with [`signal.NotifyContext`](https://pkg.go.dev/os/signal#NotifyContext) — and passes the resulting context to [`Serve`](https://pkg.go.dev/github.com/gomatic/go-httpserver#Server.Serve). When the context is cancelled (e.g. on SIGINT/SIGTERM), the server shuts down gracefully within the supplied timeout.

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gomatic/go-httpserver"
)

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	handler := http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		w.WriteHeader(http.StatusOK)
	})

	srv := httpserver.New(slog.Default(), "127.0.0.1", 8080, handler)
	if err := srv.Serve(ctx, 5*time.Second); err != nil {
		slog.Error("server failed", "error", err)
		os.Exit(1)
	}
}
```

[`Serve`](https://pkg.go.dev/github.com/gomatic/go-httpserver#Server.Serve) blocks until the context is cancelled or startup fails. Request contexts derive from the lifecycle context (via [`http.Server.BaseContext`](https://pkg.go.dev/net/http#Server)), so in-flight handlers observe the same cancellation.

### Matching the lifecycle errors

The package declares two sentinels and wraps the underlying cause with [`go-error`](https://github.com/gomatic/go-error), so callers match the _identity_ of the failure with [`errors.Is`](https://pkg.go.dev/errors#Is) — never by string:

```go
import (
	"errors"

	"github.com/gomatic/go-httpserver"
)

err := srv.Serve(ctx, 5*time.Second)
switch {
case errors.Is(err, httpserver.ErrServerStart):
	// the server could not start listening (e.g. port in use)
case errors.Is(err, httpserver.ErrServerShutdown):
	// the server did not shut down within the deadline
}
```

Because the cause is joined with `%w`, the wrapped error (e.g. the original `net` bind error) also remains recoverable with [`errors.Is`](https://pkg.go.dev/errors#Is).

### Inspecting the configured server

[`New`](https://pkg.go.dev/github.com/gomatic/go-httpserver#New) returns a [`*Server`](https://pkg.go.dev/github.com/gomatic/go-httpserver#Server) exposing two read accessors:

```go
srv := httpserver.New(slog.Default(), "127.0.0.1", 8080, handler)
srv.Addr()    // "127.0.0.1:8080" — the configured listen address
srv.Handler() // the http.Handler the server serves
```

## Design

- **Lifecycle only.** [`Server`](https://pkg.go.dev/github.com/gomatic/go-httpserver#Server) wraps an [`*http.Server`](https://pkg.go.dev/net/http#Server) and owns start, context-cancellation shutdown, and the error contract — nothing else. Routing and middleware belong to the caller's handler.
- **Startup failure is never masked.** When the context is cancelled, a pending startup error is preferred over a clean shutdown result, so a real failure (e.g. a port already in use) always surfaces.
- **Shared error mechanism.** [`ErrServerStart`](https://pkg.go.dev/github.com/gomatic/go-httpserver#pkg-constants) and [`ErrServerShutdown`](https://pkg.go.dev/github.com/gomatic/go-httpserver#pkg-constants) are declared on the [`go-error`](https://github.com/gomatic/go-error) constant/sentinel mechanism, matchable with [`errors.Is`](https://pkg.go.dev/errors#Is) and wrapping their cause with `%w`.
- **Named domain types.** [`Host`](https://pkg.go.dev/github.com/gomatic/go-httpserver#Host) and [`Port`](https://pkg.go.dev/github.com/gomatic/go-httpserver#Port) name the bind address parameters instead of bare `string`/`int`.
- **Slowloris-resistant.** A `ReadHeaderTimeout` bounds how long the server waits for request headers, preventing connections that hold open by trickling headers (gosec G112).
- **Dependency-light.** The package depends only on the standard library plus [`gomatic/go-error`](https://github.com/gomatic/go-error) for its sentinels.

## Who uses it

Every gomatic Go service that serves HTTP embeds this wrapper for its start/stop lifecycle, alongside the other [`gomatic/go-*`](https://github.com/orgs/gomatic/repositories?q=go-) libraries.
