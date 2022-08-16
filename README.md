# CosmWasm Todo-Smart-Contract

This is a template to build smart contracts in Rust to run inside a
[Cosmos SDK](https://github.com/cosmos/cosmos-sdk) module on all chains that enable it.
To understand the framework better, please read the overview in the
[cosmwasm repo](https://github.com/CosmWasm/cosmwasm/blob/master/README.md),
and dig into the [cosmwasm docs](https://www.cosmwasm.com).
This assumes you understand the theory and just want to get coding.

## Creating a new repo from template

Assuming you have a recent version of rust and cargo (v1.58.1+) installed
(via [rustup](https://rustup.rs/)),
then the following should get you a new repo to start a contract:

Install [cargo-generate](https://github.com/ashleygwilliams/cargo-generate) and cargo-run-script.
Unless you did that before, run this line now:

```sh
cargo install cargo-generate --features vendored-openssl
cargo install cargo-run-script
```

Now, use it to create your new contract.
Go to the folder in which you want to place it and run:


**Latest: 1.0.0**

```sh
cargo generate --git https://github.com/CosmWasm/cw-template.git --name PROJECT_NAME
````

For cloning minimal code repo:

```sh
cargo generate --git https://github.com/CosmWasm/cw-template.git --branch 1.0-minimal --name PROJECT_NAME
```

**Older Version**

Pass version as branch flag:

```sh
cargo generate --git https://github.com/CosmWasm/cw-template.git --branch <version> --name PROJECT_NAME
````

Example:

```sh
cargo generate --git https://github.com/CosmWasm/cw-template.git --branch 0.16 --name PROJECT_NAME
```

You will now have a new folder called `PROJECT_NAME` (I hope you changed that to something else)
containing a simple working contract and build system that you can customize.

## Create a Repo

After generating, you have a initialized local git repo, but no commits, and no remote.
Go to a server (eg. github) and create a new upstream repo (called `YOUR-GIT-URL` below).
Then run the following:

```sh
# this is needed to create a valid Cargo.lock file (see below)
cargo check
git branch -M main
git add .
git commit -m 'Initial Commit'
git remote add origin YOUR-GIT-URL
git push -u origin main
```

## CI Support

We have template configurations for both [GitHub Actions](.github/workflows/Basic.yml)
and [Circle CI](.circleci/config.yml) in the generated project, so you can
get up and running with CI right away.

One note is that the CI runs all `cargo` commands
with `--locked` to ensure it uses the exact same versions as you have locally. This also means
you must have an up-to-date `Cargo.lock` file, which is not auto-generated.
The first time you set up the project (or after adding any dep), you should ensure the
`Cargo.lock` file is updated, so the CI will test properly. This can be done simply by
running `cargo check` or `cargo unit-test`.

# Using your project
## Prerequisites

Before starting, make sure you have [rustup](https://rustup.rs/) along with a
recent `rustc` and `cargo` version installed. Currently, we are testing on 1.58.1+.

And you need to have the `wasm32-unknown-unknown` target installed as well.

You can check that via:

```sh
rustc --version
cargo --version
rustup target list --installed
# if wasm32 is not listed above, run this
rustup target add wasm32-unknown-unknown
```

## Compiling and running tests

Now that you created your custom contract, make sure you can compile and run it before
making any changes. Go into the repository and do:

```sh
# this will produce a wasm build in ./target/wasm32-unknown-unknown/release/YOUR_NAME_HERE.wasm
cargo wasm

# this runs unit tests with helpful backtraces
RUST_BACKTRACE=1 cargo unit-test

# auto-generate json schema
cargo schema
```

### Understanding the tests

The main code is in `src/contract.rs` and the unit tests there run in pure rust,
which makes them very quick to execute and give nice output on failures, especially
if you do `RUST_BACKTRACE=1 cargo unit-test`.

We consider testing critical for anything on a blockchain, and recommend to always keep
the tests up to date.

## Generating JSON Schema

While the Wasm calls (`instantiate`, `execute`, `query`) accept JSON, this is not enough
information to use it. We need to expose the schema for the expected messages to the
clients. You can generate this schema by calling `cargo schema`, which will output
4 files in `./schema`, corresponding to the 3 message types the contract accepts,
as well as the internal `State`.

These files are in standard json-schema format, which should be usable by various
client side tools, either to auto-generate codecs, or just to validate incoming
json wrt. the defined schema.

## Preparing the Wasm bytecode for production

Before we upload it to a chain, we need to ensure the smallest output size possible,
as this will be included in the body of a transaction. We also want to have a
reproducible build process, so third parties can verify that the uploaded Wasm
code did indeed come from the claimed rust code.

To solve both these issues, we have produced `rust-optimizer`, a docker image to
produce an extremely small build output in a consistent manner. The suggest way
to run it is this:

```sh
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/rust-optimizer:0.12.6
```

Or, If you're on an arm64 machine, you should use a docker image built with arm64.
```sh
docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/rust-optimizer-arm64:0.12.6
```

We must mount the contract code to `/code`. You can use a absolute path instead
of `$(pwd)` if you don't want to `cd` to the directory first. The other two
volumes are nice for speedup. Mounting `/code/target` in particular is useful
to avoid docker overwriting your local dev files with root permissions.
Note the `/code/target` cache is unique for each contract being compiled to limit
interference, while the registry cache is global.

This is rather slow compared to local compilations, especially the first compile
of a given contract. The use of the two volume caches is very useful to speed up
following compiles of the same contract.

This produces an `artifacts` directory with a `PROJECT_NAME.wasm`, as well as
`checksums.txt`, containing the Sha256 hash of the wasm file.
The wasm file is compiled deterministically (anyone else running the same
docker on the same git commit should get the identical file with the same Sha256 hash).
It is also stripped and minimized for upload to a blockchain (we will also
gzip it in the uploading process to make it even smaller).

# Importing

In [Publishing](./Publishing.md), we discussed how you can publish your contract to the world.
This looks at the flip-side, how can you use someone else's contract (which is the same
question as how they will use your contract). Let's go through the various stages.

## Verifying Artifacts

Before using remote code, you most certainly want to verify it is honest.

The simplest audit of the repo is to simply check that the artifacts in the repo
are correct. This involves recompiling the claimed source with the claimed builder
and validating that the locally compiled code (hash) matches the code hash that was
uploaded. This will verify that the source code is the correct preimage. Which allows
one to audit the original (Rust) source code, rather than looking at wasm bytecode.

We have a script to do this automatic verification steps that can
easily be run by many individuals. Please check out
[`cosmwasm-verify`](https://github.com/CosmWasm/cosmwasm-verify/blob/master/README.md)
to see a simple shell script that does all these steps and easily allows you to verify
any uploaded contract.

## Reviewing

Once you have done the quick programatic checks, it is good to give at least a quick
look through the code. A glance at `examples/schema.rs` to make sure it is outputing
all relevant structs from `contract.rs`, and also ensure `src/lib.rs` is just the
default wrapper (nothing funny going on there). After this point, we can dive into
the contract code itself. Check the flows for the execute methods, any invariants and
permission checks that should be there, and a reasonable data storage format.

You can dig into the contract as far as you want, but it is important to make sure there
are no obvious backdoors at least.

## Decentralized Verification

It's not very practical to do a deep code review on every dependency you want to use,
which is a big reason for the popularity of code audits in the blockchain world. We trust
some experts review in lieu of doing the work ourselves. But wouldn't it be nice to do this
in a decentralized manner and peer-review each other's contracts? Bringing in deeper domain
knowledge and saving fees.

Luckily, there is an amazing project called [crev](https://github.com/crev-dev/cargo-crev/blob/master/cargo-crev/README.md)
that provides `A cryptographically verifiable code review system for the cargo (Rust) package manager`.

I highly recommend that CosmWasm contract developers get set up with this. At minimum, we
can all add a review on a package that programmatically checked out that the json schemas
and wasm bytecode do match the code, and publish our claim, so we don't all rely on some
central server to say it validated this. As we go on, we can add deeper reviews on standard
packages.

If you want to use `cargo-crev`, please follow their
[getting started guide](https://github.com/crev-dev/cargo-crev/blob/master/cargo-crev/src/doc/getting_started.md)
and once you have made your own *proof repository* with at least one *trust proof*,
please make a PR to the [`cawesome-wasm`]() repo with a link to your repo and
some public name or pseudonym that people know you by. This allows people who trust you
to also reuse your proofs.

There is a [standard list of proof repos](https://github.com/crev-dev/cargo-crev/wiki/List-of-Proof-Repositories)
with some strong rust developers in there. This may cover dependencies like `serde` and `snafu`
but will not hit any CosmWasm-related modules, so we look to bootstrap a very focused
review community.

