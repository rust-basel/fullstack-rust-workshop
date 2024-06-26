# Axum Hello World

Let's start building our HTTP backend!

## Dependencies

Go into your `backend` crate and install the following dependencies:

```sh
cargo add axum
cargo add tokio -F full
cargo add serde -F derive
cargo add tower-http -F cors
```

[Axum](https://github.com/tokio-rs/axum) will be our webservice framework of choice. As it's very extensible with [tower-http](https://github.com/tower-rs/tower-http), where you can easily create middlewares.

[Serde](https://serde.rs/) is Rusts de facto standard serialization library, so we can easily receive and send `json` payloads and map them to Rust models.

[Tokio](https://tokio.rs/) is the most popular async runtime (and required by axum, which runs within tokio)

## Add a small webservice

Head to your `main.rs` and add the following minimal executable:

```rust
use axum::{extract::Path, response::IntoResponse, routing::get, Router};

async fn hello_world() -> impl IntoResponse {
    "Hello World"
}

async fn hello_name(Path(name): Path<String>) -> impl IntoResponse {
    format!("Hello {name}")
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(hello_world))
        .route("/:name", get(hello_name));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3001").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

The `Router` lets you define the endpoints of your webservice. You can attach different routes to it, as well as other middlewares.
Middlewares are logical units, that are shared across different routes. One example would be an `Auth` middleware.

The most basic function of axum with a router is **mapping requests to handlers**. A handler is anything that
returns a value that implements `IntoResponse`, and the most trivial example of this is a String, as in the line `format!("Hello {name}")`.

After adding those few lines run.

```sh
cargo run
```

And wait for your compiler to build and run the app. Your webservice is now running on `localhost:3001`. So if you go to your browser, your should see a `Hello World`.
If you put `localhost:3001/your-name` into your browser, then the path parameter `your-name` is extracted by axum, as you see in the `Path(name): Path<String>` argument.

## Responding with and receiving json payload

If you want to create a resource server side, you usually send a http `POST` to your server with a Json payload.
Serialization is hard? Not with Rust - as we have the `Serde` crate.

Add a rust struct annotated with serde's Serialize and Deserialize create.

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct Workshop {
    attendees_count: i32,
    people_like_it: bool,
}
```

Now this struct `Workshop` is automatically de- and serializable. The [Procedural Macro](https://doc.rust-lang.org/reference/procedural-macros.html) implements this for us!

If we now want to receive and respond this with Json, we have to add a fitting handler (function) to it.

Add this to your `main.rs`:

```rust
async fn workshop_echo(Json(workshop): Json<Workshop>) -> impl IntoResponse {
    Json(workshop)
}
```

This will just echo the json. But you get the idea. We also need to add this to our `Router`.
Add it as a `post` (you could also use `get`, but idiomatically json payloads are send with either `patch`, `put` or `post`).

```rust
use axum::routing::post;
```

```rust
let app = Router::new()
    .route("/", get(hello_world))
    .route("/:name", get(hello_name))
    .route("/your-route", post(workshop_echo));
```

You can test your json endpoint with the following curl call, to check whether you added it correctly ;).

```sh
curl -i \
    -H "Accept: application/json" \
    -H "Content-Type: application/json" \
    -X POST -d '{"attendees_count": 20, "people_like_it": true}' \
    http://localhost:3001/your-route
```

Whooray! You got the basic building block ready. The next steps will take care fo letting our frontend backend talk to each other. Let's 
display a list of items from our backend in the frontend in the next chapter.

# Axum alternatives
There are many other Rust server frameworks.
Axum is still rather new, and seemingly the 'bright star' in this universe. It is extremely fast, because of its simplicity and direct coupling to tokio.
Documentation and adaptation in production are still lagging somewhat, but with the current popularity, these things are bound to catch up too.
The most popular framework probably still is `actix-web`, which is also rather lightweight and revolves around the actor pattern.
The most full-fledged framework would be`rocket`, with very robust features and tools. While axum is more like expressjs, rocket is more comparable to Ruby on Rails.

Speaking about rails, there is also `loco`, a framework layer built on top of axum, with the goal of being super small and simple, perfect for one-(wo)man teams.
