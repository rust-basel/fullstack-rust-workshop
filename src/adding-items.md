# Adding Items

Now we are going to add items from our frontend!

Therefore we will add a form, that will `post` new items to our backend.
After sending the new item, the list will be fetched again, so we have the latest list.

## Creating a helper function

Let's first create a client function, that sends the post request to our backend. It's similar to the `get_items` function, we have already written.
Go to the `main.rs` of your frontend and create a `post_item` function.

```rust
async fn post_item(item: PostShopItem) -> Result<ShoppingListItem, reqwest::Error> {
    let response = reqwest::Client::new()
        .post("http://localhost:3001/items")
        .json(&item)
        .send()
        .await?
        .json::<ShoppingListItem>()
        .await?;

    Ok(response)
}
```

This function will be used in our component, that we will write next.

## Adding a form

We need a form with two fields. Namely the item name, as well as the person who wants that item.




- Create components to add items from the frontend
- Consume the created post endpoint

TODO
