# Displaying a list

Now we are going to connect our to parts!
We are going to display a list in the frontend, which is served by our backend.

## Shared Models

As we now connect the our front and backend, it makes sense to share the data models (otherwise we have to define those in both crates).
On the backend we need to serialize to json. On the frontend we need to deserialize from json.

So go ahead top-level and create a new crate. As this is a library, we create it with the `--lib` tag.

```sh
cargo new --lib model
```

In this crate we will put all of our models, that are shared between our front- and backend.
Go into this new model crate and add `serde` to this crate

```sh
cargo add serde -F derive
```

Then go ahead and add our first shared model.

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
pub struct ShoppingListItem {
    pub title: String,
    pub posted_by: String,
    pub uuid: String,
}
```

Great - now we can use this structure on both sides! Do not forget to add this crate to our workspace (top level `Cargo.toml`).

```toml
[workspace]
resolver = "2"
members = [
    "frontend",
    "backend",
    "model"
]
```

## Makefile

Let's first introduce a `Makefile.toml` which will make our life much easier. With `cargo-make`.
We can execute our three commands concurrently with one command, which is:
- servig our backend
- serving our frontend
- telling tailwind to watch files, and to recompile the css if needed.

Also we use `cargo-watch`, that will watch specific files and recompile our code, once we save one of the watched files.

Go ahead an create `Makefile.toml` at the top level.

```toml
[tasks.backend-dev]
install_crate = "cargo-watch"
command = "cargo"
args = ["watch", "-w", "backend", "-w", "model", "-x", "run --bin backend"]

[tasks.frontend-dev]
install_crate = "dioxus-cli"
command = "dx"
args = ["serve", "--bin", "frontend", "--hot-reload"]

[tasks.tailwind-dev]
command = "npx"
args = ["tailwindcss", "-i",  "./frontend/input.css", "-o", "./frontend/public/tailwind.css", "--watch"]

[tasks.dev]
run_task = { name = ["backend-dev", "tailwind-dev", "frontend-dev"], parallel = true}
```

This file describes all the processes it will execute, when you type in `cargo make`.
But for this to work go ahead and install `cargo-watch` `cargo-make`

```sh
cargo install cargo-watch
cargo install cargo-make
```

Great! Now, if you execute

```sh
cargo make --no-workspace dev
```

Every single command we used before is now executed within one command. With hot-reload!

## Backend

Before we can fetch a list in the frontend, let's offer a list in backend first.
For this, go to your backend create and use the struct, we defined in our model crate. 

Therefore, add the `model` crate as dependency to the `backend` crate to its `Cargo.toml`.

```toml
model = { path = "../model" }
```

Then add a use statement:

```rust
use model::ShoppingListItem;
```

Then we need a controller, that sends those items:

```rust
async fn get_items() -> impl IntoResponse {
    let items = vec!["milk", "eggs", "potatoes", "dogfood"];

    let uuid: &str = "a28e2805-196b-4cdb-ba5c-a1ac18ea264a";
    let result: Vec<ShoppingListItem> = items
        .iter()
        .map(|item| ShoppingListItem {
            title: item.to_string(),
            posted_by: "Roland".to_string(),
            uuid: uuid.to_string(),
        })
        .collect();

    Json(result)
}
```

For now we ignore the uuid generation. Every item has the same - but we'll need it for later.
Same for Roland ;o.

Now add this to our router

```rust
      .route("/items", get(get_items));
```

and run `cargo make --now-workspace dev`. Opening a webbrowser with `localhost:3001/items` should give you those items.

## Frontend

Now that you have a backend, which can serve a list of items. Let's build a rudimentary frontend, that can display the list.
Hop to your `frontend` crate.

### Fetching items

First let's add some logic to fetch data from the backend. We can do this by using the `reqwest` crate. We also need to add our 
`model crate`.

```rust
cargo add reqwest -F json
cargo add model --path ../model
```

Let's add a small function, that is fetching items.

```rust
use model::ShoppingListItem;

async fn get_items() -> Result<Vec<ShoppingListItem>, reqwest::Error> {
    let url = "http://localhost:3001/items";
    let list = reqwest::get(url)
        .await?
        .json::<Vec<ShoppingListItem>>()
        .await;

    list
}
```

Quite some things are happening here. Let us look at the lines. We fetch something from the backend with `reqwest::get(url).await`, that returns
a Result, containing the Payload of that request, or an error. The `?`-operator unwraps this Result - continueing the chain, if the Result was ok.
If it was an error, the `?`-operator stops immediatly and returns the error. It also converts the error to the error type from the function, that is `reqwest:Error`.

Then the successfully loaded payload is deserialized with `.json::<Vec<ShoppingListItem>>`. That also can fail, so this returns a Result.
But instead of checking this Result with another `?`-operator, we just return this Result from the function.
