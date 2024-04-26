# Add Routing

**important distinction**: in a SPA setup, we have two routing systems: 

1. The browser navigator (what you see in the URL bar)
2. the routes for API endpoints

This might be obvious if you are familiar with SPA, but there are also different generations of server side rendered frameworks that just have one routing system, and
where the routes in the URL bars are identical with what you request over network (= normal page load, like it's 1999).
Sometimes, you can find combinations of the two separated by a `#`, e.g. `your.backendroute.com/admin#users/2`

We already saw API endpoint routes in action in the axum chapters, so this chapter is a about navigator routing, and we will use routes without any `#`.

## Add the router feature to cargo.toml
```toml
dioxus = { version = "0.5.1", features = ["web", "router"] }
```

## Create a new mock component (profile)
...and refactor your App component (call it e.g. "Home" instead)

```rust
pub fn Profile() -> Element {
    rsx! {
        div {
            div {
                class: "flex flex-col gap-4 w-full",
                div {
                    class: "flex gap-4 items-center",
                    div {
                        class: "skeleton w-16 h-16 rounded-full shrink-0"
                    }
                    div {
                        class: "flex flex-col hap-4",
                        div {
                            class: "skeleton h-4 w-20"
                        }
                        div {
                            class: "skeleton h-4 w-28"
                        }
                    }
                }
                div {
                    class: "skeleton h-32 w-full"
                }
                div {
                    class: "skeleton h-32 w-full"
                }
            }
        }
    }
}
```
Since we are just getting started with Routing and don't want to implement the profile page yet,
we just use daisyui skeletons for a "under construction" site.

## define and use a Router enum
```rust
#[derive(Routable, Clone)]
enum Route {
    #[route("/")]
    Home {},
    #[route("/profile")]
    Profile {}
}
```

This enums maps the route "/" to the Home component (what we had so far under App), and "/profile" to the new Profile page.

```rust
#[allow(non_snake_case)]
fn App() -> Element {
    rsx! {
        Router::<Route>{}
    }
}
```
We can use the router with our new App component, which now renders something different depending on the currently active route.

**Try it in the URL bar!** with `localhost:8080/` and `localhost:8080/profile` to see your routes in action.

# Add a navigation bar to the base layout
We could add Links to our routes, e.g. with
`Link { to: Route::Profile{}, "Profile" }` for the profile page. Instead of repeating these all over as we build more and more pages, let's add a navigation bar to a common layout, which will appear on all pages.

A layout is a component which has an `Outlet`. An oulet is like a window through which you can see the router content, in our example everything below the navigation bar.
At the same time, we can take care about some styling aspects for the layout, e.g. putting 
everything into a container with auto margins to the left and right.

The layout could look like this:
```rust
#[component]
pub fn Layout() -> Element {
    rsx! {
        div {
            class: "min-h-screen bg-base-300",
            div {
                class: "navbar flex",
                div {
                    Link { class: "p-4", to: Route::Home{}, "Home" }
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

and we update our route with the layout info:
```rust
#[derive(Routable, Clone)]
pub enum Route {
    #[layout(Layout)]
        #[route("/")]
        Home {},
        #[route("/profile")]
        Profile {}
}
```

The indentation as suggested by the dioxus docs is not very elegant, but it helps see the structure.

Routes and layouts are the building blocks for complex navigation patterns, they can even be nested e.g. for submenus.
And they can contain dynamic placeholders.
