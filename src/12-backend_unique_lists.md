# Backend unique lists 

We changed our database - so it's able to handle different lists with different shopping items. But currently we have only one list - as its ID is hardcoded.
Let's change that in this chapter. We expand our `REST` api in such a way, so we can create new lists and load existing ones, without needed to hardcode an ID (in the backend).

## Changing the routes

We currently had the following route: `/items`. But now that we can also have different lists, we should rename our exising routes.

Go to our `main.rs` in the backend - and change the routes from:
- `/items` to `/list/:list_uuid/items`
- `/items/:uuid` to `/list/:list_uuid/items/:item_uuid`

```rust
let app = Router::new()
    .route("/list/:list_uuid/items", get(get_items).post(add_item))
    .route("/list/:list_uuid/items/:item_uuid", delete(delete_item))
    .layer(CorsLayer::permissive())
    .with_state(db);
```

The `:list_uuid` will be extracted from the `path` and is from now on available in our controllers in the `controllers.rs` module.
Perfect! Now let's use the new variable inside our controllers and get rid of the hardcoded `LIST_UUID` in the `controllers.rs` module.

## Using new Path parameters

Now with the new routes, we can extract the extra `list_uuid` parameter - and use it in our controllers.
If you have two path parameters, like in `delete_item`, you now have to extract path parameters as a tuple.
The names inside the tuple bind to the names given in your `route` path. 

Let's start with the `delete_item`, as it's the hardest (path wise):

```rust
pub async fn delete_item(
    State(state): State<Database>,
    Path((list_uuid, item_uuid)): Path<(Uuid, Uuid)>,
) -> impl IntoResponse {
    let Ok(mut db) = state.write() else {
        return StatusCode::SERVICE_UNAVAILABLE;
    };

    db.delete_item(&list_uuid.to_string(), &item_uuid.to_string());

    StatusCode::OK
}
```

Like stated above - your `Path` extractor now needs to work with a tuple of type `(Uuid,Uuid)`. So if one of those parameters is not parsable as an uuid, we will get a 400 error automatically (invalid input).

Let's change the other existing controllers respectively: 

```rust
pub async fn add_item(
    Path(list_uuid): Path<Uuid>,
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

    db.insert_item(&list_uuid.to_string(), &item_uuid, item);

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
and 


```rust
pub async fn get_items(
    Path(list_uuid): Path<Uuid>,
    State(state): State<Database>,
) -> impl IntoResponse {
    let items: Vec<ShoppingListItem> = state.read().unwrap().as_vec(&list_uuid.to_string());

    Json(items)
}
```

Great! Now we adapted our existing controllers and already use the new `list_uuid`. You can now also safely delete the hardcoded `LIST_UUID` value ;).

## Adding new routes

Now that we adapted the existing routes, let's add a new one to create a fresh list.
As we only have to request a `/list` endpoint without to provide any data - we can get away with a `GET` endpoint.
Usually, when you want to create new ressources server side, you would supply a `post` with some data. But for our case we do not need this.

What we need though is a response for this endpoint, as we want to know the `list_uuid` of the newly created list.

Go to the model create and add a new `serializable` and `deserializable` model to the `lib.rs`:

```rust
#[derive(Serialize, Deserialize)]
pub struct CreateListResponse {
    pub uuid: String,
}
```

This model will also be deserialized in the frontend later. So also put a Deserialize from `serde` ontop of the struct.

Go ahead an create a new async function in the `controller.rs` module in our `backend` crate.

```rust
use model::{CreateListResponse, PostShopItem, ShoppingListItem}; // import what is needed.

pub async fn create_shopping_list(State(state): State<Database>) -> impl IntoResponse {
    let uuid = Uuid::new_v4().to_string();
    state.write().unwrap().create_list(&uuid);

    Json(CreateListResponse { uuid })
}
```

The last thing, that is needed, is to put the `create_shopping_list` into our `Router`. You know the drill!
Add it to the router with a `/list` route - fetched via a `GET`.

```rust
.route("/list", get(create_shopping_list))
```

Your are getting somewhere! :). Let's hop on to change the `frontend` also, to get/post and delete with the new routes, we just created.
