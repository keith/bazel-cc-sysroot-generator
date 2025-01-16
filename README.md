# bazel-cc-sysroot-generator

This is a CLI for generating Ubuntu and macOS hermetic sysroots for use
with bazel.

## Usage

### Prerequisites

Please make sure you have at least Python 3.12, `tar`, `zstd`, and `xz-utils` installed.

### Generate sysroots

Create a config file named `sysroot-config.toml` (or `.json`) to define
your sysroots:

```toml
[[platforms]]
os = "jammy"
archs = ["aarch64", "x86_64"]
packages = [
  "libstdc++-11-dev",
  "libstdc++6",
]
deleted_patterns = [
  "etc",
  "usr/bin",
]

[[platforms]]
os = "macos"
deleted_patterns = [
  "*Swift*",
  "*iOSSupport*",
  "usr/share",
  "usr/libexec",
]
```

Run `./bazel-cc-sysroot-generator` (optionally passing `--config
path/to/sysroot-config.toml`).

This example config generates 3 sysroots in the current directory.

> [!NOTE]
> macOS sysroots can only be generated on macOS and copy the
> currently selected SDK directory.

### Include in bazel

Once you have generated the sysroots, you can upload them somewhere and
reference them in bazel with something like:

```bzl
# MODULE.bazel
bazel_dep(name = "sysroot-jammy-x86_64")
archive_override(
    module_name = "sysroot-jammy-x86_64",
    integrity = "...",
    urls = [
        "https://...",
    ],
)
```

And use them with a hermetic CC toolchain such as
[toolchains_llvm](https://github.com/bazel-contrib/toolchains_llvm) with
something like:

```bzl
# MODULE.bazel
llvm = use_extension("@toolchains_llvm//toolchain/extensions:llvm.bzl", "llvm")
llvm.toolchain(
    name = "llvm_toolchain",
    llvm_versions = {"": "19.1.9"},
    stdlib = {"": "dynamic-stdc++"},
)

llvm.sysroot(
    name = "llvm_toolchain",
    label = "@sysroot-jammy-x86_64",
    targets = ["linux-x86_64"],
)
use_repo(llvm, "llvm_toolchain")

register_toolchains("@llvm_toolchain//:all")
```

Optionally you can disable bazel's builtin CC toolchain to make sure
it's an error if you accidentally use it by adding this to your
`.bazelrc`:

```
common --repo_env=BAZEL_DO_NOT_DETECT_CPP_TOOLCHAIN=1
```

> [!TIP]
> While iterating on your required packages, you can use
> `--override_module=sysroot-jammy-x86_64=path/to/sysroot-jammy-x86_64`
