# chromium-linux-arm64

This repository provides instructions on how to build Chromium on arm64 linux systems, 
since google does not officially support Chromium on linux-arm64 :(

# Fetching the Source Code

**Since gclient expects nacl and reclient to be fetched on a amd64 system, we will need to manually override this after most of the code is fetched.**

First, fetch most of the chromium source code.

```shell
fetch --nohooks --no-history chromium
```

This should fail torwards the end of the fetching process, because reclient isn't available on arm64. 
So let's update the `src/.gclient` file with the following contents:

```
solutions = [
  {
    "name": "src",
    "url": "https://chromium.googlesource.com/chromium/src.git",
    "managed": False,
    "custom_deps": {},
    "custom_vars": {
      "checkout_nacl": False
    },
  },
]
target_os = ['linux']
```

Then, cd into the src directory and apply the patch file to disable fetching reclient.

```
cd src
git apply ../deps.patch
```

Now you can run `gclient sync -j$(nproc)` to finish fetching the rest of the dependendencies.

Unfortunately, chromium also downloads prebuilt node.js binaries for the wrong architecture. Let's override this, assuming you have node.js installed globally on your system.

```
rm third_party/node/linux/node-linux-x64/bin/node
ln -s $(which node) third_party/node/linux/node-linux-x64/bin/node
```

## Preparing For Compilation

You need to install rustup with rust nightly and bindgen with static libclang.a. This may require building LLVM from source using `-DLIBCLANG_BUILD_STATIC=ON`, as most distros do not ship libclang.a.
This is required because gn does not pass through `LIBCLANG_PATH` to the rest of the build process.

```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
cargo install bindgen-cli --no-default-features --features static
```

Unfortunately, depot tools expects a amd64 architecture for the prebuilt gn executable.
So you need to have gn installed globally.
Now you can run `<gn> args out/Default`, where `<gn>` is the path to your globally installed gn executable.
Insert the following arguments, subsituting `src/llvm-builds/install` with your llvm/clang installation directory and `rustc_version` with the output of `rustc -V`:

> `treat_warnings_as_errors=false` is needed because harfbuzz-ng fails to build with Werror and the latest clang built from the main branch of llvm.

```
# Set build arguments here. See `gn help buildargs`.
enable_nacl=false
symbol_level=1
blink_symbol_level=0
is_clang=true
is_debug = false
is_component_build = true
clang_base_path = getenv("HOME") + "/src/llvm-builds/install"
clang_use_chrome_plugins = false
llvm_force_head_revision = true
enable_rust=true
rust_sysroot_absolute = getenv("HOME") + "/.rustup/toolchains/nightly-aarch64-unknown-linux-gnu"
rust_bindgen_root = getenv("HOME") + "/.cargo"
rustc_version="rustc 1.84.0-nightly (1e4f10ba6 2024-10-29)"
treat_warnings_as_errors=false
```

## Building

```
gn gen --export-compile-commands --export-rust-project out/Default
ninja -C oout/Default
```

## Relevant Issues

- https://issues.chromium.org/issues/40277110

