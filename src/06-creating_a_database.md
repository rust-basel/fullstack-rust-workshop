# Creating a database

For this workshop we will create a small in memory database for simplicity reasons. Basically a tiny wrapper around a `HashMap` with items.
For a real adapter, please have a look at [sqlx](https://github.com/launchbadge/sqlx) or an Object Relational Mapper (ORM) like
[SeaOrm](https://www.sea-ql.org/SeaORM/) or [Diesel](https://diesel.rs/). The cool thing about axum is,
it let's you inject our database like the ones referenced. So if you decide later on to have production ready
database implementation - you use it the same way within `axum`.

But what exactly should our database be capable of?

- Get/Read an item
- Get all items
- Create an item
- (Update an existing item)
- Delete an item

This translates directly to Create-Read-Update-Delete (`CRUD`). Also, for simplicity, we do not use update here. 
We only delete, create or get items. Deleting is therefore the same as checking of an item.

## Create a database module

Let's create an own module in our `backend` crate for this.
Add a new file called `database.rs` next to the `main.rs`. To make the `backend` aware of the new module,
add 

```rust
mod database;
```

to the top of your `main.rs`.

## Create the database

Inside the new `database.rs` module create a new struct, called `InMemoryDatabase`, which wraps a simple HashMap

```rust
use std::collections::HashMap;

pub struct InMemoryDatabase {
    inner: HashMap<String, ShoppingItem>,
}
```

Let's also add some logic to retrieve, add and delete items

```rust
impl InMemoryDatabase {
    fn get_item(&self, uuid: &str) -> Option<&ShoppingItem> {
        self.inner.get(uuid)
    }

    fn insert_item(&mut self, uuid: &str, item: ShoppingItem) {
        self.inner.insert(uuid.to_string(), item);
    }

    fn delete_item(&mut self, uuid: &str) {
        self.inner.remove(uuid);
    }
}
```

Let's also add some fixture items, when creating a default database

```rust
impl Default for InMemoryDatabase {
    fn default() -> Self {
        let inner: HashMap<String, ShoppingItem> = [
            (
                "b8906da9-0c06-45a7-b117-357b784a8612".to_string(),
                ShoppingItem {
                    title: "Salt".to_string(),
                    creator: "Yasin".to_string(),
                },
            ),
            (
                "ac18131a-c7b8-4bdc-95b5-e1fb6cad4576".to_string(),
                ShoppingItem {
                    title: "Milk".to_string(),
                    creator: "Tim".to_string(),
                },
            ),
        ]
        .into_iter()
        .collect();

        Self { inner }
    }
}
```

## Adding the database to our axum backend

Adding this database is fairly simple with axums `Router` builder.
Go inside the `backends` main function, init a default database and use it inside the router.

Before we can use our database, we have to make sure that there is no data race. Or - the compiler checks that for us.

Make a type alias, where we wrap our database in an `Arc<RwLock>`. An Arc is an `atomic` and reference counted smart pointer in Rust.
The `RwLock` makes sure, that reads and writes are mutually exclusive. There is only one writer holding a lock. But several readers can hold onto the same lock.

```rust
use std::sync::{Arc, RwLock};
use database::InMemoryDatabase;

type Database = Arc<RwLock<InMemoryDatabase>>;
```

Afterwards you can just initialize this database as default - and inject it with axums `with_state(...)` router method.

```rust
#[tokio::main]
async fn main() {
    let db = Database::default();
    let app = Router::new()
        .route("/", get(hello_world))
        .route("/:name", get(hello_name))
        .route("/your-route", post(workshop_echo))
        .route("/items", get(get_items))
        .with_state(db);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3001").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

This allows to access the state (db) from each handler function by using it as function parameters.

In the next chapter, we will write `controllers` functions for update, read and delete endpoints, where we use the database.
