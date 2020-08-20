# cargo xtask

cargo-xtask is way to add free-form automation to a Rust project, a-la `make`, `npm run` or bespoke bash scripts.

The two distinguishing features of xtask are:

* It doesn't require any other binaries besides `cargo` and `rustc`, it fully bootstraps from them
* Unlike bash, it can more easily be cross platform, as it doesn't use the shell.

## Status

cargo-xtask is neither an officially recommended workflow, nor a de-facto standard (yet?).
It might or might not work for your use case.
Moreover, xtask is new, so expect changes!

## How Does It Work?

cargo-xtask is a polyfill for [cargo workflows](http://aturon.github.io/tech/2018/04/05/workflows/) feature.
It is a way to extend stock, stable cargo with custom commands (`xtasks`), written in Rust.

This polyfill doesn't need any code, just a particular configuration of a cargo project.
This repository serves as a specification of such configuration.

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

Then, the alias. This is where the magic happens. Create a `.cargo`:

```console
$ mdkir .cargo
```

and create a file in it named `config` with these contents:

```toml
[alias]
xtask = "run --package xtask --"
```


Example directory layout:

```
/testing
  .git
  .cargo/
    config
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
Additionally, you will be able to use `xtask = "run --package xtask --"` as an alias, which works regardless of Cargo's working directory
If `xtask` is not a part of the workspace, you can use different feature sets for shared dependencies, and you can cache `xtask/target` more easily on CI.
It is advisable to commit `xtask` lockfile to the repository.

It is advisable to minimize the compile time of xtasks.

You can find some examples of xtasks in the [`./examples`](https://github.com/matklad/cargo-xtask/blob/master/examples) directory in this repository.

The current recommendation is to define various task as subcommands of the single `xtask` binary.
An alternative is to use a separate binary and a separate entry in `.cargo/config` for each task.

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

The following specifies the names and behaviors of some common xtasks, to help establish common conventions.
If you want to tweak behavior of a standard task for your project, you can add custom flags to it.
If you feel an important common task is missing, feel free to submit a PR!

### `cargo xtask`, `cargo xtask --help`

When run without argument or with the `--help` argument, `xtask` should print a help message which lists available tasks.

### `cargo xtask dist`

This should *package* the software and produce a set of distributable artifacts.
Artifacts should be placed into `./target/dist` directory.
The precise meaning of artifacts is not defined, but, for a CLI tool, you can expect the binary itself (build in release mode and stripped), man pages and shell completion files.
The `dist` command should clean the `./target/dist` directory before populating it with artifacts.
It is expected that the `dist` command calls `cargo build --release` internally.

See [#3](https://github.com/matklad/cargo-xtask/issues/3) for additional discussion.

### `cargo xtask codegen`

This command should run code generation, which happens outside of `build.rs`.
For example, if you are writing a gPRC server, and would like to commit the generated code into the repository (so that the clients don't have to have `protoc` installed), you can implement code generation as `cargo xtask codegen`.

### `cargo xtask ci`

This task should run `cargo test` and any additional checks that are required on CI, like checking formatting, running `miri` test, checking links in the documentation.
The CI configuration should generally look like this:

```yaml
script:
  - cargo xtask ci
```

The expectation is that, if `cargo xtask ci` passes locally, the CI will be green as well.

You don't need this task if `cargo test` is enough for your purposes.
Moreover, there are certain tradeoffs associated with using xtasks instead of CI provider's built-in ways to specify CI process.
So, we do not recommend to blindly use `xtask ci` over `.travis.yml`, but, if you want to use xtasks for CI, use `ci` as the name of the task.

See [#1](https://github.com/matklad/cargo-xtask/issues/1) for discussion.

## Tooling

Libraries:
- [devx](https://github.com/elastio/devx): collection of useful utilities (spawning processes, git pre-commit hooks, etc.)

If you write tools or libraries for xtasks, send a PR to this document.
Some possible ideas:

* cargo subcomand to generate `xtask` template
* implementations of common xtasks, like "check that code is formatted with rustfmt" or "build completions for a clap app", as libraries.

## Background

To my knowledge, the idea of xtasks was first introduced in [this post](https://matklad.github.io/2018/01/03/make-your-own-make.html).
In some sense, the present document just specifies some conventions around original idea.

The name `xtask` is chosen so as not to conflict with potential future built-in cargo feature for tasks.
