# cargo xtask

cargo-xtask is way to add free-form automation to a Rust project, a-la `make` or bespoke bash scripts.

The two distinguishing features of xtask are:

* It doesn't require any other binaries besides `cargo` and `rustc`, it fully bootstraps from them
* Unlike bash, it is cross platform

## How Does It Work?

cargo-xtask is a polyfill for [cargo workflows](http://aturon.github.io/tech/2018/04/05/workflows/) feature.
It is a way to extend stock, stable cargo with custom commands (`xtasks`), written in Rust.

This polyfill doesn't need any code, just a particular configuration of a cargo project.
This repository serves as a specification of such configuration.

## Defining xtasks

In the root of the project repository, there should be an `xtask` directory, which is a cargo crate with one binary target.
In the root of the project, there should be a `./cargo/config` file with the following entry:

```toml
[alias]
xtask = "run --manifest-path ./xtask/Cargo.toml"
```

Example directory layout:

```
/my-project
  .git
  .gitignore
  .cargo/
    config
  Cargo.toml
  src/
    lib.rs
  xtask/
    Cargo.toml
    src/
      main.rs
```

Both `xtask` directory and `./cargo/config` should be committed to the version control system.

The `xtask` binary should expect at least one positional argument, which is a name of the task to be executed.
Tasks are implemented in Rust, and can use arbitrary crates from crates.io.
Tasks can execute `cargo` (it is advisable to use `CARGO` environmental variable to get the right `cargo`).

The `xtask` crate may or may not be a part of the main workspace.

## Using xtasks

Use `cargo xtask task-name` command to run the task.

Example:
```
cargo xtask deploy
```

Note that this doesn't require any additional setup besides cloning the repository, and will automatically build the `xtask` binary on the first run.

## Not Using xtasks

xtasks are entirely optional, and you don't have to use them!
In particular, if, for your purposes, `cargo build` and `cargo test` are enough, don't use xtasks.
If you prefer to write a short bash script, and don't need to support windows, there's no need to use xtasks either.

## Standard xtasks

The following specifies the names and behaviors of some common xtasks, to help establish common conventions.
If you feel an important common task is missing, feel free to submit a PR!

### `cargo xtask`, `cargo xtask --help`

When run without argument or with the `--help` argument, `xtask` should print a help message which lists available tasks and contains the link to this specification.

### `cargo xtask ci`

This task should run `cargo test` and any additional checks that are required on CI, like checking formatting, running `miri` test, checking links in the documentation.
The CI configuration should generally look like this:

```yaml
script:
  - cargo xtask ci
```

The expectation is that, if `cargo xtask ci` passes locally, the CI will be green as well.

You don't need this task if `cargo test` is enough for your purposes.

### `cargo xtask dist`

This should *package* the software and produce a set of distributable artifacts.
Artifacts should be placed into `./target/dist` directory.
The precise meaning of artifacts is not defined, but, for a CLI tool, you can expect the binary itself (build in release mode and stripped), man pages and shell completion files.
The `dist` command should clean the `./target/dist` directory before populating it with artifacts.
It is expected that the `dist` command calls `cargo build --release` internally.

### `cargo xtask codegen`

This command should run code generation, which happens outside of `build.rs`.
For example, if you are writing a gPRC server, and would like to commit the generated code into the repository (so that the clients don't have to have `protoc` installed), you can implement code generation as `cargo xtask codegen`.

## Tooling

There's no specific tooling to support xtasks at the moment.
If you write tools or libraries for xtasks, send a PR to this document.
Some possible ideas:

* cargo subcomand to generate `xtask` template
* implementations of common xtasks, like "check that code is formatted with rustfmt" or "build completions for a clap app", as libraries.

## Background

To my knowledge, the idea of xtasks was first introduced in [this post](https://matklad.github.io/2018/01/03/make-your-own-make.html).
In some sense, the present document just specifies some conventions around original idea.

The name `xtask` is chosen so as not to conflict with potential future built-in cargo feature for tasks.
