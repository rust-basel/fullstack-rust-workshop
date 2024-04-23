# Frontend load unique list

In this final chapter we will do the follwing:
- Creating a helper to create a new list
- Adapter our controller functions to the new routes, we defined in the backend
- Writing a component to either create, or load a list.
- Route to our list after creating or loading one

## Creating a helper to create a new list

In order to create a new list, we have to write a new controller function and add it to our `controllers.rs` module. As you already know, how to do this one - go ahead and this to your `controllers.rs`:

```rust
async fn create_list() -> Result<CreateListResponse, reqwest::Error> {
    let response = reqwest::Client::new()
        .get("http://localhost:3001/list")
        .send()
        .await?
        .json::<CreateListResponse>()
        .await?;

    Ok(response)
}
```

This will make a `GET` request to our `/list` endpoint - and in return we will receive an uuid for our new list. 

## Adapt controllers to new endpoints

As you remember - we changed our endpoints in our `backend`. Go ahead and change the other controllers.
As an example we look at the existing `delete_item` function.

```rust
pub async fn delete_item(item_id: &str) -> Result<(), reqwest::Error> {
    reqwest::Client::new()
        .delete(format!("http://localhost:3001/items/{}", item_id))
        .send()
        .await?;

    Ok(())
}
```

As we now have to know in which list we want to delete the item from, we have to expand the arguments list for a `list_id` (the lists uuid).
You can just append the argument to the `format!(..)` string. In the end, the function should look like this:

```rust
pub async fn delete_item(list_id: &str, item_id: &str) -> Result<(), reqwest::Error> {
    reqwest::Client::new()
        .delete(format!(
            "http://localhost:3001/list/{}/items/{}",
            list_id, item_id
        ))
        .send()
        .await?;

    Ok(())
}
```

As the other functions are kind of straight forward - go ahead and change the other functions, which spoke against the "old" REST api.

```rust
pub async fn get_items(list_id: &str) -> Result<Vec<ShoppingListItem>, reqwest::Error> {
    let url = format!("http://localhost:3001/list/{}/items", list_id);
    let list = reqwest::get(&url)
        .await?
        .json::<Vec<ShoppingListItem>>()
        .await;

    list
}

pub async fn post_item(
    list_id: &str,
    item: PostShopItem,
) -> Result<ShoppingListItem, reqwest::Error> {
    let response = reqwest::Client::new()
        .post(format!("http://localhost:3001/list/{}/items", list_id))
        .json(&item)
        .send()
        .await?
        .json::<ShoppingListItem>()
        .await?;

    Ok(response)
}
```
