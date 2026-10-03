# Codex for Termux

A native `aarch64-linux-android` build of the OpenAI Codex CLI, packaged for Termux.

Based on [wallentx/codex-termux](https://github.com/wallentx/codex-termux), which carries
the Android build and file-locking fixes, plus the changes below.

## Install

Download the `.deb` from the releases page, then:

```
pkg install ./codex_<version>_aarch64.deb
```

Sign in with **Device Code** (`codex login --device-auth`); the browser redirect flow
does not work well on a phone.

## What differs from upstream Codex

| Area | Upstream on Termux | This build |
|---|---|---|
| DNS / TLS | Linux musl build can't find `/etc/resolv.conf` or `/etc/ssl` | Native Android build uses the system resolver and Termux certs |
| File locks | `flock` unsupported on `/data` → startup errors | Falls back to lock directories (from wallentx) |
| App-server daemon | Socket dir hard-coded to `/tmp`, which Android lacks | Uses `$PREFIX/tmp`, fixed at build time |
| Command sandbox | bubblewrap needs user namespaces, which Android blocks; commands silently run unsandboxed | Requests approval for commands outside the trusted command set by default; an explicit approval policy overrides this |
| Updates | Self-updates from npm / GitHub | Disabled; update with `pkg upgrade codex` |
| Code mode (V8) | Bundles V8 | Not shipped. Packaged Android builds use direct tools, including for models that request code mode |

## App-server daemon

The package includes the layout required by `codex app-server daemon start`.
The daemon keeps a private copy pinned to this Termux package and does not
automatically install standalone releases. After upgrading the package, update
that copy with `codex app-server daemon update --from-cli --yes`.

After configuring the signed repository below, update with `pkg upgrade codex`.
Manual installation of newer `.deb` files is also supported.

## Building

CI builds it in `.github/workflows/termux-package.yml`. The binary is built with
`CODEX_TERMUX_PACKAGED=1`, which turns off the built-in updater.

## Signed package repository

Package artifacts are hosted separately from the private source repository:
`https://red-banana-official.github.io/codex-termux-apt/`.
The repository uses a dedicated signing key with fingerprint
`A249096DFDE86DA5A84F7E708D4E41608382F64A` (expires October 2028).
Configure a repository-specific `signed-by` keyring; never use `trusted=yes`.

To prepare a release on the signing machine, install `apt-utils`, `gnupg`, and
`dpkg`, then run:

```bash
python3 packaging/termux/build-repository.py \
  --gnupghome /path/to/private/gnupg-home \
  --key A249096DFDE86DA5A84F7E708D4E41608382F64A \
  --output /path/to/new/repository \
  /path/to/tested/codex_0.160.0-2_aarch64.deb
```

The output must not already exist. Publish only the generated directory, never
the signing home, private-key backup, or revocation certificate. The builder
signs `InRelease` and `Release.gpg` and exports only the public key.
Use the exact CI package that passed phone acceptance; no recompilation is needed.

Run local repository regression tests on the VM before publishing:

```bash
python3 packaging/termux/test_build_repository.py
```

Tests use disposable signing keys and isolated APT lists. They verify signatures,
package indexing, normal APT updates, and rejection of modified signed metadata.

### Configure Termux

```bash
mkdir -p "$PREFIX/etc/apt/keyrings"
curl --fail --location --output "$PREFIX/etc/apt/keyrings/codex-termux.gpg" \
  https://red-banana-official.github.io/codex-termux-apt/codex-termux.gpg
printf '%s  %s\n' \
  00b9aec02f7196b5063979c9d707b04f652bda9ab05978686845cdda0ab633b7 \
  "$PREFIX/etc/apt/keyrings/codex-termux.gpg" | sha256sum -c -
```

Continue only if the checksum reports `OK`:

```bash
printf 'deb [signed-by=%s/etc/apt/keyrings/codex-termux.gpg] https://red-banana-official.github.io/codex-termux-apt/ ./\n' \
  "$PREFIX" > "$PREFIX/etc/apt/sources.list.d/codex-termux.list"
pkg update
pkg install codex
```

After package upgrades, refresh the daemon's private copy:

```bash
codex app-server daemon update --from-cli --yes
```
