# Deleting Items

In this chapter we are going to clean up our frontend crate, as well as creating components, that will delete items from our
shopping list. 

## Cleanup

As you already see, we have `#[components]` as well as helper functions in our code base. It would make sense to put all components inside a 
components module and helper functions to fetch and post (and later delete) items into a util/controller module.

Go ahead an put all Components and our `ListChanged` struct into a `components.rs` module, as well as helper functions for getting and posting data to our backend into 
a `controllers.rs` module.

Do also not forget to put the moved functions into the `main.rs`:

```rust
mod controllers;
mod components;
```

In order to use symbols across modules (structs and functions), you have to make those `pub` (public). In the end the `main.rs` function should only be left with our `App` function, the main entry point for our app.

## Creating a helper function to delete items

Like in the previous examples (e.g. adding an item) we now add another helper function to delete items. Go to your new `controllers.rs` module in the frontend crate and create an async function `delete_item`, which takes an item_id, a `&str` as input.

```rust
pub async fn delete_item(item_id: &str) -> Result<(), reqwest::Error> {
    reqwest::Client::new()
        .delete(format!("http://localhost:3001/items/{}", item_id))
        .send()
        .await?;

    Ok(())
}
```

In this case we are just interested if the deletion was ok. If an error happens, the `?`-Operator returns a `reqwest::Error`.

## Create the component

Let's add a component to an already existing component! We plan to create a button, which is on top of our item in the list. So if you click the button on the respective item - this item then gets deleted.

Go ahead to your `components.rs` module and add a component with a button with a cross on it:

```rust
#[component]
fn ItemDeleteButton() -> Element {
    // We change this later
    let onclick = move |_| {};

    rsx! {
    button {
        onclick: onclick,
        class: "btn btn-circle",
            svg {
                class: "h-6 w-6",
                view_box: "0 0 24 24",
                stroke: "currentColor",
                stroke_width: "2",
                stroke_linecap: "round",
                stroke_linejoin: "round",
                fill: "none",
                path {
                    d: "M6 18L18 6M6 6l12 12"
                }
            }
        }
    }
}
```

If you add this component then to our `ShoppingListItemComponent`, then you can already see the new button.

```rust
#[component]
fn ShoppingListItemComponent(display_name: String, posted_by: String) -> Element {
    rsx! {
        div {
            class: "flex items-center space-x-2",
            p {
                class: "grow text-2xl",
                "{display_name}"
            }
            span {
                "posted by {posted_by}"
            }
            ItemDeleteButton {}
        }
    }
}
```

## Delete item onclick

We currrently do nothing. Let's implement the onclick action - so when we click the item, a delete request is sent to our backend.

At the top of your `components.rs` module:

```rust
use crate::controllers::{delete_item, get_items, post_item};
```

And the onclick in your `ItemDeleteButton`:

```rust
  let onclick = move |_| {
      spawn({
          let item_id = item_id.clone();
          async move {
              let _ = delete_item(&item_id).await;
          }
      });
  };
```

Of course, this does not work yet. Somehow you need an `item_id`. Let's add it as a component `prop`. Go ahead, and change the signature and the use of the following components:

`ItemDeleteButton`:
```rust
#[component]
fn ItemDeleteButton(item_id: String) -> Element {...}
```

`ShoppingLisItemComponent`:

```rust
#[component]
fn ShoppingListItemComponent(display_name: String, posted_by: String, item_id: String) -> Element {
    rsx! {
        div {
            class: "flex items-center space-x-2",
            p {
                class: "grow text-2xl",
                "{display_name}"
            }
            span {
                "posted by {posted_by}"
            }
            ItemDeleteButton {item_id}
        }
    }
}
```

And finally the `ul` in our `ShoppingList` component:

```rust
      ul {
          class: "menu bg-base-200 w-200 rounded-box gap-1",
          for i in list {
              li {
                  key: "{i.uuid}",
                  ShoppingListItemComponent{
                      display_name: i.title.clone(),
                      posted_by: i.posted_by.clone(),
                      item_id: i.uuid.clone()
                  },
              }
          }
      }
```

With this - you will now see in the developer settings in your browser, that you send `delete` requests to the backend.

But again: You do not see the list changing. We have to notify other components, that our list has changed. Let's add this in the next step.


## Propagate our ListChanged signal to notify other components

In order to notify others, we introduced a `ListChanged` signal before. Let's reuse this, to `write` to it - after we successfully deleted an item.
Let's propagate the signal to all involved components:

`ItemDeleteButton`:
```rust
#[component]
fn ItemDeleteButton(item_id: String, change_signal: Signal<ListChanged>) -> Element {
    let onclick = move |_| {
        spawn({
            let item_id = item_id.clone();
            async move {
                let response = delete_item(&item_id).await;
                if response.is_ok() {
                    change_signal.write();
                }
            }
        });
    };

    ...
}
```

`ShoppingListItemComponent`:
```rust
#[component]
fn ShoppingListItemComponent(
    display_name: String,
    posted_by: String,
    item_id: String,
    change_signal: Signal<ListChanged>,
) -> Element {
    rsx! {
        div {
            class: "flex items-center space-x-2",
            p {
                class: "grow text-2xl",
                "{display_name}"
            }
            span {
                "posted by {posted_by}"
            }
            ItemDeleteButton {item_id, change_signal}
        }
    }
}
```

`ShoppingList`:
```rust
#[component]
pub fn ShoppingList(change_signal: Signal<ListChanged>) -> Element {
    let items_request = use_resource(move || async move {
        change_signal.read();
        get_items().await
    });

    match &*items_request.read_unchecked() {
        Some(Ok(list)) => rsx! {
            div { class: "grid place-items-center min-h-500",
                ul {
                    class: "menu bg-base-200 w-200 rounded-box gap-1",
                    for i in list {
                        li {
                            key: "{i.uuid}",
                            ShoppingListItemComponent{
                                display_name: i.title.clone(),
                                posted_by: i.posted_by.clone(),
                                item_id: i.uuid.clone(),
                                change_signal
                            },
                        }
                    }
                }
            }
        },
        Some(Err(err)) => {
            rsx! {
                p {
                    "Error: {err}"
                }
            }
        }
        None => {
            rsx! {
                p {
                    "Loading items..."
                }
            }
        }
    }
}
```

If you now run everything - you can now also delete items with the newly added button! As our `use_resource` hook will be rerun after we `write` to our signal,
the latest list from the backend is fetched - after we deleted an item.

## Small Recap

What did we do in this chapter:
- Adding a new components
- Delete items, when we click a button on this item
- Propagate our signal and notify other components about our deletion
