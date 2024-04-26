# Displaying a list

Now we are going to connect our to parts!
We are going to display a list in the frontend, which is served by our backend.

## Shared Models

As we now connect our front- and backend, it makes sense to share the data models (otherwise we have to define those in both crates).
On the backend we need to serialize to json. On the frontend we need to deserialize from json.

So go ahead top-level and create a new crate. As this is a library, we create it with the `--lib` tag.

```sh
cargo new --lib model
```

In this crate we will put all of our models, that are shared between our front- and backend.
Go into this new model crate and add `serde` to this crate

```sh
cargo add serde -F derive
```

Then go ahead and add our first shared model.

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
pub struct ShoppingListItem {
    pub title: String,
    pub posted_by: String,
    pub uuid: String,
}
```

Great - now we can use this structure on both sides! Do not forget to add this crate to our workspace (top level `Cargo.toml`).

```toml
[workspace]
resolver = "2"
members = [
    "frontend",
    "backend",
    "model"
]
```

## Makefile

Let's first introduce a `Makefile.toml` which will make our life much easier. With `cargo-make`.
We can execute our three commands concurrently with one command, which is:
- serving our backend
- serving our frontend
- telling tailwind to watch files, and to recompile the css if needed.

Also we use `cargo-watch`, that will watch specific files and recompile our code, once we save one of the watched files.

Go ahead an create `Makefile.toml` at the top level.

```toml
[tasks.backend-dev]
install_crate = "cargo-watch"
command = "cargo"
args = ["watch", "-w", "backend", "-w", "model", "-x", "run --bin backend"]

[tasks.frontend-dev]
install_crate = "dioxus-cli"
command = "dx"
args = ["serve", "--bin", "frontend", "--hot-reload"]

[tasks.tailwind-dev]
command = "npx"
args = ["tailwindcss", "-i",  "./frontend/input.css", "-o", "./frontend/public/tailwind.css", "--watch"]

[tasks.dev]
run_task = { name = ["backend-dev", "tailwind-dev", "frontend-dev"], parallel = true}
```

This file describes all the processes it will execute, when you type in `cargo make`.
But for this to work go ahead and install `cargo-watch` `cargo-make`

```sh
cargo install cargo-watch
cargo install cargo-make
```

Great! Now, if you execute

```sh
cargo make --no-workspace dev
```

Every single command we used before is now executed within one command. With hot-reload!

## Backend

Before we can fetch a list in the frontend, let's offer a list in backend first.
For this, go to your backend crate and use the struct, we defined in our model crate. 

Therefore, add the `model` crate as dependency to the `backend` `Cargo.toml`.

```toml
model = { path = "../model" }
```

Then add a use statement in our current `main.rs`:

```rust
use model::ShoppingListItem;
```

Then we need a controller, that sends those items:

```rust
async fn get_items() -> impl IntoResponse {
    let items = vec!["milk", "eggs", "potatoes", "dogfood"];

    let uuid: &str = "a28e2805-196b-4cdb-ba5c-a1ac18ea264a";
    let result: Vec<ShoppingListItem> = items
        .iter()
        .map(|item| ShoppingListItem {
            title: item.to_string(),
            posted_by: "Roland".to_string(),
            uuid: uuid.to_string(),
        })
        .collect();

    Json(result)
}
```

For now we ignore the uuid generation. Every item has the same - but we'll need it for later.
Same for Roland ;o.

Now add this to our router

```rust
      .route("/items", get(get_items));
```

and run `cargo make --now-workspace dev`. Opening a webbrowser with `localhost:3001/items` should give you those items.

## Frontend

Now that you have a backend, which can serve a list of items. Let's build a rudimentary frontend, that can display the list.
Hop to your `frontend` crate.

### Fetching items

First let's add some logic to fetch data from the backend. We can do this by using the `reqwest` crate (a https client library). We also need to add our 
`model crate`.

```sh
cargo add reqwest -F json
cargo add model --path ../model
```

Let's add a small function, that is fetching items.

```rust
use model::ShoppingListItem;

