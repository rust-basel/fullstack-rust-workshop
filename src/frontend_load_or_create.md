# Frontend: Load or Create - that is the Question

You almost made it. In this chapter we write our component to either load or create a list.
The idea:
- We get a uuid by input or we get a uuid by creating a new list in the backend
- We then route to our main `Home` component to with a `list_uuid`.

## Add a dynamic route

In order to route dynamically to our main `Home` component two things have to be done.

1. We need a dynamic route, something like `/list/<uuid>`.
2. This `<uuid`> parameter then has the be given to our `Home` component. 


Let's start with our `Home` component. Add a `list_uuid` component property.

```rust
#[component]
pub fn Home(list_uuid: String) -> Element {
    let list_uuid = use_signal(|| list_uuid);
    let change_signal = use_signal(|| ListChanged);
    rsx! {
        ShoppingList{list_uuid, change_signal}
        ItemInput{list_uuid, change_signal}
    }
}
```

And you might noticed - we removed our hardcoded uuid :).

Then we also need to have a dynamic route. Go to our `Route` enum (in our frontend).

```rust
#[derive(Routable, Clone)]
pub enum Route {
    #[layout(Layout)]
    #[route("/list/:list_uuid")]
    Home { list_uuid: String },
    #[route("/profile")]
    Profile {},
}
```

Great! with this we are now dynamically routable. Let's create a new component that will route to our new `Home` with a `list_uuid`.

## Create a new list

Let's create a new component, that load or create a new list. You also see the `use_navigator` hook. It allows is to route to other pages we have!

```rust
#[component]
pub fn LoadOrCreateList() -> Element {
    let nav = use_navigator();

    let on_create_list_click = move |_| {
        let nav = nav.clone();
        spawn({
            async move {
                let response = create_list().await;
                if let Ok(created_list) = response {
                    nav.push(Route::Home {
                        list_uuid: created_list.uuid,
                    });
                }
            }
        });
    };

    rsx! {
        div{
            class: "grid place-content-evently grid-cols-1 md:grid-cols-2 w-full gap-4",
            div {
                class: "card glass min-h-500 flex flex-col content-end gap-4 p-4",
                button{
                    class: "btn btn-primary",
                    onclick: on_create_list_click,
                    "Create new List"
                }
            }
        }
    }
}
```

Add this new component to the `Route` as this is now the first page we see, when we open up our url:

```rust
#[derive(Routable, Clone)]
pub enum Route {
    #[layout(Layout)]
    #[route("/")]
    LoadOrCreateList {},
    #[route("/list/:list_uuid")]
    Home { list_uuid: String },
    #[route("/profile")]
    Profile {},
}
```

If you then also route correctly in our `Layout` then we have a working routing in our frontend :) We can only create new list's though

```rust
#[component]
pub fn Layout() -> Element {
    rsx! {
        div {
            class: "min-h-screen bg-base-300",
            div {
                class: "navbar flex",
                div {
                    Link { class: "p-4", to: Route::LoadOrCreateList{}, "Home" }
                    Link { class: "p-4", to: Route::Profile{}, "Profile" }
                }
            }
            div { class: "container mx-auto max-w-[1024px] p-8",
                Outlet::<Route>{}
            }
        }
    }
}
```

If you now run all the code - you can create a new list with a click of a button - and it will route you to the page with the freshly created list.
