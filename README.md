# solana-tools-arm64

Prebuilt **aarch64 Linux** binaries of the two Solana build drivers that ship
no arm64 release upstream:

| Tool | Crate | What it is |
|---|---|---|
| `cargo-build-sbf` (+ `cargo-test-sbf`) | [cargo-build-sbf](https://crates.io/crates/cargo-build-sbf) | The driver that turns a Rust crate into an SBF program |
| `anchor` | [anchor-cli](https://crates.io/crates/anchor-cli) | The Anchor CLI |

They exist for [Seeker IDE](https://github.com/cesp99/SeekerIDE), which
installs a Solana toolchain into a Debian userland on a Solana Seeker phone.
Compiling these two crates on the phone took nine of the thirteen minutes of
that setup; downloading them takes seconds.

## How a build happens

Every build is a GitHub Actions run on GitHub's own `ubuntu-24.04-arm`
runner — nothing is cross-compiled — inside a `debian:stable-slim`
container, the same image the phone unpacks, so the binaries link against
the same glibc and OpenSSL. The recipe is `cargo install <crate> --version
<version>` with the crate's own release profile, then `strip`. Nothing is
patched. See [`.github/workflows/build.yml`](.github/workflows/build.yml).

To bump a tool, edit [`versions.json`](versions.json) and push to `main`.
The workflow builds every tool whose release tag does not exist yet and
publishes one release per tool, tagged `<tool>-v<version>`, with:

- `<tool>-v<version>-aarch64-unknown-linux-gnu.tar.gz` — a `bin/` directory
  with the executables, ready to unpack over `/opt/solana/cli`;
- the `.sha256` of that archive;
- `build-info.txt` — image, glibc, rustc, the exact command and the run URL;
- a signed [build provenance attestation](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations/using-artifact-attestations-to-establish-provenance-for-builds).

Verify an archive came from this workflow:

```
gh attestation verify cargo-build-sbf-v4.2.0-aarch64-unknown-linux-gnu.tar.gz --repo cesp99/solana-tools-arm64
```

## The manifest

[`manifest.json`](manifest.json) is a copy of Seeker IDE's toolchain
manifest (`app/src/main/assets/solana/toolchain/manifest.json`), published
here so the IDE's **Check for updates** can fetch it without an app release:
the IDE adopts it when its `released` date is later than the one in use and
reinstalls only the rows whose pinned revision changed. Keep the two files
identical — a toolchain bump is an edit there, a copy here, one commit each.

## Consuming from Seeker IDE

The IDE's toolchain manifest
(`app/src/main/assets/solana/toolchain/manifest.json`) pins each tool by
release URL and the SHA-256 from the `.sha256` asset, and unpacks it with
the phone's own `tar` into `/opt/solana/cli`. A bump there is: bump
`versions.json` here, wait for the release, copy the hash.

## Licence

The workflow and this repository are MIT. The binaries are unmodified
builds of the upstream crates and carry their licences: cargo-build-sbf
(Apache-2.0, Anza) and anchor-cli (Apache-2.0, Coral).
