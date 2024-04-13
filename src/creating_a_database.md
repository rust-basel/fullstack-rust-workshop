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

This translates directly to Create-Read-Update-Delete (`CRUD`). Also, for simplicity, we do not use update here. We only delete, create or get items

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

Let's also add some starting items, when creating a default database

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
