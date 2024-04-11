# Displaying a list

Now we are going to connect our to parts!
We are going to display a list in the frontend, which is served by our backend.

## Makefile

But first we introduce a `Makefile.toml` which will make our life much easier. With `cargo-make`.
We can execute our three commands concurrently with one command, which is:
- servig our backend
- serving our frontend
- telling tailwind to watch files, and to recompile the css if needed.

Go ahead an create `Makefile.toml` at the top level.

```toml
[tasks.backend-dev]
install_crate = "cargo-watch"
command = "cargo"
args = ["watch", "-w", "backend", "-x", "run --bin backend"]

[tasks.frontend-dev]
install_crate = "dioxus-cli"
command = "dx"
args = ["serve", "--bin", "frontend", "--hot-reload"]

[tasks.tailwind-dev]
command = "npx"
args = ["tailwindcss", "-i",  "./frontend/input.css", "-o", "./frontend/public/tailwind.css"]

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

every stepped we talked before, is now executed within one command. With hot-reload!

## Backend

Before we can fetch a list in the frontend, let's offer a list in backend first.
For this, go to your backend create and add a Serialize struct, that we are going to send.
This will be our Shopping Item:

```rust
#[derive(Serialize, Deserialize, PartialEq, Clone, Debug)]
pub struct ShoppingListItem {
    pub title: String,
    pub posted_by: String,
    pub uuid: String,
}
```

Then we obviously need a controller, that sends those items:

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
