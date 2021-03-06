---
title: actix-web
sidebar_order: 2
---

*Crate: `sentry-actix` (has to be installed separately)*

The `sentry-actix` *crate* adds a middleware for
[`actix-web`](https://actix.rs/) that captures errors and report them to
`Sentry`.

To use this middleware just configure Sentry and then add it to your actix web
app as a middleware.  Because actix is generally working with non sendable
objects and highly concurrent this middleware creates a new hub per request.
As a result many of the sentry integrations such as breadcrumbs do not work
unless you bind the actix hub.

## Example

In your `Cargo.toml`:

```toml
[dependencies]
sentry = "{% sdk_version sentry.rust %}"
sentry-actix = "{% package_version cargo:sentry-actix %}"
```

And your Rust code:

```rust
extern crate actix_web;
extern crate sentry;
extern crate sentry_actix;

use std::env;
use std::io;

use actix_web::{server, App, Error, HttpRequest};
use sentry_actix::SentryMiddleware;

fn failing(_req: &HttpRequest) -> Result<String, Error> {
    Err(io::Error::new(io::ErrorKind::Other, "An error happens here").into())
}

fn main() {
    let _guard = sentry::init("___PUBLIC_DSN___");
    env::set_var("RUST_BACKTRACE", "1");
    sentry::integrations::panic::register_panic_handler();

    server::new(|| {
        App::new()
            .middleware(SentryMiddleware::new())
            .resource("/", |r| r.f(failing))
    }).bind("127.0.0.1:3001")
        .unwrap()
        .run();
}
```

## Reusing the Hub

If you use this integration the `Hub::current()` returned hub is typically the wrong one.
To get the request specific one you need to use the `ActixWebHubExt` trait:

```rust
use sentry::{Hub, Level};
use sentry_actix::ActixWebHubExt;

let hub = Hub::from_request(req);
hub.capture_message("Something is not well", Level::Warning);
```

The hub can also be made current:

```rust
use sentry::{Hub, Level};
use sentry_actix::ActixWebHubExt;

let hub = Hub::from_request(req);
Hub::run(hub, || {
    sentry::capture_message("Something is not well", Level::Warning);
});
```
