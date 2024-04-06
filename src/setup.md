# Setup

In order to have our fullstack project, we need some setup first:

## Prerequistes

For the setup we need as prerequisites:
- [Rust](https://www.rust-lang.org/tools/install) installed
- [npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm) installed 
- [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) installed 

## Project structure 

We will have everything in the same repository. So the backend and the frontend are easier to sync.

First start with a directory, let's call it *fullstack-workshop* and hop into this directory (and init git).

```sh
mkdir fullstack-workshop
cd fullstack-workshop
git init
```

After this one, create two new cargo projects within this directory.

```
cargo new frontend
cargo new backend
```

After this one, let's create a workspace!
Add a top level Cargo.toml (in the *fullstack-workshop* directory)

```toml
[workspace]
resolver = "2"
members = [
    "frontend",
    "backend"
]
```

Perfect! Now the core setup stands.
If you run `cargo build` both, the frontend and your backend crate should be build.

## Install Dioxus CLI

```sh
cargo install dioxus-cli
```

If you had problems with the setup, you can just copy the code from [here](https://github.com/rust-basel/workshop-2/tree/main/setup/fullstack-workshop)
