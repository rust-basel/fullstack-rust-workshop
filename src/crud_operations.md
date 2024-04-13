# CRUD Operations

Now that we have a "database" usable in our axum backend. Let's write handling functions, which have access to the database and read and modify it.
Head to your `backend` crate and create again a new module. This time it's called `controllers.rs`. Why named controllers?
There is a famous design pattern called [model, view, controller](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller). Controllers are usually those parts, which execute your logic.
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

Afterwards your `main.rs` should look like this:

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

As well as your `controller.rs`:

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

## Reading things

Let's rewrite our `get_items` controller. Instead of returning a hardcoded list, let's fetch them from the database.
Change the signature of your function.

```rust
use axum::extract::State;

pub async fn get_items(State(state): State<Database>) -> impl IntoResponse {...}
```

Now you can access the database, that you've already injected with the `with_state(...)` method of the `Router`.

But first, got to your `database.rs` module and add a new method to your `impl InMemoryDatabase` to return all items at once. 

```rust
  pub fn as_vec(&self) -> Vec<(String, ShoppingItem)> {
      self.inner
          .iter()
          .map(|(uuid, item)| (uuid.clone(), item.clone()))
          .collect()
  }
```

For this, the database item must be clone, so add this also to the item. Morover the struct itself and its members must be public,
so one can use it outside the `database.rs` module.

```rust
#[derive(Clone)]
pub struct ShoppingItem {
    pub title: String,
    pub creator: String,
}
```

With those changes, we now can rewrite our `get_items` function to fetch items from the database.

```rust
pub async fn get_items(State(state): State<Database>) -> impl IntoResponse {
    let items: Vec<ShoppingListItem> = state
        .read()
        .unwrap()
        .as_vec()
        .iter()
        .cloned()
        .map(|(uuid, item)| ShoppingListItem {
            title: item.title,
            posted_by: item.creator,
            uuid,
        })
        .collect();

    Json(items)
}
```

If you now execute all our do-all command `cargo make --no-workspace dev`, you get the items displayed, which are in the database :). Good Job!
