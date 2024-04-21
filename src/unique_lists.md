# Database unique lists

In the end you want share a list right? So we have to extend our database model.

Instead of having only a `HashMap` of items, let's have a `HashMap` of `HashMap` of items (that sounds already bad - but is more than sufficient for our workshop!)
This way we can have unique lists - and a list is identifiable with its `uuid`.

## Wrap a HashMap with a new struct

Go to your `database.rs` module and create a new struct. This is a list in our sense, that maps `item uuids` to `ShoppingItem`s.

```rust
struct ShoppingList {
    list: HashMap<String, ShoppingItem>,
}
```

Let's also create some sane defaults, so our `ShoppingList::default()` already has some items:

```rust
impl Default for ShoppingList {
    fn default() -> Self {
        Self {
            list: [
                (
                    "6855cfc9-78fd-4b66-8671-f3c90ac2abd8".to_string(),
                    ShoppingItem {
                        title: "Coffee".to_string(),
                        creator: "Roland".to_string(),
                    },
                ),
                (
                    "3d778d1c-5a4e-400f-885d-10212027382d".to_string(),
                    ShoppingItem {
                        title: "Tomato Seeds".to_string(),
                        creator: "Tania".to_string(),
                    },
                ),
            ]
            .into(),
        }
    }
}
```

If you never come accross the `into()` method: It transform this `[]`-array into the type of `Self`, which is a `HashMap<String, ShoppingItem>`.

## Change InMemoryDatabase to wrap HashMap<String, ShoppingList>

TODO: add new member with defaults

## Change the InMemoryDatabase API

TODO: list AND item uuid

## Chaining things

TODO: change implementation of functions + chain with `and_then(...)`
