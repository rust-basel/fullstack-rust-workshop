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

We need a form with two fields. Namely the `item name`, as well as the person who wants that item. An `author`.

Let's begin simple. We need a form that has two input fields - and a button to commit those fields. Let's add those.

```rust
#[component]
fn ItemInput() -> Element {
    let mut item = use_signal(|| "".to_string());
    let mut author = use_signal(|| "".to_string());

    let onsubmit = move |evt: FormEvent| {};

    rsx! {
        div {
            class: "w-300 m-4 mt-16 rounded",
            form { class: "grid grid-cols-3 gap-2",
                onsubmit: onsubmit,
                div {
                    input {
                        value: "{item}",
                        class: "input input-bordered input-primary w-full",
                        placeholder: "next item..",
                        r#type: "text",
                        id: "item_name",
                        name: "item_name",
                        oninput: move |e| item.set(e.data.value().clone())
                    }
                }
                div {
                    input {
                        value: "{author}",
                        class: "input input-bordered input-primary w-full",
                        placeholder: "wanted by..",
                        r#type: "text",
                        id: "author",
                        name: "author",
                        oninput: move |e| author.set(e.data.value().clone())
                    }
                }
                button {
                    class: "btn btn-primary w-full",
                    r#type: "submit",
                    "Commit"
                }
            }
        }
    }
}
```

The content inside the `rsx` might seem long. But in the end you have three elements. Namely two `input` fields, as well as a `button` to commit the
two input fields. Currently, we do nothing. If you ask yourself what the `use_signal`s are doing: They give this component state.

When the component is rendered first, the we initialize `item` and `author` to empty Strings. Everytime we input something into the input fields,
the respective symbols are set. The plan is: If we press the button, then we commit the current stored strings and build an item out them,
which we will push to our `backend`.

But let's add this component to our existing app.

```rust
pub fn App() -> Element {
    let rust_basel = "Rust Basel";
    rsx! {
        h1{
            "Welcome to {rust_basel}"
        }
        button{
            class: "btn",
            "My stylish button"
        }
        ShoppingList{}
        ItemInput{}
    }
}
```

If you run our `cargo make --no-workspace dev`, then you can already see the finished form.




- Create components to add items from the frontend
- Consume the created post endpoint

TODO
