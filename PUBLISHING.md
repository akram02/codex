# Publishing

This guide is for publishing the forked npm packages from this checkout.

It assumes the local publish patch is present so the CLI packages are named
`@akramkhan/codex` and `@akramkhan/codex-*`.

## Prerequisites

- Run the commands below from the repo root.
- Use this modified repo checkout, not upstream `main`.
- Install `curl`, `node`, `npm`, `python3`, `rustup`, `cargo`, and `rg`.
- Use the same `VERSION` on every machine.
- Publish platform packages first and the meta package last.

## Set Release Version

Before running any `cargo build`, temporarily update
`codex-rs/Cargo.toml` so the native `codex` binary reports the release
version instead of the dev placeholder.

Edit the `[workspace.package]` section to match the release you are
publishing:

```toml
[workspace.package]
version = "0.120.0"
```

Keep this change local and do not commit it. Every machine that builds a
native package needs the same temporary edit. After publishing, restore
the file:

```bash
git restore codex-rs/Cargo.toml
```

## Build Linux x64

Run on a Linux x64 machine.

This mirrors the musl-specific setup from `.github/workflows/rust-release.yml`.

```bash
VERSION=0.120.0
TARGET=x86_64-unknown-linux-musl
PACKAGE=codex-linux-x64
TAG=linux-x64
OUT="$PWD/out/$TAG"
STAGE_ROOT="$OUT/stage"

rustup target add "$TARGET"

ZIG_DIR=/tmp/zig-0.14.0
if [ ! -x "$ZIG_DIR/zig" ]; then
  rm -rf /tmp/zig-0.14.0 /tmp/zig-linux-x86_64-0.14.0 /tmp/zig-linux-x86_64-0.14.0.tar.xz
  curl -fsSL https://ziglang.org/download/0.14.0/zig-linux-x86_64-0.14.0.tar.xz -o /tmp/zig-linux-x86_64-0.14.0.tar.xz
  tar -xJf /tmp/zig-linux-x86_64-0.14.0.tar.xz -C /tmp
  mv /tmp/zig-linux-x86_64-0.14.0 "$ZIG_DIR"
fi
export PATH="$ZIG_DIR:$PATH"

if command -v apt-get >/dev/null 2>&1; then
  sudo apt-get update -y
  sudo DEBIAN_FRONTEND=noninteractive apt-get install -y libubsan1
fi

ENVFILE="$(mktemp)"
export TARGET GITHUB_ENV="$ENVFILE"
bash .github/scripts/install-musl-build-tools.sh
while IFS= read -r line; do
  export "$line"
done < "$ENVFILE"

ubsan=""
if command -v ldconfig >/dev/null 2>&1; then
  ubsan="$(ldconfig -p | grep -m1 'libubsan\.so\.1' | sed -E 's/.*=> (.*)$/\1/')"
fi
WRAPPER="${RUNNER_TEMP:-/tmp}/rustc-ubsan-wrapper"
cat > "$WRAPPER" <<EOF
#!/usr/bin/env bash
set -euo pipefail
if [[ -n "${ubsan}" ]]; then
  export LD_PRELOAD="${ubsan}\${LD_PRELOAD:+:\${LD_PRELOAD}}"
fi
exec "\$1" "\${@:2}"
EOF
chmod +x "$WRAPPER"
export RUSTC_WRAPPER="$WRAPPER"
export RUSTC_WORKSPACE_WRAPPER=

export AWS_LC_SYS_NO_JITTER_ENTROPY=1
export AWS_LC_SYS_NO_JITTER_ENTROPY_X86_64_UNKNOWN_LINUX_MUSL=1
export RUSTFLAGS=
export CARGO_ENCODED_RUSTFLAGS=
export RUSTDOCFLAGS=
export CARGO_BUILD_RUSTFLAGS=
export CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_RUSTFLAGS=
export CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_RUSTFLAGS=
export CARGO_TARGET_X86_64_UNKNOWN_LINUX_MUSL_RUSTFLAGS=
export CARGO_TARGET_AARCH64_UNKNOWN_LINUX_MUSL_RUSTFLAGS=

version="$(python3 .github/scripts/rusty_v8_bazel.py resolved-v8-crate-version)"
release_tag="rusty-v8-v${version}"
base_url="https://github.com/openai/codex/releases/download/${release_tag}"
binding_dir="${RUNNER_TEMP:-/tmp}/rusty_v8"
binding_path="${binding_dir}/src_binding_release_${TARGET}.rs"
mkdir -p "$binding_dir"
curl -fsSL "${base_url}/src_binding_release_${TARGET}.rs" -o "$binding_path"
export RUSTY_V8_ARCHIVE="${base_url}/librusty_v8_release_${TARGET}.a.gz"
export RUSTY_V8_SRC_BINDING_PATH="$binding_path"

(
  cd codex-rs
  cargo build --release --target "$TARGET" --bin codex
)

rm -rf "$STAGE_ROOT"
mkdir -p "$STAGE_ROOT"
python3 codex-cli/scripts/install_native_deps.py --component rg "$STAGE_ROOT"

VENDOR="$STAGE_ROOT/vendor"
mkdir -p "$VENDOR/$TARGET/codex"
cp "codex-rs/target/$TARGET/release/codex" "$VENDOR/$TARGET/codex/codex"

python3 codex-cli/scripts/build_npm_package.py \
  --package "$PACKAGE" \
  --release-version "$VERSION" \
  --vendor-src "$VENDOR" \
  --pack-output "$OUT/akramkhan-codex-$VERSION-$TAG.tgz"
```

