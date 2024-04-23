# Frontend load unique list

In this final chapter we will do the follwing:
- Creating a helper to create a new list
- Adapter our controller functions to the new routes, we defined in the backend
- Writing a component to either create, or load a list.
- Route to our list after creating or loading one

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