async fn get_items() -> Result<Vec<ShoppingListItem>, reqwest::Error> {
    let url = "http://localhost:3001/items";
    let list = reqwest::get(url)
        .await?
        .json::<Vec<ShoppingListItem>>()
        .await;

    list
}
```

Quite some things are happening here. Let us look at the lines. We fetch something from the backend with `reqwest::get(url).await`, that returns
a `Result`, containing the payload of that request, or an error. The `?`-operator unwraps this Result - continuing the chain, if the Result was ok.
If it was an error, the `?`-operator stops immediately and returns the error. It also converts the error to the error type from the function, that is `reqwest:Error`.

Then the successfully loaded payload is deserialized with `.json::<Vec<ShoppingListItem>>`. That also can fail, so this returns a Result.
But instead of checking this Result with another `?`-operator, we just return this Result from the function.

### Displaying items 

`Dioxus` is a component based web framework - it is comparable to e.g. `React`, where you nest components into one another.
Also, if one of the properties in a component changes, the components will be re-rendered.

So let's create a component, that displays a single item - and then embed this item component into a list, displaying all components

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
        }
    }
}
```

Anytime one of the properties, either `display_name` or `posted_by` is changing, the component is going to be re-rendered.

Now we are going to use the above component to display a list. In Another component

```rust
#[component]
fn ShoppingList() -> Element {
    let items_request = use_resource(move || async move { get_items().await });

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
                                posted_by: i.posted_by.clone()
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
The `use_resource` is one of `Dioxus` [hooks](https://dioxuslabs.com/learn/0.5/reference/hooks). Hooks help you to have stateful functionality in your components.
`use_resource` especially lets you run async closures and return a result. In this case we `match` the result
of the close. On the first render, there will be no data available. So matching will result in a `None`. The component will be re-rendered,
once the future is finished and the result will be `Some(...)`thing. 

If you now add the `ShoppingList` component to our App and execute our `cargo make` command (if it's not running already)
you will see our list!

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
    }
}
```

Great! Now you got the basics! Let's head to our backend to create a rudimentary database - so we can later also add or remove items to the list.


### CORS

It may happen, that your browser denies fetching data from your backend because of Cross-Origin Resource Sharing (CORS).
For security reasons browser restrict cross-origin HTTP requests. Your server has to explicitly tell the browser, which 
sources, or oirigins, are allowed to fetch the server's resources.
For more info have a read at the [MDN Web docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS).

In order to tell our server, that it is ok to fetch data from it we can add a `cors` layer to our `Router`. That `layer` is another middleware.
So the `cors` logic is applied to all routes we already wrote.

Therefore add the `tower-http` crate.

```sh
cargo add tower-http -F cors
```

Afterwards you can add a layer around your existing routes, like so:

```rust
use tower_http::cors::CorsLayer;
```

And in your main function:
```rust
let app = Router::new()
    .route("/", get(hello_world))
    .route("/:name", get(hello_name))
    .route("/your-route", post(workshop_echo))
    .route("/items", get(get_items))
    .layer(CorsLayer::permissive());
```

Note: Do not use `permissive` in production - but choose a cors policy, that fits your setting in production.

### CSS hot-reloading
You might have noticed so far that you had to refresh or recompile for tailwind classes to refresh on the screen.
For proper hot reloading, you need the line

`const _STYLE: &str = manganis::mg!(file("public/tailwind.css"));`

in the frontend main, as well as an explicit `Dioxus.toml` file. 
(so far we have worked with implicit defaults, such as the style property).
The `style` property has to be set to `[]` for manganis to work.

It is a good indicator of how mature a frontend framework is in 2024, if it provides reliable tailwind hot reloading support 
out of the box or with a plugin or two. Dioxus is surprisingly far, but the reloading is not fully reliable yet.