## Build macOS

Run on a native Mac.

Apple Silicon:

```bash
VERSION=0.120.0
TARGET=aarch64-apple-darwin
PACKAGE=codex-darwin-arm64
TAG=darwin-arm64

rustup target add "$TARGET"

(
  cd codex-rs
  cargo build --release --target "$TARGET" --bin codex
)

VENDOR="$PWD/out/$TAG/vendor"
mkdir -p "$VENDOR/$TARGET/codex" "$VENDOR/$TARGET/path"
cp "codex-rs/target/$TARGET/release/codex" "$VENDOR/$TARGET/codex/codex"
cp "$(command -v rg)" "$VENDOR/$TARGET/path/rg"

python3 codex-cli/scripts/build_npm_package.py \
  --package "$PACKAGE" \
  --release-version "$VERSION" \
  --vendor-src "$VENDOR" \
  --pack-output "$PWD/out/$TAG/akramkhan-codex-$VERSION-$TAG.tgz"
```

Intel Mac:

```bash
VERSION=0.120.0
TARGET=x86_64-apple-darwin
PACKAGE=codex-darwin-x64
TAG=darwin-x64

rustup target add "$TARGET"

(
  cd codex-rs
  cargo build --release --target "$TARGET" --bin codex
)

VENDOR="$PWD/out/$TAG/vendor"
mkdir -p "$VENDOR/$TARGET/codex" "$VENDOR/$TARGET/path"
cp "codex-rs/target/$TARGET/release/codex" "$VENDOR/$TARGET/codex/codex"
cp "$(command -v rg)" "$VENDOR/$TARGET/path/rg"

python3 codex-cli/scripts/build_npm_package.py \
  --package "$PACKAGE" \
  --release-version "$VERSION" \
  --vendor-src "$VENDOR" \
  --pack-output "$PWD/out/$TAG/akramkhan-codex-$VERSION-$TAG.tgz"
```

## Build Windows

Run in PowerShell on native Windows.

Windows x64:

```powershell
$Version = "0.120.0"
$Target = "x86_64-pc-windows-msvc"
$Package = "codex-win32-x64"
$Tag = "win32-x64"

rustup target add $Target

Push-Location codex-rs
cargo build --release --target $Target --bin codex --bin codex-windows-sandbox-setup --bin codex-command-runner
Pop-Location

$Vendor = Join-Path $PWD "out\$Tag\vendor"
New-Item -ItemType Directory -Force -Path "$Vendor\$Target\codex" | Out-Null
New-Item -ItemType Directory -Force -Path "$Vendor\$Target\path" | Out-Null

Copy-Item "codex-rs\target\$Target\release\codex.exe" "$Vendor\$Target\codex\codex.exe"
Copy-Item "codex-rs\target\$Target\release\codex-windows-sandbox-setup.exe" "$Vendor\$Target\codex\codex-windows-sandbox-setup.exe"
Copy-Item "codex-rs\target\$Target\release\codex-command-runner.exe" "$Vendor\$Target\codex\codex-command-runner.exe"
Copy-Item (Get-Command rg.exe).Source "$Vendor\$Target\path\rg.exe"

py -3 codex-cli\scripts\build_npm_package.py `
  --package $Package `
  --release-version $Version `
  --vendor-src $Vendor `
  --pack-output (Join-Path $PWD "out\$Tag\akramkhan-codex-$Version-$Tag.tgz")
```

