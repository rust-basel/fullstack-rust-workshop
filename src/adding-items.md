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

    // We implement this closure later
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
two input fields. Currently, we do not push anything to our `backend` yet. This going to happen in the `onsubmit` action, which we will implement later. If you ask yourself what the `use_signal`s are doing: They give this component state.

When the component is rendered first, we initialize `item` and `author` to empty Strings. Every time we input something into the input fields (`oninput` actions),
the respective symbols are set, when we enter a character. The plan: If we press the button, then we use the current stored strings, build an `PostShopItem` out them and push this to our `backend`.

But let's add this component to our existing app first, so you can already examine the UI.

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

## OnSubmit Action - posting items

Currently we do nothing - except changing the state of the two `hooks`. Let's add a closure, that posts items to our backend.
For this we will use `spawn`. This allows us to run async function inside our components, which are fired once.

```rust
let onsubmit = move |_| {
    spawn({
        async move {
            let item_name = item.read().to_string();
            let author = author.read().to_string();
            let _ = post_item(PostShopItem {
                title: item_name,
                posted_by: author,
            })
            .await;
        }
    });
};
```

As we stored the item as well as the author value inside hooks (our components state), we can just read those values and copy them as strings.
For now we just ignore the result of our http call for simplicity.

If you now run everything you are able to create new items. The downside: You do not recognize that you already post items, as the list 
is not getting updated. But if you open your browser's developer settings (usually `f12`), you will see in your network settings, that you post items.

But let us fix the list in the next step.

## Use resource - dependencies

You can configure the `use_resource` hook. It is run when the components renders the first time. If you want to `re-run` the hook, then you
have to add a dependency. If this dependency changes, the `use_resource` is run again. Let's do exactly this.

Let's first introduce something, that signals a change.

```rust
struct ListChanged;
```

In our case a `zero-sized` object is sufficient.

The idea:
- When we post an item - and the post request has been successfull - then we write to that signal.
- Our list display subscribes this signal. Because we write to it, anything that depends on it (i.e. our `use_resource` hook), will be executed again.

In order that both components `ItemInput` and `ShoppingList` both have access to that signal, we have to `hoist` (lifting the state up) the state.

So let's add a `use_signal` hook on top level and add the signal to both components:

New components signature:

```rust
fn ShoppingList(change_signal: Signal<ListChanged>) -> Element {...}
```
```rust
fn ItemInput(change_signal: Signal<ListChanged>) -> Element {...}
```

and add it to your App function:
```rust
pub fn App() -> Element {
    let change_signal = use_signal(|| ListChanged);
    let rust_basel = "Rust Basel";
    rsx! {
        h1{
            "Welcome to {rust_basel}"
        }
        button{
            class: "btn",
            "My stylish button"
        }
        ShoppingList{change_signal}
        ItemInput{change_signal}
    }
}
```

Let's `read` the signal on the list fetching side - and `write` on the `input` side in the next steps.

### Writing the signal

In the `ItemInput` component, go into your `onsubmit` callback. Now we can use the `Result` coming from our `post_item` http request.
Change it and check the result, if it is ok. If it is ok, we `write` the signal (without writing anything to it).

```rust
  let item_name = item.read().to_string();
  let author = author.read().to_string();
  let response = post_item(PostShopItem {
      title: item_name,
      posted_by: author,
  })
  .await;

  if response.is_ok() {
      change_signal.write();
  }
```

But that is sufficient enough, that other components, that subscribe to that signal (automatically when you have a `read`), to re-render or execute some logic attached to it.

### Reading the signal

Now let's subscribe to that signal by `read`ing it inside our `ShoppingList` component's `use_resource` hook.

```rust
  let items_request = use_resource(move || async move {
      change_signal.read();
      get_items().await
  });
```

That is all you need to do in order let your components *communicate* with each other.

If you did it correctly (if not - have a look at the solution) - when you now add a new item, the list is fetched afterwards and the list updates.

Great! :) 

Let's do the same to delete items in our next chapter.
