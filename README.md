# rust-bpf-sysroot

[![Build Status](https://travis-ci.org/Cartallum/rust-bpf-sysroot.svg?branch=master)](https://travis-ci.org/Cartallum/rust-bpf-sysroot)

Rust sysroot source for Berkley Packet Filter Rust programs

Contains submodules, to sync use:

``` bash
git clone --recurse-submodules
```

---

Building Rust modules require a collection of standard libraries that
provide the fundamentals of the Rust language.  These standard
libraries include things like types and trait definitions, arithmetic
operations, formatting definitions, structure definitions (slice, vec)
and operations.  Typically Rust modules link in the `std` libraries
which expose the underlying operating system features like threads,
display and file io, networking, etc.

Rust modules for Cartallum CBE are built using the Berkley Packet Filter
(BPF) ABI and run within a limited virtual machine that will not
provide most of the native OS features.  One reason is that program
state must be recordable in the ledger with known inputs and outputs.
Things like files provide an untraceable input to the ledger,
multi-threading can lead to timing differences that may result in
unpredictable program output based on the same inputs.
[Rust-cross](https://github.com/japaric/rust-cross) is a good overview
of cross-compiling Rust.

This repo contains only the pieces of the Rust libraries required by
Cartallum CBE modules, and in some cases, these pieces might include
customizations required by either Cartallum CBE or to be compatible with the
BPF ABI.  It is the goal of this repo to be temporary, as support is
added to Cartallum CBE and BPF for things like unsigned division, 128-bit
types, etc. Cartallum CBE should be able to refer to the libraries in the
Rust mainline eventually.

The Cartallum CBE SDK pulls this repo in as source to make it available to
[xargo](https://github.com/japaric/xargo).  Xargo then builds and uses
it as the cargo sysroot for Cartallum CBE modules.

You can build this repo independently of the Cartallum CBE SDK in the same
way that CI ensures the repo stays healthy.  The build script
downloads Cartallum CBE's custom [rustc and
clang](https://github.com/Cartallum/bpf-tools) binaries and updates
the forked submodules.  Take a look at
[`test/build.sh`](https://github.com/dmakarov/rust-bpf-sysroot/blob/master/test/build.sh)
for details.

To build the test:
``` bash
./test/build.sh
```

Notes:
- If building on Linux, ensure you are using Ubuntu 18 or newer since
  Cartallum CBE's custom rustc is not compatible with older versions.
- `src/lib.rs` is only provided to enable building this repo
  independently of the Cartallum CBE SDK; it is not built as part of the
  sysroot by Xargo.

Upgrading Cartallum CBE BPF toolchain
--------------------------------------------

Rust-bpf-sysroot is an essential part of Cartallum CBE Rust/Clang/LLVM BPF
toolchain. Whenever the toolchain is upraded to a new version of
rust/clang/llvm, rust-bpf-sysroot must be upgraded to match the
changes in the Rust/Clang/LLVM compilers. The following is an outline
and checklist of the upgrade process

1. Upgrade the compilers

    - choose the version of
      [rust-lang/rust](https://github.com/rust-lang/rust/tags) to upgrade
      the toolchain to.
    - upgrade
      [Cartallum/llvm-project](https://github.com/Cartallum/llvm-project)
      to the version of
      [rust-lang/llvm-project](https://github.com/rust-lang/llvm-project)
      that corresponds to the selected rust-lang/rust version.
    - upgrade [Cartallum/rust](https://github.com/Cartallum/rust) to
      the chosen version of
      [rust-lang/rust](https://github.com/rust-lang/rust).
    - build the compiler binaries and keep them available.

2. Upgrade rust-bpf-sysroot submodules

    rust-bpf-sysroot includes 4 submodules
    - Cartallum CBE forks
      - [Cartallum/cfg-if](https://github.com/Cartallum/cfg-if) of [alexcrichton/cfg-if](https://github.com/alexcrichton/cfg-if)
      - [Cartallum/compiler-builtins](https://github.com/Cartallum/compiler-builtins)
        of [rust-lang/compiler-builtins](https://github.com/rust-lang/compiler-builtins)
      - [Cartallum/hashbrown](https://github.com/Cartallum/hashbrown)
        of [rust-lang/hashbrown](https://github.com/rust-lang/hashbrown)
    - [rust-lang/stdarch](https://github.com/rust-lang/stdarch)

    Check which version of each submodule is used by the chosen
    version of [rust-lang/rust](https://github.com/rust-lang/rust) and
    update Cartallum CBE's forks and bump the version of
    [rust-lang/stdarch](https://github.com/rust-lang/stdarch) in
    [`src`](https://github.com/Cartallum/rust-bpf-sysroot/tree/master/src)
    subdirectory. The versions required by rust-lang/rust can be
    checked in
    [rust-lang/rust/libraries/std/Cargo.toml](https://github.com/rust-lang/rust/blob/master/library/std/Cargo.toml).

    If a Cartallum CBE fork submodule is updated it is better to postpone
    committing the updated submodule to its Cartallum CBE repository until
    the upgrade of
    [Cartallum/rust-bpf-sysroot](https://github.com/Cartallum/rust-bpf-sysroot)
    is finalized. In `Cartallum/rust-bpf-sysroot/src/<submodule>`
    pull from your fork of the submodule the branch that contains the
    version of the submodule with the Cartallum CBE specific changes. When
    the updates to rust-bpf-sysroot are finalized the changes to the
    submodules must be committed to their corresponding Cartallum
    repositories.

3. Upgrade rust-bpf-sysroot

   - pull the latest master of
     [Cartallum/rust-bpf-sysroot](https://github.com/Cartallum/rust-bpf-sysroot)
   - copy the subdirectories of `Cartallum/rust-bpf-sysroot/src`
     which are not submodules from the corresponding subidrectories of
     updated `Cartallum/rust/libraries`, overwriting the contents of
     these subdirectories. The directories are
     - `alloc`
     - `core`
     - `panic_abort`
     - `rustc-std-workspace-alloc`
     - `rustc-std-workspace-core`
     - `std`
     - `unwind`
     - `compiler-rt` is copied from `Cartallum/llvm-project/compiler-rt`.
   - commit the changes with the commit message "_Pull in Rust 1.XX
     changes_" where _XX_ is the chosen version of rust-lang/rust. Note
     the committed changes should be only what was copied from
     `Cartallum/rust/libraries` and
     `Cartallum/llvm-project/compiler-rt`. Thus we can keep the
     local changes in separate commits, which should make subsequents
     upgrades manageable.
   - cherry-pick the commits starting from the commit following the
     previous commit with the commit message "_Pull in Rust 1.XX
     changes_" and reapply them on top of the just committed new _Pull
     in Rust 1.XX changes_ commit. Note, that some commits in the
     history will not have changes in the libraries source files. Such
     commits must not be cherry-picked and applied. To make this
     process manageable, commits must never mix changes to files in
     libraries with any other changes. The description line of commits
     that modify libraries files should have the prefix _[SOL]_ and
     other commits should not have such prefix to clearly distinguish
     betweeen the commits that need to be cherry-picked.
   - after reapplying all Cartallum CBE specific changes on top of the
     updated libraries source files, start building the source tree,
     by running the script `./test/build.sh`. Make sure to build the
     tree using the updated compiler binaries from the step 1. An easy
     way to use a custom compiler binaries is to create a subdirectory
     `bpf-tools` in the directory `rust-bpf-sysroot/test/dependencies`
     and in `bpf-tools` create two symbolic links, e.g.
     ``` bash
     ln -s <path to Cartallum/rust>/build/x86_64-apple-darwin/llvm llvm
     ln -s <path to Cartallum/rust>/build/x86_64-apple-darwin/stage1 rust
     ```
     Fix any build errors, and compiler warnings.
   - build and run `Cartallum/cbe/programs/bpf` using the new
     rust-bpf-sysroot and the new rust/clang compilers. To use the new
     rust-bpf-sysroot redirect the symbolic link `rust-bpf-sysroot` in
     `<path to Cartallum/cbe>/sdk/bpf/dependencies/` to `<path to
     Cartallum/rust-bpf-sysroot>`. When all tests build and run
     successfully
        - commit updated submodules to their corresponding repositories,
        - commit changes that had to be done in libraries source files
          with the description line prefixed with _[SOL]_ tag,
        - make a new release branch of rust-bpf-sysroot,
        - make a new release of
          [Cartallum/bpf-tools](https://github.com/Cartallum/bpf-tools)
          that contains the tarball packages with the new compiler binaries.
   - update
     [`Cartallum/cbe/sdk/bpf/scripts/install.sh`](https://github.com/Cartallum/cbe/blob/master/sdk/bpf/scripts/install.sh)
     to install the new version of compiler binaries and
     rust-bpf-sysroot source tree for the Cartallum CBE SDK. Other files that
     may have to be updated are
     - `Cartallum/cbe/sdk/bpf/env.sh`
     - `Cartallum/cbe/sdk/bpf/scripts/{dump.sh,objcopy.sh,strip.sh}`
     - `Cartallum/cbe/sdk/bpd/c/{bpf.ld,bpf.mk}`
     - `Cartallum/cbe/sdk/bpd/rust/bpf.ld`
   - update this file with any corrections and changes to the upgrade
     process.
