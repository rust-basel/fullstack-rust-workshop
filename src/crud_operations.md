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

## Creating things

In order to add new items to the lists, we have to implement a controller enabling a `POST` with a `json` paylod. If you remember, you did this one
in the beginning.


In order to create a new item, we need two fields. One for the title and one for the creator. Let's create a Data Transfer Object (DTO).
As this one is used in both of our crates, the `model` crate would be a good fit the place it in.

```rust
#[derive(Serialize, Deserialize)]
pub struct PostShopItem {
    pub title: String,
    pub posted_by: String,
}
```

Now our controller in the `backend` can deserialize this object, coming from our frontend.
But who creates the unique id? Our backend does this. Therefore, let's add a crate that creates uuids.

```sh
cargo add uuid -F serde -F v4
```

Now let's create our controller:

Add the correct use statements at the top:

```rust
use model::{PostShopItem, ShoppingListItem};
use uuid::Uuid;
use axum::http::StatusCode;
use crate::database::ShoppingItem;
```

Then the controller - using all these:

```rust
pub async fn add_item(
    State(state): State<Database>,
    Json(post_request): Json<PostShopItem>,
) -> impl IntoResponse {
    let item = ShoppingItem {
        title: post_request.title.clone(),
        creator: post_request.posted_by.clone(),
    };
    let uuid = Uuid::new_v4().to_string();

    let Ok(mut db) = state.write() else {
        return (StatusCode::SERVICE_UNAVAILABLE).into_response();
    };

    db.insert_item(&uuid, item);

    (
        StatusCode::OK,
        Json(ShoppingListItem {
            title: post_request.title,
            posted_by: post_request.posted_by,
            uuid,
        }),
    )
        .into_response()
}
```

There is happening a lot. Let's go through it.

The `State` extractor fetches your database, you injected in you main. We need this to write to the database and insert a new item.
Then the `Json` extractor deserializes your recently created `PostShopItem`. The cool thing here: If the request is not correct - i.e. the client
posts some other json the expected, the controller will automatically return a 400 code for invalid input data.

In the body we create the datamodel, we have in our database. Therefore we transform the json payload to our `ShoppingItem` type.
Then we create a universally unique identifier with the `Uuid` crate (There are different uuid version - we just pick v4). 

Then there is rust syntax, the `let-else` statement, you probably did not see before. It binds the Ok value out of the Result coming from the `state.write()` statement.
That is, if we get the lock to write - then we proceed. Otherwise, the `else` path, we return early with a service unavailable error code (503).

If we can write, we then insert the new item with the generated uuid and respond with the newly generated item as a json serialized `ShoppingListItem`.

The last thing we have to do, is now add this controller as `POST` callback into our axum `Router`.
Go ahead an do that and do not forget the use statement on top:

```rust
use controllers::{add_item, get_items};
```

In your `fn main`:
```rust
let app = Router::new()
    .route("/items", get(get_items).post(add_item))
    .with_state(db);
```

If you now run everything with our `all-do` cargo make, then you can already add new items with e.g. curl, or postman.

Here a curl example:

```sh
curl -i \
    -H "Accept: application/json" \
    -H "Content-Type: application/json" \
    -X POST -d '{"title": "Pepperoni", "posted_by": "Rustacean"}' \
    http://localhost:3001/items
```

After adding the item(s), reload the our frontend, that is running on `localhost:8080` (if you ran the `all-do` cargo make).

Great - now you know how to add items! Let's fast forward and add the code to also delete items.
