---
layout: post
title: "Rust: How to build a Docker image with private Cargo dependencies"
category: blog
date: 2019-05-01 22:00
author: cameron
tags: [rust, docker, git, cargo, github]
---

[Rust](https://www.rust-lang.org/) is growing in popularity, having earned the most loved language in [Stack Overflow's 2019 Developer Survey](https://insights.stackoverflow.com/survey/2019#most-loved-dreaded-and-wanted) for the fourth year in a row. While [Cargo](https://doc.rust-lang.org/cargo/) and Rust's tooling are great, there's still a few tricks in getting them to work in a production setting

In this article, I'm going to show you how to fetch private Cargo dependencies and source them when building a Docker image. This solves a key issue with Docker of not copying over SSH keys when building an image

This is the first of two blog posts on building Docker images for Rust applications. The second article will discuss techniques on improving and optimizing build speeds

The code for this blog post is at <https://github.com/c-ameron/rocket-add>

## Example Application: Rocket-Add

I've developed an example API called [Rocket-Add](https://github.com/c-ameron/rocket-add) which built on top of a very cool web framework in Rust called [Rocket](https://github.com/SergioBenitez/Rocket/)

This application extends the 'Hello World' example with a new api call `add/<num1>/<num2>` which adds two numbers and outputs the result

``` bash
$ curl localhost:8000/add/5/10
The sum of 5 and 10 is 15
```

The API endpoint is written as a function
``` rust
#[get("/add/<num1>/<num2>")]
fn add(num1: i32, num2: i32) -> String {
    let s: String = math_utils::add(num1, num2).to_string();
    format!("The sum of {} and {} is {}", num1, num2, s)
}
```

Looking closely, you can see the addition calls an external library `math_utils` with the call `math_utils::add(num1, num2)`

The `math_utils` crate is from a private Github repository `https://github.com/c-ameron/rust-math-utils.git`

For reference, this is the code for the `add` function in `math_utils` being used
``` rust
pub fn add(num1: i32, num2: i32) -> i32 {
    num1 + num2
}
```

## Fetching crates in private repositories

Before we go any further, we're going to take a page from Ruby Bundler's book and create a local `.cargo` folder in the project root. This will provide a project independent place to store our crates and Cargo config

``` bash
git clone git@github.com:c-ameron/rocket-add.git
cd rocket-add
mkdir -p $(git rev-parse --show-toplevel)/.cargo
export CARGO_HOME=$(git rev-parse --show-toplevel)/.cargo
```

Currently Cargo by default won't [fetch from private repositories](https://github.com/rust-lang/cargo/issues/1851)

To get around this, create a `.cargo/config` file which tells Cargo to use the [git cli for fetching](https://doc.rust-lang.org/nightly/cargo/reference/config.html#configuration-keys)
```
$ cat .cargo/config
[net]
git-fetch-with-cli = true
```

Then in our `Cargo.toml` we tell Cargo to fetch our crate with SSH

``` toml
math_utils = { version = "0.1.0", git = "ssh://git@github.com/c-ameron/rust-math-utils.git"}
```

Now we can run `cargo fetch` and it will download all the crates to the local `.cargo` folder

## Building a Rust Docker image with private dependencies

A key problem when building a Docker image is downloading private dependencies without mounting an SSH key

To solve this problem, we can copy over our pre-fetched `.cargo` folder to the Docker image and build from there. Cargo won't need to use any SSH keys to fetch from private repositories

Note that we're having to set the `WORKDIR` and `CARGO_HOME` inside the Docker image for Cargo to resolve its crates correctly

``` Dockerfile
FROM rustlang/rust:nightly AS builder
WORKDIR /workdir                       
ENV CARGO_HOME=/workdir/.cargo                       
COPY ./Cargo.toml ./Cargo.lock ./                       
COPY ./.cargo ./.cargo
COPY ./src ./src
RUN cargo +nightly build --release
```

We can also take advantage of [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) in Docker, and create a smaller binary image

``` Dockerfile
FROM rustlang/rust:nightly AS builder
WORKDIR /workdir                       
ENV CARGO_HOME=/workdir/.cargo                       
COPY ./Cargo.toml ./Cargo.lock ./                       
COPY ./.cargo ./.cargo
COPY ./src ./src
RUN cargo +nightly build --release

FROM debian:stretch-slim
EXPOSE 8000
COPY --from=0 /workdir/target/release/rocket-add /usr/local/bin
ENTRYPOINT ["/usr/local/bin/rocket-add"]
```

This approach saves time downloading the same crates and we also don't need to pass any SSH keys to Docker. However, if there's any change to our dependencies it will require a rebuild. For a larger application with a lot of changing dependencies, this often means rebuilding from scratch several times a day

In the second blog post, I will be showing approaches to improve and optimize these Docker build speeds


## Notes

 - Docker has implemented experimental support for SSH forwarding for building images in [18.09](https://docs.docker.com/develop/develop-images/build_enhancements/#using-ssh-to-access-private-data-in-builds), potentially negating the need for copying `.cargo` in the future
 - If you want to fetch or build an image completely offline with Cargo, you can use the `-Z offline` flag. See <https://doc.rust-lang.org/cargo/reference/unstable.html#offline-mode>
 - With recently released Rust 1.34, Cargo now has support for [alternative registries](https://blog.rust-lang.org/2019/04/11/Rust-1.34.0.html#alternative-cargo-registries) other than `crates.io`
