# Frontend: new Endpoints

In this chapter we will do the follwing:
- Creating a helper to create a new list
- Adapt our controller functions to the new routes, we defined in the backend - so we can build again :)

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

## Propagate the list uuid from top to down 

Now we need the list's uuid everywhere, where we fetch data from the backend. So somehow we need a uuid for this.
Do you have a guess how?

Go to your Home `component` and add a `list_uuid` signal there. For now we hardcode the value to be `9e137e61-08ac-469d-be9d-6b3324dd20ad` (the first existing list in our backend - if you remember - also hardcoded ;)).
The idea now: propagate down this `list_uuid` to all components, that need a `list_uuid`, which are the `ShoppingList` and the `ItemInput` components (and all the components they use).

```rust
#[component]
pub fn Home() -> Element {
    let list_uuid = use_signal(|| "9e137e61-08ac-469d-be9d-6b3324dd20ad".to_string());
    let change_signal = use_signal(|| ListChanged);
    rsx! {
        ShoppingList{list_uuid, change_signal}
        ItemInput{list_uuid, change_signal}
    }
}
```

Of course, those components have to be adjusted in their parameters (or so called: Component `props`). Go ahead and change the props of those. Then you can use the `list_uuid` in the requests fired by the hooks used in the components. The easiest way to accomplish this is forwarding the signal, and read it inside the component.

To spare you more headaches - use these adjusted components. The only thing that changed, is that they now all use the added `list_uuid`.

`ShoppingListItemComponent`
```rust
#[component]
fn ShoppingListItemComponent(
    display_name: String,
    posted_by: String,
    list_uuid: String,
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
            ItemDeleteButton {list_uuid, item_id, change_signal}
        }
    }
}
```
`ShoppingList`
```rust
#[component]
pub fn ShoppingList(list_uuid: Signal<String>, change_signal: Signal<ListChanged>) -> Element {
    let items_request = use_resource(move || async move {
        change_signal.read();
        get_items(list_uuid.read().as_str()).await
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
                                list_uuid,
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

`ItemInput`
```rust
#[component]
pub fn ItemInput(list_uuid: Signal<String>, change_signal: Signal<ListChanged>) -> Element {
    let mut item = use_signal(|| "".to_string());
    let mut author = use_signal(|| "".to_string());

    let onsubmit = move |_| {
        spawn({
            async move {
                let item_name = item.read().to_string();
                let author = author.read().to_string();
                let response = post_item(
                    list_uuid.read().as_str(),
                    PostShopItem {
                        title: item_name,
                        posted_by: author,
                    },
                )
                .await;

                if response.is_ok() {
                    change_signal.write();
                }
            }
        });
    };

...
}
```

`ItemDeleteButton`
```rust
#[component]
fn ItemDeleteButton(
    list_uuid: String,
    item_id: String,
    change_signal: Signal<ListChanged>,
) -> Element {
    let onclick = move |_| {
        spawn({
            let list_uuid = list_uuid.clone();
            let item_id = item_id.clone();
            async move {
                let response = delete_item(&list_uuid, &item_id).await;
                if response.is_ok() {
                    change_signal.write();
                }
            }
        });
    };

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

After you changed these components - your frontend should build again by the way :) You can test it with again with 

```sh
cargo make --no-workspace dev
```

In the next chapter we will add a new component, that creates a new list or gets a list uuid by user input.

You almost made it :) hang on!
