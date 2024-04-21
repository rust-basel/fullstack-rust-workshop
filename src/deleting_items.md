# Deleting Items

In this chapter we are going to clean up our frontend crate, as well as creating components, that will delete items from our
shopping list. 

## Cleanup

As you already see, we have `#[components]` as well as helper functions in our code base. It would make sense to put all components inside a 
components module and helper functions to fetch and post (and later delete) items into a util/controller module.

Go ahead an put all Components and our `ListChanged` struct into a `components.rs` module, as well as helper functions for get- and posting data to our backend into 
a `controllers.rs` module.

Do also not forget to put the moved functions into the `main.rs`:

```rust
mod controllers;
mod components;
```

In order to use symbols across modules (structs and functions), you have to make those `pub` (public). In the end the `main.rs` function should only be left with our `App` function, the main entry point for our app.

## Creating a helper function to delete items

Like in the previous examples (e.g. adding an item) we now add another helper function to delete items. Go to your new `controllers.rs` module in the frontend crateand create an async function `delete_item`, which takes an item_id, a `&str` as input.

```rust
pub async fn delete_item(item_id: &str) -> Result<(), reqwest::Error> {
    reqwest::Client::new()
        .delete(format!("http://localhost:3001/items/{}", item_id))
        .send()
        .await?;

    Ok(())
}
```

In this case we are just interested, if the deletion was ok. If there would happen an error, the `?`-Operator would return a `reqwest::Error`.

## Create the component

TODO

## Add our ListChanged Signal

TODO
