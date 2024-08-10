---
title: 'Feedback on `cargo-info` to prepare it for merging'
layout: post

categories: post
tags:
- Rust
- Cargo
- CLI
---

# Reminder

I am working on merging the `cargo-info` subcommand into Cargo. There are a few points that need to be addressed before merging. I would like to get feedback on these points.
You can find the key points of concern at the end of this post.

# Background

This adds a new subcommand to Cargo, `cargo info`. This subcommand would allow users to get information about a crate from the command line, without having to go to the web.

The main motivation for this is to make it easier to get information about a crate from the command line. Currently, the way to get information about a crate is to go to the web and look it up on [crates.io] or find the crate's source code and look at the `Cargo.toml` file. This is not very convenient, especially not all information is displayed on the [crates.io] page.
This command also has been requested by the community for a long time. You can find more discussion about this in [cargo#948].

Another motivation is to make the workflow of finding and evaluating crates more efficient. In the current workflow, users can search for crates using `cargo search`, but then they have to go to the web to get more information about the crate. This is not very efficient, especially if the user is just trying to get a quick overview of the crate. This would allow users to quickly get information about a crate without having to leave the terminal.

[crates.io]: https://crates.io
[cargo#948]: https://github.com/rust-lang/cargo/issues/948

Example usage:

```console
$ cargo info clap
    Updating crates.io index
clap #argument #cli #arg #parser #parse
A simple to use, efficient, and full-featured Command Line Argument Parser
version: 4.4.18
license: MIT OR Apache-2.0
rust-version: 1.70.0
documentation: https://docs.rs/clap/4.4.18
repository: https://github.com/clap-rs/clap
features:
  default         = [std, color, help, usage, error-context, suggestions]
  std             = [clap_builder/std]
  color           = [clap_builder/color]
  help            = [clap_builder/help]
  usage           = [clap_builder/usage]
  error-context   = [clap_builder/error-context]
  suggestions     = [clap_builder/suggestions]
  cargo           = [clap_builder/cargo]
  debug           = [clap_builder/debug, clap_derive?/debug]
  deprecated      = [clap_builder/deprecated, clap_derive?/deprecated]
  derive          = [dep:clap_derive]
  env             = [clap_builder/env]
  string          = [clap_builder/string]
  unicode         = [clap_builder/unicode]
  unstable-doc    = [clap_builder/unstable-doc, derive]
  unstable-styles = [clap_builder/unstable-styles]
  unstable-v5     = [clap_builder/unstable-v5, clap_derive?/unstable-v5, deprecated]
  wrap_help       = [clap_builder/wrap_help]
dependencies:
  clap_builder@=4.4.18
  clap_derive@=4.4.7
owners:
  kbknapp (Kevin K.)
  github:rust-cli:maintainers (Maintainers)
  github:clap-rs:admins (Admins)
```

## Detailed design

| Content                                                                    | Explanation                         | Why                                                                               |
| -------------------------------------------------------------------------- | ----------------------------------- | --------------------------------------------------------------------------------- |
| clap                                                                       | Name                                | The basic information.                                                            |
| #argument #cli #arg #parser #parse                                         | Keywords                            | It's more like a category, which you can use to search for relevant alternatives. |
| A simple to use, efficient, and full-featured Command Line Argument Parser | Description                         | The basic information.                                                            |
| version: 4.4.18                                                            | Version                             | The basic information.                                                            |
| license: MIT OR Apache-2.0                                                 | License                             | When choosing a crate, it is crucial to consider the license.                     |
| rust-version: 1.70.0                                                       | MSRV                                | When choosing a crate, it is crucial to make sure it can work with your MSRV.     |
| documentation: <https://docs.rs/clap/4.4.18>                               | Documentation Link                  | Use these links can find more docs and information.                               |
| repository: <https://github.com/clap-rs/clap>                              | Repo Link                           | Use these links can find more docs and information.                               |
| features:                                                                  | Default Features And Other Features | It helps for enabling features.                                                   |
| dependencies:                                                              | All dependencies                    | It indicates what it depends on.                                                  |
| owners:                                                                    | Owners                              | It indicates who maintains the crate.                                             |

# Some important notes

## Downloading the crate from any Cargo compatible registry

The `cargo info` command will download the crate from any Cargo compatible registry. It will then extract the information from the `Cargo.toml` file and display it in the terminal.

If the crate is already in the local cache, it will not download the crate again. It will get the information from the local cache.

## Pick the correct version from the workspace

When executed in a workspace directory, the cargo info command chooses the version that the workspace is currently using.

If there's a lock file available, the version from this file will be used. In the absence of a lock file, the command attempts to select a version that is compatible with the Minimum Supported Rust Version (MSRV). And the lock file will be generated automatically.

The following hierarchy is used to determine the MSRV:

- First, the MSRV of the parent directory package is checked, if it exists.
- If the parent directory package does not specify an MSRV, the minimal MSRV of the workspace is checked.
- If neither the workspace nor the parent directory package specify an MSRV, the version of the current Rust compiler (rustc --version) is used.

# Prior art

## NPM

[npm] has a similar command called `npm info`.
For example:

```console
$ npm info lodash

lodash@4.17.21 | MIT | deps: none | versions: 114
Lodash modular utilities.
https://lodash.com/

keywords: modules, stdlib, util

dist
.tarball: https://registry.npmjs.org/lodash/-/lodash-4.17.21.tgz
.shasum: 679591c564c3bffaae8454cf0b3df370c3d6911c
.integrity: sha512-v2kDEe57lecTulaDIuNTPy3Ry4gLGJ6Z1O3vE1krgXZNrsQ+LFTGHVxVjcXPs17LhbZVGedAJv8XZ1tvj5FvSg==
.unpackedSize: 1.4 MB

maintainers:
- mathias <mathias@qiwi.be>
- jdalton <john.david.dalton@gmail.com>
- bnjmnt4n <benjamin@dev.ofcr.se>

dist-tags:
latest: 4.17.21

published over a year ago by bnjmnt4n <benjamin@dev.ofcr.se>
```

[npm]: https://www.npmjs.com/

## Poetry

[Poetry] has a similar command called `poetry show`.

For example:

```console
$ poetry show pendulum

name        : pendulum
version     : 1.4.2
description : Python datetimes made easy

dependencies
 - python-dateutil >=2.6.1
 - tzlocal >=1.4
 - pytzdata >=2017.2.2

required by
 - calendar >=1.4.0
```

[Poetry]: https://python-poetry.org/

# Known Points of Concern

1. Report crates metrics? [cargo-information#20]

    Proposal:

    - recent download count (popularity)
    - last updated (give a feel for how active development is)
    - Only for crates.io.

2. What dependency fields might be relevant to indicate? [cargo-information#23]

    Proposal:

    - From @epage: Dependencies are mostly an implementation detail (except public) but people sometimes care, so I figure that holding off on private dependencies to --verbose might buy us more space.
    - From @Rustin170506: How about we show all the dependencies and only show the dev-dependencies and build-dependencies for a --verbose. I guess checking its dependencies before you use it in your project would always be considered.

3. How should we render features? [cargo-information#26]

    Proposal: Currently, it's a simple list of features and their dependencies. We could consider a tree view:

    ```console
    features:
      parent1
        child1
      parent2
        child1*
    ```

4. What version should we default to within a workspace? What if it isn't the direct dependency but is a transitive dependency? [cargo-information#29]

[cargo-information#20]: https://github.com/Rustin170506/cargo-information/issues/20
[cargo-information#23]: https://github.com/Rustin170506/cargo-information/issues/23
[cargo-information#26]: https://github.com/Rustin170506/cargo-information/issues/26
[cargo-information#29]: https://github.com/Rustin170506/cargo-information/issues/29
