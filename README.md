# cargo xtask

cargo-xtask is way to add free-form automation to a Rust project, a-la `make`, `npm run` or bespoke bash scripts.

The two distinguishing features of xtask are:

* It doesn't require any other binaries besides `cargo` and `rustc`, it fully bootstraps from them
* Unlike bash, it can more easily be cross platform, as it doesn't use the shell.

## How Does it Work?

cargo-xtask is a polyfill for [cargo workflows](http://aturon.github.io/tech/2018/04/05/workflows/) feature.
It is a way to extend stock, stable cargo with custom commands (`xtasks`), written in Rust.

This polyfill doesn't need any code, just a particular configuration of a cargo project.
This repository serves as a specification of such configuration.

## Status

cargo-xtask is not an officially recommended workflow, but it is a somewhat common pattern across
the ecosystem. Notably, Cargo itself
[uses xtasks](https://github.com/rust-lang/cargo/blob/0.78.0/.cargo/config.toml#L2-L4).

It might or might not work for your use case!

## Defining xtasks

The best way to create an xtask is to do so inside of a Cargo workspace. If you don't have a workspace already,
you can create one inside your package by moving the contents into a new directory. Let's say that our package
is named "testing." We first move everything into a sub-directory:

```console
$ mkdir testing

# then move all of the stuff except your .git directory into the new testing directory:
$ mv src testing
$ mv Cargo.toml testing
$ mv .gitignore testing
$ mv README.md testing

# Don't forget anything else your package may have.
```

Then, add a new package named `xtask`:

```console
$ cargo new --bin xtask
```

Then, we need to create a `Cargo.toml` for our workspace:

```toml
[workspace]
members = [
    "testing",
    "xtask",
]
```

If you had a workspace previously, you'd add `xtask` to your existing workspace `Cargo.toml`.

Then, the alias. **This is where the magic happens**. Create a `.cargo`:

```console
$ mkdir .cargo
```

and create a file in it named `config.toml` with these contents:

```toml
[alias]
xtask = "run --package xtask --"
```


Example directory layout:

```
/testing
  .git
  .cargo/
    config.toml
  Cargo.toml
  testing/
    Cargo.toml
    .gitignore
    src/
      lib.rs
  xtask/
    Cargo.toml
    src/
      main.rs
```

Both the `xtask` directory and the `.cargo/config` should be committed to the version control system.

If you don't want to use a workspace, you can use `run --manifest-path ./xtask/Cargo.toml --` for the alias, but this is not recommended.

The `xtask` binary should expect at least one positional argument, which is a name of the task to be executed.
Tasks are implemented in Rust, and can use arbitrary crates from crates.io.
Tasks can execute `cargo` (it is advisable to use `CARGO` environmental variable to get the right `cargo`).

The `xtask` crate may or may not be a part of the main workspace. Usually, but not always, the workspace setup is better.
If `xtask` is a part of the workspace, you can share dependencies between `xtask` and main crates, and dependencies update process is easier.
Additionally, you will be able to use `xtask = "run --package xtask --"` as an alias, which works regardless of Cargo's working directory.
If `xtask` is not a part of the workspace, you can use different feature sets for shared dependencies, and you can cache `xtask/target` more easily on CI.
It is advisable to commit `xtask` lockfile to the repository.

It is advisable to minimize the compile time of xtasks.

You can find some examples of xtasks in the [`./examples`](https://github.com/matklad/cargo-xtask/blob/master/examples) directory in this repository.

The current recommendation is to define various task as subcommands of the single `xtask` binary.
An alternative is to use a separate binary and a separate entry in `.cargo/config` for each task.

## External examples

- [rust-analyzer](https://github.com/rust-lang/rust-analyzer/tree/master/xtask): Releasing, performance metrics, and much more.
- [helix-editor/helix](https://github.com/helix-editor/helix/tree/master/xtask): Validating embedded query files, generating docs.
- [containers/bootc](https://github.com/containers/bootc/tree/main/xtask): Generates RPMs, custom lints for `dbg!`.
- [rust-lang/cargo](https://github.com/rust-lang/cargo/blob/e5e68c4093af9de3f80e9427b979fa5a0d8361cc/.cargo/config.toml#L1-L4): These days, even Cargo itself uses this pattern!

And many more examples can be found via e.g. [Github Code search](https://cs.github.com/?scopeName=All+repos&scope=&q=lang%3Arust+path%3Axtask).

## Limitations

xtasks do not integrate with Cargo lifecycle.
If you need to do custom post-processing after `cargo build`, you'll need to define and call `cargo xtask build` task, which calls `cargo build` internally.
There's no way to intercept stock `cargo build` command.

It's impossible to use xtasks from dependencies, xtasks are project-local.
However, it is possible to share logic for implementing common xtasks as crates.io packages.

If `xtask` is not a workspace member, `cargo xtask` will work only from the project's root directory.

## Using xtasks

Use `cargo xtask task-name` command to run the task.

Example:

```bash
cargo xtask deploy
```

Note that this doesn't require any additional setup besides cloning the repository, and will automatically build the `xtask` binary on the first run.

## Not Using xtasks

xtasks are entirely optional, and you don't have to use them!
In particular, if, for your purposes, `cargo build` and `cargo test` are enough, don't use xtasks.
If you prefer to write a short bash script, and don't need to support windows, there's no need to use xtasks either.

## Standard xtasks

In theory, it might be beneficial to specify a convention for some common tasks to enable high order
tooling. For example, if many CLI applications can provide `cargo xtask dist` which builds a
distribution of the application in questions (compiled binary + man pages + shell completions + any
other auxiliary resources), that could be re-used by downstream Linux distributions to package Rust
applications in a uniform way.

To my knowledge, no such conventional xtasks emerged so far. If you think it will be a good idea to
codify some repeating patterns, consider publishing your own specification (e.g., create
`cargo-xtask-dist` repository). If it catches up (used by at least three different projects by
different authors), please send a PR to this document with a link to the spec.

See [#1](https://github.com/matklad/cargo-xtask/issues/1) for discussion.

## Tooling

Libraries:
- [devx](https://github.com/elastio/devx): collection of useful utilities (spawning processes, git pre-commit hooks, etc.)
- [xshell](https://github.com/matklad/xshell): ergonomic "bash" scripting in Rust
- [duct](https://github.com/oconnor663/duct.rs): a library for running child processes with support for pipelines and IO redirection

If you write tools or libraries for xtasks, send a PR to this document.
Some possible ideas:

* cargo subcomand to generate `xtask` template
* implementations of common xtasks, like "check that code is formatted with rustfmt" or "build completions for a clap app", as libraries.

## Background

To my knowledge, the idea of xtasks was first introduced in [this post](https://matklad.github.io/2018/01/03/make-your-own-make.html).
In some sense, the present document just specifies some conventions around original idea.

The name `xtask` is chosen so as not to conflict with potential future built-in cargo feature for tasks.
