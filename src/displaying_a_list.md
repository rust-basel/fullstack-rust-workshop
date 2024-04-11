# Displaying a list

Now we are going to connect our to parts!
We are going to display a list in the frontend, which is served by our backend.

## Makefile

But first we introduce a `Makefile.toml` which will make our life much easier. With `cargo-make`.
We can execute our three commands concurrently with one command, which is:
- servig our backend
- serving our frontend
- telling tailwind to watch files, and to recompile the css if needed.

Go ahead an create `Makefile.toml` at the top level.

```toml
[tasks.backend-dev]
install_crate = "cargo-watch"
command = "cargo"
args = ["watch", "-w", "backend", "-x", "run --bin backend"]

[tasks.frontend-dev]
install_crate = "dioxus-cli"
command = "dx"
args = ["serve", "--bin", "frontend", "--hot-reload"]

[tasks.tailwind-dev]
command = "npx"
args = ["tailwindcss", "-i",  "./frontend/input.css", "-o", "./frontend/public/tailwind.css"]

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

every stepped we talked before, is now executed within one command. With hot-reload!
