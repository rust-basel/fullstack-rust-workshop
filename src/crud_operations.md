# CRUD Operations

Now that we have a "database" usuable in our axum backend. Let's write handling functions, which have access to the database and read and modify it.
Head to your `backend` crate and create again a new module. This time it's called `controllers.rs`. Why named controllers?
There is a famous design pattern [called model, view, controller](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller). Controller are usually the parts, which execute your logic.
In our case, you've already seen them. These are the functions, we give the `Router` as callback in a `route`.

Do not forget to add new module in the `main.rs` file.

```rust
mod controllers;
```

## Cleanup

Let's remove any other controller we currently have - except the `get_items` controller. This function you can move to the new 
`controllers.rs` module - as this will be our first controller we rewrite to use the database instead of the hardcoded list.

Do not forget to make this one public - by prepending a pub. Otherwise it's not visible outside this new module.

```rust
pub async fn get_items() -> impl IntoResponse {...}
```

Afterwards your `main.rs` should look like:

```rust
mod controllers;
mod database;

use std::sync::{Arc, RwLock};

use axum::{routing::get, Router};
use controllers::get_items;
use database::InMemoryDatabase;

type Database = Arc<RwLock<InMemoryDatabase>>;

#[tokio::main]
async fn main() {
    let db = Database::default();
    let app = Router::new().route("/items", get(get_items)).with_state(db);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3001").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

As well as your `controller.rs`

```rust
use axum::{response::IntoResponse, Json};
use model::ShoppingListItem;

pub async fn get_items() -> impl IntoResponse {
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

This should be our starting point.