Windows ARM64:

```powershell
$Version = "0.120.0"
$Target = "aarch64-pc-windows-msvc"
$Package = "codex-win32-arm64"
$Tag = "win32-arm64"

rustup target add $Target

Push-Location codex-rs
cargo build --release --target $Target --bin codex --bin codex-windows-sandbox-setup --bin codex-command-runner
Pop-Location

$Vendor = Join-Path $PWD "out\$Tag\vendor"
New-Item -ItemType Directory -Force -Path "$Vendor\$Target\codex" | Out-Null
New-Item -ItemType Directory -Force -Path "$Vendor\$Target\path" | Out-Null

Copy-Item "codex-rs\target\$Target\release\codex.exe" "$Vendor\$Target\codex\codex.exe"
Copy-Item "codex-rs\target\$Target\release\codex-windows-sandbox-setup.exe" "$Vendor\$Target\codex\codex-windows-sandbox-setup.exe"
Copy-Item "codex-rs\target\$Target\release\codex-command-runner.exe" "$Vendor\$Target\codex\codex-command-runner.exe"
Copy-Item (Get-Command rg.exe).Source "$Vendor\$Target\path\rg.exe"

py -3 codex-cli\scripts\build_npm_package.py `
  --package $Package `
  --release-version $Version `
  --vendor-src $Vendor `
  --pack-output (Join-Path $PWD "out\$Tag\akramkhan-codex-$Version-$Tag.tgz")
```

## Build Meta Package

Run on any machine after the platform tarballs are ready:

```bash
VERSION=0.120.0

python3 codex-cli/scripts/build_npm_package.py \
  --package codex \
  --release-version "$VERSION" \
  --pack-output "$PWD/out/meta/akramkhan-codex-$VERSION.tgz"
```

## Publish

Publish platform tarballs first. Publish the meta package last.

```bash
VERSION=0.120.0

npm publish "out/linux-x64/akramkhan-codex-$VERSION-linux-x64.tgz" --tag linux-x64 --access public
npm publish "out/linux-arm64/akramkhan-codex-$VERSION-linux-arm64.tgz" --tag linux-arm64 --access public
npm publish "out/darwin-x64/akramkhan-codex-$VERSION-darwin-x64.tgz" --tag darwin-x64 --access public
npm publish "out/darwin-arm64/akramkhan-codex-$VERSION-darwin-arm64.tgz" --tag darwin-arm64 --access public
npm publish "out/win32-x64/akramkhan-codex-$VERSION-win32-x64.tgz" --tag win32-x64 --access public
npm publish "out/win32-arm64/akramkhan-codex-$VERSION-win32-arm64.tgz" --tag win32-arm64 --access public
npm publish "out/meta/akramkhan-codex-$VERSION.tgz" --access public
```

If you want Linux x64 first and other platforms later, this order is safe:

```bash
VERSION=0.120.0

npm publish "out/linux-x64/akramkhan-codex-$VERSION-linux-x64.tgz" --tag linux-x64 --access public
npm publish "out/meta/akramkhan-codex-$VERSION.tgz" --access public
```

The remaining platform packages for that same `VERSION` can be published
later without republishing meta.

If your npm account enforces 2FA for publish, npm will prompt for the OTP
interactively. Add `--otp=123456` only if you want a non-interactive command.

## Install

After publish:

```bash
npm install -g @akramkhan/codex
```

For local testing from a tarball:

```bash
VERSION=0.120.0

npm install -g "./out/linux-x64/akramkhan-codex-$VERSION-linux-x64.tgz"
```

## Verify

```bash
codex --version
codex
```
