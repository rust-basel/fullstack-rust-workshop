# Database unique lists

In the end you want share a list right? So we have to extend our database model.

Instead of having only a `HashMap` of items, let's have a `HashMap` of `HashMap` of items (that sounds already bad - but is more than sufficient for our workshop!)
This way we can have unique lists - and a list is identifiable with its `list_uuid`.

## Wrap a `HashMap` with a new `struct`

Go to your `database.rs` module and create a new struct. This is a list in our sense, that maps `item uuids` to `ShoppingItem`s.

```rust
struct ShoppingList {
    list: HashMap<String, ShoppingItem>,
}
```

Let's also create some sane defaults, so our `ShoppingList::default()` already has some items:

```rust
impl Default for ShoppingList {
    fn default() -> Self {
        Self {
            list: [
                (
                    "6855cfc9-78fd-4b66-8671-f3c90ac2abd8".to_string(),
                    ShoppingItem {
                        title: "Coffee".to_string(),
                        creator: "Roland".to_string(),
                    },
                ),
                (
                    "3d778d1c-5a4e-400f-885d-10212027382d".to_string(),
                    ShoppingItem {
                        title: "Tomato Seeds".to_string(),
                        creator: "Tania".to_string(),
                    },
                ),
            ]
            .into(),
        }
    }
}
```

If you never came accross the `into()` method: It transform this `[]`-array into the type of `Self`, which is a `HashMap<String, ShoppingItem>`.

## Change `InMemoryDatabase` to wrap `HashMap<String, ShoppingList>`

In this chapter we create the outer HashMap - which we already have.
Change our existing `InMemoryDatabase`. 

```rust
pub struct InMemoryDatabase {
    inner: HashMap<String, ShoppingList>,
}
```
Also - change the `Default` implementation of `InMemoryDatabase`:

```rust
impl Default for InMemoryDatabase {
    fn default() -> Self {
        let mut inner = HashMap::new();
        inner.insert(
            "9e137e61-08ac-469d-be9d-6b3324dd20ad".to_string(),
            ShoppingList::default(),
        );
        InMemoryDatabase { inner }
    }
}
```

We now changed the members of our `InMemoryDatabase`. Let's change our API next.
 
## Change the `InMemoryDatabase` API and implementation

As we are going to have more than one lists - when we for instance want to receive an item, we also have to know from which list we want to receive that item.
So all our current APIs of the `InMemoryDatabase` now also need a `list_uuid` to an `item_uuid`.

Let's change our implementation and API at once and look at it in detail afterwards:

```rust
impl InMemoryDatabase {
    pub fn insert_item(&mut self, list_uuid: &str, item_uuid: &str, shopping_item: ShoppingItem) {
        self.inner
            .get_mut(list_uuid)
            .and_then(|list| list.list.insert(item_uuid.to_string(), shopping_item));
    }

    pub fn delete_item(&mut self, list_uuid: &str, item_uuid: &str) {
        self.inner
            .get_mut(list_uuid)
            .and_then(|list| list.list.remove(item_uuid));
    }

    pub fn create_list(&mut self, list_uuid: &str) {
        self.inner
            .insert(list_uuid.to_string(), ShoppingList::default());
    }

    fn get_list(&self, list_uuid: &str) -> Option<&ShoppingList> {
        self.inner.get(list_uuid)
    }

    pub fn as_vec(&self, list_uuid: &str) -> Vec<ShoppingListItem> {
        let list = self.get_list(list_uuid);
        match list {
            Some(list) => list
                .list
                .iter()
                .map(|(key, item)| ShoppingListItem {
                    title: item.title.clone(),
                    posted_by: item.creator.clone(),
                    uuid: key.clone(),
                })
                .collect(),
            None => Vec::default(),
        }
    }
}
```

If you ask yourself now - what the hell is `and_then`? It allows you to chain calls together. For example look at the new `delete_item` method. We first try to receive a list, given a `list_uuid`.
That can return `Some(list)`, or None, if we do not have list with that given uuid. If it is something, then `and_then` will continue with whatever is given to it as closure.

If the `get_mut` would return None, then the `and_then` is a noop, doing nothing.

If you noticed: We also added two new methods - `get_list` and `create_list`. Of course, when we want to create a new list, we have to enter an entry in our outer HashMap. The `get_list` is used only internally in the
`as_vec` method.

## Making things compile again.

We changed a lot. Our new API needs a `list_uuid`. For now, let's hardcode that with a fixed value, wherever the API is used - and make it adaptable later.
Go to your `controllers.rs` module in the backend - and hardcode our `list_uuid`

```rust
const LIST_UUID: &str = "9e137e61-08ac-469d-be9d-6b3324dd20ad";
```

and use it everywhere, a `list_uuid` is needed. We will change it later, when we have a `list_uuid` available.

Below the changed controllers:

```rust
pub async fn get_items(State(state): State<Database>) -> impl IntoResponse {
    let items: Vec<ShoppingListItem> = state.read().unwrap().as_vec(LIST_UUID);

    Json(items)
}
```

```rust
pub async fn add_item(
    State(state): State<Database>,
    Json(post_request): Json<PostShopItem>,
) -> impl IntoResponse {
    let item = ShoppingItem {
        title: post_request.title.clone(),
        creator: post_request.posted_by.clone(),
    };
    let item_uuid = Uuid::new_v4().to_string();

    let Ok(mut db) = state.write() else {
        return (StatusCode::SERVICE_UNAVAILABLE).into_response();
    };

    db.insert_item(LIST_UUID, &item_uuid, item);

    (
        StatusCode::OK,
        Json(ShoppingListItem {
            title: post_request.title,
            posted_by: post_request.posted_by,
            uuid: item_uuid,
        }),
    )
        .into_response()
}
```

```rust
pub async fn delete_item(
    State(state): State<Database>,
    Path(uuid): Path<Uuid>,
) -> impl IntoResponse {
    let Ok(mut db) = state.write() else {
        return StatusCode::SERVICE_UNAVAILABLE;
    };

    db.delete_item(LIST_UUID, &uuid.to_string());

    StatusCode::OK
}
```

If you run everything - nothing should have changed. Hardcoding values is not nice. We will fix that issue in the next chapter, when we change our `backend` API. We also 
add new `route`s, such as creating a list.
