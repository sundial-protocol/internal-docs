# SBOM - Sundial Runtime Packages

Generated: 2026-08-17

## Purpose

This document is a stage-independent dependency snapshot for the demo Midgard TypeScript packages. It is intended as a living SBOM artifact for release review: regenerate it whenever `demo/pnpm-lock.yaml` or any scoped `package.json` changes.

The release-facing aggregate is `demo/midgard-node` (`sundial-node`). It declares `@al-ft/midgard-sdk` as a workspace dependency. `demo/midgard-ts` is also captured because the node imports it by relative source path and the node Docker build copies it into the build context, even though it is not declared as a package dependency of `sundial-node`.

Current determination: this document is the current public-facing SBOM snapshot
for the Sundial node runtime dependency closure as of the generated date.

## Scope

| Item | Value |
| --- | --- |
| Workspace | `demo/pnpm-workspace.yaml` |
| Lockfile | `demo/pnpm-lock.yaml` |
| Included package | `demo/midgard-node` / `sundial-node@1.0.0` |
| Included package | `demo/midgard-sdk` / `@al-ft/midgard-sdk@0.1.0` |
| Included package | `demo/midgard-ts` / `midgard-ts@0.1.0` |
| Dependency classes | Production `dependencies` only; `devDependencies` excluded except where a package is currently declared under `dependencies` |
| Excluded | OS packages, Docker base images, Cardano node/provider services, registry license metadata, vulnerability status, and generated build output |

## Source Integrity

| Source file | SHA-256 |
| --- | --- |
| `demo/pnpm-lock.yaml` | `f6a3c8393dd5aa1524a6882fb776f825f6c2051a7c74357cc3418ef91e5c6c7c` |
| `demo/midgard-node/package.json` | `5e214ec451567cfa589a9f24d8b2e8033058e34236b1e8d8a377f80f7c980baf` |
| `demo/midgard-sdk/package.json` | `ae12b600d16fa497bbc27ae6067c1e150eeb9121619194c781ba7bf73b61a793` |
| `demo/midgard-ts/package.json` | `322028720522534ab1c3f271a0ae5b2021d84d7d1e848d651003b1ff7b20cc49` |

## Snapshot Summary

| Snapshot | Component count | Notes |
| --- | ---: | --- |
| `sundial-node` production closure | 423 | Includes workspace `@al-ft/midgard-sdk` and its production closure. |
| `@al-ft/midgard-sdk` production closure | 162 | Captured independently for SDK release traceability. |
| `midgard-ts` production closure | 1 | Single direct runtime dependency in the lockfile. |
| Node-facing aggregate | 424 | 422 registry components, 1 workspace dependency, 1 source-linked workspace component. |

## Workspace Dependency Shape

```text
sundial-node@1.0.0
|-- @al-ft/midgard-sdk@0.1.0 (workspace dependency declared as workspace:*)
|   `-- registry production dependencies resolved by demo/pnpm-lock.yaml
`-- midgard-ts@0.1.0 (workspace source dependency: relative imports + Docker build context)
    `-- @dcspark/cardano-multiplatform-lib-nodejs@6.2.0
```

## Direct Runtime Dependency Roots

| Package | Dependency | Requested | Resolved |
| --- | --- | --- | --- |
| `sundial-node` | `@aiken-lang/merkle-patricia-forestry` | `^1.3.1` | `1.3.1` |
| `sundial-node` | `@al-ft/midgard-sdk` | `workspace:*` | `link:../midgard-sdk` |
| `sundial-node` | `@dcspark/cardano-multiplatform-lib-nodejs` | `^6.2.0` | `6.2.0` |
| `sundial-node` | `@effect/cluster` | `^0.58.2` | `0.58.2` |
| `sundial-node` | `@effect/experimental` | `^0.60.0` | `0.60.0` |
| `sundial-node` | `@effect/opentelemetry` | `^0.63.0` | `0.63.0` |
| `sundial-node` | `@effect/platform` | `^0.96.1` | `0.96.1` |
| `sundial-node` | `@effect/platform-node` | `^0.106.0` | `0.106.0` |
| `sundial-node` | `@effect/rpc` | `^0.75.1` | `0.75.1` |
| `sundial-node` | `@effect/sql` | `^0.51.1` | `0.51.1` |
| `sundial-node` | `@effect/sql-pg` | `^0.52.1` | `0.52.1` |
| `sundial-node` | `@electric-sql/pglite` | `^0.4.5` | `0.4.5` |
| `sundial-node` | `@ethereumjs/mpt` | `7.0.0-alpha.1` | `7.0.0-alpha.1` |
| `sundial-node` | `@ethereumjs/util` | `^9.1.0` | `9.1.0` |
| `sundial-node` | `@harmoniclabs/bytestring` | `^1.0.0` | `1.0.0` |
| `sundial-node` | `@harmoniclabs/cbor` | `^1.6.6` | `1.6.6` |
| `sundial-node` | `@harmoniclabs/crypto` | `^0.3.0` | `0.3.0` |
| `sundial-node` | `@harmoniclabs/pair` | `^1.0.0` | `1.0.0` |
| `sundial-node` | `@lucid-evolution/lucid` | `^0.4.30` | `0.4.30` |
| `sundial-node` | `@opentelemetry/api` | `^1.9.1` | `1.9.1` |
| `sundial-node` | `@opentelemetry/exporter-prometheus` | `^0.217.0` | `0.217.0` |
| `sundial-node` | `@opentelemetry/exporter-trace-otlp-http` | `^0.218.0` | `0.218.0` |
| `sundial-node` | `@opentelemetry/sdk-trace-base` | `^2.7.1` | `2.7.1` |
| `sundial-node` | `@opentelemetry/sdk-trace-node` | `^2.7.1` | `2.7.1` |
| `sundial-node` | `@opentelemetry/sdk-trace-web` | `^2.7.1` | `2.7.1` |
| `sundial-node` | `chalk` | `^5.6.2` | `5.6.2` |
| `sundial-node` | `commander` | `^13.1.0` | `13.1.0` |
| `sundial-node` | `dotenv` | `^16.6.1` | `16.6.1` |
| `sundial-node` | `effect` | `3.21.2` | `3.21.2` |
| `sundial-node` | `ioredis` | `^5.10.1` | `5.10.1` |
| `sundial-node` | `level` | `^10.0.0` | `10.0.0` |
| `sundial-node` | `memory-level` | `^3.1.0` | `3.1.0` |
| `sundial-node` | `rimraf` | `^6.1.3` | `6.1.3` |
| `sundial-node` | `vitest` | `3.2.6` | `3.2.6` |
| `@al-ft/midgard-sdk` | `@harmoniclabs/crypto` | `^0.3.0` | `0.3.0` |
| `@al-ft/midgard-sdk` | `@lucid-evolution/lucid` | `^0.4.30` | `0.4.30` |
| `@al-ft/midgard-sdk` | `@lucid-evolution/utils` | `^0.1.66` | `0.1.66` |
| `@al-ft/midgard-sdk` | `@noble/hashes` | `^1.8.0` | `1.8.0` |
| `@al-ft/midgard-sdk` | `effect` | `3.21.2` | `3.21.2` |
| `midgard-ts` | `@dcspark/cardano-multiplatform-lib-nodejs` | `^6.2.0` | `6.2.0` |

## Regeneration

Regenerate this document per release, after dependency installation has succeeded with the checked-in lockfile. The dependency tree can be checked with:

```bash
pnpm --dir demo install --frozen-lockfile
pnpm --dir demo --filter sundial-node list --prod --depth Infinity
pnpm --dir demo --filter @al-ft/midgard-sdk list --prod --depth Infinity
pnpm --dir demo --filter midgard-ts list --prod --depth Infinity
```

For the node release SBOM, the `sundial-node` closure is the primary package-manager view. Keep the explicit `midgard-ts` entry unless the node stops importing/copying it by source path or begins declaring it as a normal workspace package dependency.

## SBOM Determination

Sundial has a documented runtime dependency inventory for the current
`sundial-node` release-facing package closure, including its workspace-linked
runtime dependencies.

This SBOM is a dependency inventory and integrity artifact. It is not a
vulnerability attestation, license opinion, or operating-environment inventory.

## Resolved Runtime Inventory

This flattened inventory is derived from the lockfile `importers`, `packages`, and `snapshots` sections. Multiple versions of the same package are listed separately because they are distinct resolved components. Optional platform packages are retained when present in the lockfile closure.

| Component | Version | Source |
| --- | --- | --- |
| `@aiken-lang/merkle-patricia-forestry` | `1.3.1` | registry |
| `@al-ft/midgard-sdk` | `0.1.0` | workspace |
| `@anastasia-labs/cardano-multiplatform-lib-browser` | `6.0.2-2` | registry |
| `@anastasia-labs/cardano-multiplatform-lib-browser` | `6.0.2-3` | registry |
| `@anastasia-labs/cardano-multiplatform-lib-nodejs` | `6.0.2-2` | registry |
| `@anastasia-labs/cardano-multiplatform-lib-nodejs` | `6.0.2-3` | registry |
| `@biglup/is-cid` | `1.0.3` | registry |
| `@cardano-ogmios/client` | `6.9.0` | registry |
| `@cardano-ogmios/schema` | `6.9.0` | registry |
| `@cardano-sdk/core` | `0.46.12` | registry |
| `@cardano-sdk/crypto` | `0.4.5` | registry |
| `@cardano-sdk/util` | `0.17.1` | registry |
| `@cardanosolutions/json-bigint` | `1.1.0` | registry |
| `@cbor-extract/cbor-extract-darwin-arm64` | `2.2.2` | registry |
| `@cbor-extract/cbor-extract-darwin-x64` | `2.2.2` | registry |
| `@cbor-extract/cbor-extract-linux-arm` | `2.2.2` | registry |
| `@cbor-extract/cbor-extract-linux-arm64` | `2.2.2` | registry |
| `@cbor-extract/cbor-extract-linux-x64` | `2.2.2` | registry |
| `@cbor-extract/cbor-extract-win32-x64` | `2.2.2` | registry |
| `@chainsafe/is-ip` | `2.1.0` | registry |
| `@chainsafe/netmask` | `2.0.0` | registry |
| `@dcspark/cardano-multiplatform-lib-nodejs` | `6.2.0` | registry |
| `@dnsquery/dns-packet` | `6.1.1` | registry |
| `@effect/cluster` | `0.58.2` | registry |
| `@effect/experimental` | `0.60.0` | registry |
| `@effect/opentelemetry` | `0.63.0` | registry |
| `@effect/platform` | `0.71.7` | registry |
| `@effect/platform` | `0.96.1` | registry |
| `@effect/platform-node` | `0.106.0` | registry |
| `@effect/platform-node-shared` | `0.59.0` | registry |
| `@effect/rpc` | `0.75.1` | registry |
| `@effect/schema` | `0.66.16` | registry |
| `@effect/schema` | `0.68.27` | registry |
| `@effect/sql` | `0.51.1` | registry |
| `@effect/sql-pg` | `0.52.1` | registry |
| `@effect/workflow` | `0.18.1` | registry |
| `@electric-sql/pglite` | `0.4.5` | registry |
| `@emnapi/core` | `1.11.1` | registry |
| `@emnapi/runtime` | `1.11.1` | registry |
| `@emnapi/wasi-threads` | `1.2.2` | registry |
| `@emurgo/cardano-message-signing-browser` | `1.1.0` | registry |
| `@emurgo/cardano-message-signing-nodejs` | `1.1.0` | registry |
| `@esbuild/aix-ppc64` | `0.28.1` | registry |
| `@esbuild/android-arm` | `0.28.1` | registry |
| `@esbuild/android-arm64` | `0.28.1` | registry |
| `@esbuild/android-x64` | `0.28.1` | registry |
| `@esbuild/darwin-arm64` | `0.28.1` | registry |
| `@esbuild/darwin-x64` | `0.28.1` | registry |
| `@esbuild/freebsd-arm64` | `0.28.1` | registry |
| `@esbuild/freebsd-x64` | `0.28.1` | registry |
| `@esbuild/linux-arm` | `0.28.1` | registry |
| `@esbuild/linux-arm64` | `0.28.1` | registry |
| `@esbuild/linux-ia32` | `0.28.1` | registry |
| `@esbuild/linux-loong64` | `0.28.1` | registry |
| `@esbuild/linux-mips64el` | `0.28.1` | registry |
| `@esbuild/linux-ppc64` | `0.28.1` | registry |
| `@esbuild/linux-riscv64` | `0.28.1` | registry |
| `@esbuild/linux-s390x` | `0.28.1` | registry |
| `@esbuild/linux-x64` | `0.28.1` | registry |
| `@esbuild/netbsd-arm64` | `0.28.1` | registry |
| `@esbuild/netbsd-x64` | `0.28.1` | registry |
| `@esbuild/openbsd-arm64` | `0.28.1` | registry |
| `@esbuild/openbsd-x64` | `0.28.1` | registry |
| `@esbuild/openharmony-arm64` | `0.28.1` | registry |
| `@esbuild/sunos-x64` | `0.28.1` | registry |
| `@esbuild/win32-arm64` | `0.28.1` | registry |
| `@esbuild/win32-ia32` | `0.28.1` | registry |
| `@esbuild/win32-x64` | `0.28.1` | registry |
| `@ethereumjs/mpt` | `7.0.0-alpha.1` | registry |
| `@ethereumjs/rlp` | `10.1.1` | registry |
| `@ethereumjs/rlp` | `5.0.2` | registry |
| `@ethereumjs/rlp` | `6.0.0-alpha.1` | registry |
| `@ethereumjs/util` | `10.1.1` | registry |
| `@ethereumjs/util` | `9.1.0` | registry |
| `@foxglove/crc` | `0.0.3` | registry |
| `@harmoniclabs/bigint-utils` | `1.0.0` | registry |
| `@harmoniclabs/biguint` | `1.0.0` | registry |
| `@harmoniclabs/bitstream` | `1.0.0` | registry |
| `@harmoniclabs/bytestring` | `1.0.0` | registry |
| `@harmoniclabs/cbor` | `1.6.6` | registry |
| `@harmoniclabs/crypto` | `0.2.5` | registry |
| `@harmoniclabs/crypto` | `0.3.0` | registry |
| `@harmoniclabs/obj-utils` | `1.0.0` | registry |
| `@harmoniclabs/pair` | `1.0.0` | registry |
| `@harmoniclabs/plutus-data` | `1.2.6` | registry |
| `@harmoniclabs/uint8array-utils` | `1.0.4` | registry |
| `@harmoniclabs/uplc` | `1.4.1` | registry |
| `@ioredis/commands` | `1.5.1` | registry |
| `@jridgewell/sourcemap-codec` | `1.5.5` | registry |
| `@leichtgewicht/ip-codec` | `2.0.5` | registry |
| `@libp2p/interface` | `3.2.2` | registry |
| `@lucid-evolution/core-types` | `0.1.22` | registry |
| `@lucid-evolution/core-utils` | `0.1.16` | registry |
| `@lucid-evolution/crc8` | `0.1.8` | registry |
| `@lucid-evolution/lucid` | `0.4.30` | registry |
| `@lucid-evolution/plutus` | `0.1.29` | registry |
| `@lucid-evolution/provider` | `0.1.91` | registry |
| `@lucid-evolution/sign_data` | `0.1.25` | registry |
| `@lucid-evolution/uplc` | `0.2.20` | registry |
| `@lucid-evolution/utils` | `0.1.66` | registry |
| `@lucid-evolution/wallet` | `0.1.72` | registry |
| `@msgpackr-extract/msgpackr-extract-darwin-arm64` | `3.0.3` | registry |
| `@msgpackr-extract/msgpackr-extract-darwin-x64` | `3.0.3` | registry |
| `@msgpackr-extract/msgpackr-extract-linux-arm` | `3.0.3` | registry |
| `@msgpackr-extract/msgpackr-extract-linux-arm64` | `3.0.3` | registry |
| `@msgpackr-extract/msgpackr-extract-linux-x64` | `3.0.3` | registry |
| `@msgpackr-extract/msgpackr-extract-win32-x64` | `3.0.3` | registry |
| `@multiformats/dns` | `1.0.13` | registry |
| `@multiformats/mafmt` | `12.1.6` | registry |
| `@multiformats/multiaddr` | `12.5.1` | registry |
| `@multiformats/multiaddr` | `13.0.1` | registry |
| `@napi-rs/wasm-runtime` | `1.1.5` | registry |
| `@noble/ciphers` | `1.3.0` | registry |
| `@noble/curves` | `1.4.2` | registry |
| `@noble/curves` | `1.9.0` | registry |
| `@noble/curves` | `2.2.0` | registry |
| `@noble/hashes` | `1.4.0` | registry |
| `@noble/hashes` | `1.8.0` | registry |
| `@noble/hashes` | `2.2.0` | registry |
| `@opentelemetry/api` | `1.9.1` | registry |
| `@opentelemetry/api-logs` | `0.218.0` | registry |
| `@opentelemetry/context-async-hooks` | `2.7.1` | registry |
| `@opentelemetry/core` | `2.8.0` | registry |
| `@opentelemetry/exporter-prometheus` | `0.217.0` | registry |
| `@opentelemetry/exporter-trace-otlp-http` | `0.218.0` | registry |
| `@opentelemetry/otlp-exporter-base` | `0.218.0` | registry |
| `@opentelemetry/otlp-transformer` | `0.218.0` | registry |
| `@opentelemetry/resources` | `2.7.1` | registry |
| `@opentelemetry/sdk-logs` | `0.218.0` | registry |
| `@opentelemetry/sdk-metrics` | `2.7.1` | registry |
| `@opentelemetry/sdk-trace-base` | `2.7.1` | registry |
| `@opentelemetry/sdk-trace-node` | `2.7.1` | registry |
| `@opentelemetry/sdk-trace-web` | `2.7.1` | registry |
| `@opentelemetry/semantic-conventions` | `1.40.0` | registry |
| `@oxc-project/types` | `0.137.0` | registry |
| `@parcel/watcher` | `2.5.6` | registry |
| `@parcel/watcher-android-arm64` | `2.5.6` | registry |
| `@parcel/watcher-darwin-arm64` | `2.5.6` | registry |
| `@parcel/watcher-darwin-x64` | `2.5.6` | registry |
| `@parcel/watcher-freebsd-x64` | `2.5.6` | registry |
| `@parcel/watcher-linux-arm-glibc` | `2.5.6` | registry |
| `@parcel/watcher-linux-arm-musl` | `2.5.6` | registry |
| `@parcel/watcher-linux-arm64-glibc` | `2.5.6` | registry |
| `@parcel/watcher-linux-arm64-musl` | `2.5.6` | registry |
| `@parcel/watcher-linux-x64-glibc` | `2.5.6` | registry |
| `@parcel/watcher-linux-x64-musl` | `2.5.6` | registry |
| `@parcel/watcher-win32-arm64` | `2.5.6` | registry |
| `@parcel/watcher-win32-ia32` | `2.5.6` | registry |
| `@parcel/watcher-win32-x64` | `2.5.6` | registry |
| `@rolldown/binding-android-arm64` | `1.1.2` | registry |
| `@rolldown/binding-darwin-arm64` | `1.1.2` | registry |
| `@rolldown/binding-darwin-x64` | `1.1.2` | registry |
| `@rolldown/binding-freebsd-x64` | `1.1.2` | registry |
| `@rolldown/binding-linux-arm-gnueabihf` | `1.1.2` | registry |
| `@rolldown/binding-linux-arm64-gnu` | `1.1.2` | registry |
| `@rolldown/binding-linux-arm64-musl` | `1.1.2` | registry |
| `@rolldown/binding-linux-ppc64-gnu` | `1.1.2` | registry |
| `@rolldown/binding-linux-s390x-gnu` | `1.1.2` | registry |
| `@rolldown/binding-linux-x64-gnu` | `1.1.2` | registry |
| `@rolldown/binding-linux-x64-musl` | `1.1.2` | registry |
| `@rolldown/binding-openharmony-arm64` | `1.1.2` | registry |
| `@rolldown/binding-wasm32-wasi` | `1.1.2` | registry |
| `@rolldown/binding-win32-arm64-msvc` | `1.1.2` | registry |
| `@rolldown/binding-win32-x64-msvc` | `1.1.2` | registry |
| `@rolldown/pluginutils` | `1.0.1` | registry |
| `@scure/base` | `1.1.9` | registry |
| `@scure/base` | `1.2.6` | registry |
| `@scure/bip32` | `1.4.0` | registry |
| `@scure/bip32` | `1.7.0` | registry |
| `@scure/bip39` | `1.3.0` | registry |
| `@scure/bip39` | `1.6.0` | registry |
| `@sinclair/typebox` | `0.32.35` | registry |
| `@standard-schema/spec` | `1.1.0` | registry |
| `@tybys/wasm-util` | `0.10.2` | registry |
| `@types/chai` | `5.2.3` | registry |
| `@types/deep-eql` | `4.0.2` | registry |
| `@types/estree` | `1.0.9` | registry |
| `@types/json-bigint` | `1.0.4` | registry |
| `@types/node` | `22.19.18` | registry |
| `@vitest/expect` | `3.2.6` | registry |
| `@vitest/mocker` | `3.2.6` | registry |
| `@vitest/pretty-format` | `3.2.6` | registry |
| `@vitest/runner` | `3.2.6` | registry |
| `@vitest/snapshot` | `3.2.6` | registry |
| `@vitest/spy` | `3.2.6` | registry |
| `@vitest/utils` | `3.2.6` | registry |
| `@zxing/text-encoding` | `0.9.0` | registry |
| `abort-error` | `1.0.2` | registry |
| `abstract-level` | `1.0.4` | registry |
| `abstract-level` | `3.1.1` | registry |
| `assertion-error` | `2.0.1` | registry |
| `available-typed-arrays` | `1.0.7` | registry |
| `b4a` | `1.8.1` | registry |
| `balanced-match` | `4.0.4` | registry |
| `base64-js` | `1.5.1` | registry |
| `bech32` | `2.0.0` | registry |
| `bip39` | `3.1.0` | registry |
| `blake2b` | `2.1.4` | registry |
| `blake2b-wasm` | `2.4.0` | registry |
| `brace-expansion` | `5.0.9` | registry |
| `browser-level` | `1.0.1` | registry |
| `browser-level` | `3.0.0` | registry |
| `buffer` | `6.0.3` | registry |
| `cac` | `6.7.14` | registry |
| `call-bind` | `1.0.9` | registry |
| `call-bind-apply-helpers` | `1.0.2` | registry |
| `call-bound` | `1.0.4` | registry |
| `catering` | `2.1.1` | registry |
| `cbor-extract` | `2.2.2` | registry |
| `cbor-x` | `1.6.4` | registry |
| `chai` | `5.3.3` | registry |
| `chalk` | `5.6.2` | registry |
| `check-error` | `2.1.3` | registry |
| `cipher-base` | `1.0.7` | registry |
| `classic-level` | `1.4.1` | registry |
| `classic-level` | `3.0.0` | registry |
| `cluster-key-slot` | `1.1.2` | registry |
| `commander` | `13.1.0` | registry |
| `core-util-is` | `1.0.3` | registry |
| `create-hash` | `1.2.0` | registry |
| `create-hmac` | `1.1.7` | registry |
| `cross-fetch` | `3.2.0` | registry |
| `debug` | `4.4.3` | registry |
| `deep-eql` | `5.0.2` | registry |
| `define-data-property` | `1.1.4` | registry |
| `denque` | `2.1.0` | registry |
| `detect-libc` | `2.1.2` | registry |
| `dotenv` | `16.6.1` | registry |
| `dunder-proto` | `1.0.1` | registry |
| `effect` | `3.21.2` | registry |
| `es-define-property` | `1.0.1` | registry |
| `es-errors` | `1.3.0` | registry |
| `es-module-lexer` | `1.7.0` | registry |
| `es-object-atoms` | `1.1.1` | registry |
| `esbuild` | `0.28.1` | registry |
| `estree-walker` | `3.0.3` | registry |
| `ethereum-cryptography` | `2.2.1` | registry |
| `ethereum-cryptography` | `3.2.0` | registry |
| `eventemitter3` | `5.0.4` | registry |
| `expect-type` | `1.3.0` | registry |
| `fast-check` | `3.23.2` | registry |
| `fastq` | `1.20.1` | registry |
| `fdir` | `6.5.0` | registry |
| `find-my-way-ts` | `0.1.6` | registry |
| `for-each` | `0.3.5` | registry |
| `fraction.js` | `4.0.1` | registry |
| `fsevents` | `2.3.3` | registry |
| `function-bind` | `1.1.2` | registry |
| `functional-red-black-tree` | `1.0.1` | registry |
| `generator-function` | `2.0.1` | registry |
| `get-intrinsic` | `1.3.0` | registry |
| `get-proto` | `1.0.1` | registry |
| `glob` | `13.0.6` | registry |
| `gopd` | `1.2.0` | registry |
| `has-property-descriptors` | `1.0.2` | registry |
| `has-symbols` | `1.1.0` | registry |
| `has-tostringtag` | `1.0.2` | registry |
| `hash-base` | `3.1.2` | registry |
| `hashlru` | `2.3.0` | registry |
| `hasown` | `2.0.3` | registry |
| `i` | `0.3.7` | registry |
| `ieee754` | `1.2.1` | registry |
| `inherits` | `2.0.4` | registry |
| `ioredis` | `5.10.1` | registry |
| `ip-address` | `10.3.1` | registry |
| `is-arguments` | `1.2.0` | registry |
| `is-buffer` | `2.0.5` | registry |
| `is-callable` | `1.2.7` | registry |
| `is-extglob` | `2.1.1` | registry |
| `is-generator-function` | `1.1.2` | registry |
| `is-glob` | `4.0.3` | registry |
| `is-regex` | `1.2.1` | registry |
| `is-typed-array` | `1.1.15` | registry |
| `isarray` | `1.0.0` | registry |
| `isarray` | `2.0.5` | registry |
| `iso-url` | `1.2.1` | registry |
| `isomorphic-ws` | `4.0.1` | registry |
| `jiti` | `2.7.0` | registry |
| `js-tokens` | `9.0.1` | registry |
| `kubernetes-types` | `1.30.0` | registry |
| `level` | `10.0.0` | registry |
| `level` | `8.0.1` | registry |
| `level-supports` | `4.0.1` | registry |
| `level-supports` | `6.2.0` | registry |
| `level-transcoder` | `1.0.1` | registry |
| `libsodium-sumo` | `0.8.4` | registry |
| `libsodium-wrappers-sumo` | `0.8.4` | registry |
| `lightningcss` | `1.32.0` | registry |
| `lightningcss-android-arm64` | `1.32.0` | registry |
| `lightningcss-darwin-arm64` | `1.32.0` | registry |
| `lightningcss-darwin-x64` | `1.32.0` | registry |
| `lightningcss-freebsd-x64` | `1.32.0` | registry |
| `lightningcss-linux-arm-gnueabihf` | `1.32.0` | registry |
| `lightningcss-linux-arm64-gnu` | `1.32.0` | registry |
| `lightningcss-linux-arm64-musl` | `1.32.0` | registry |
| `lightningcss-linux-x64-gnu` | `1.32.0` | registry |
| `lightningcss-linux-x64-musl` | `1.32.0` | registry |
| `lightningcss-win32-arm64-msvc` | `1.32.0` | registry |
| `lightningcss-win32-x64-msvc` | `1.32.0` | registry |
| `lodash` | `4.18.1` | registry |
| `lodash.defaults` | `4.2.0` | registry |
| `lodash.isarguments` | `3.1.0` | registry |
| `loupe` | `3.2.1` | registry |
| `lru-cache` | `10.1.0` | registry |
| `lru-cache` | `11.3.6` | registry |
| `magic-string` | `0.30.21` | registry |
| `main-event` | `1.0.4` | registry |
| `math-intrinsics` | `1.1.0` | registry |
| `maybe-combine-errors` | `1.0.0` | registry |
| `md5.js` | `1.3.5` | registry |
| `memory-level` | `3.1.0` | registry |
| `midgard-ts` | `0.1.0` | workspace-source |
| `mime` | `3.0.0` | registry |
| `minimatch` | `10.2.5` | registry |
| `minipass` | `7.1.3` | registry |
| `module-error` | `1.0.2` | registry |
| `ms` | `2.1.3` | registry |
| `msgpackr` | `1.11.12` | registry |
| `msgpackr-extract` | `3.0.3` | registry |
| `multiformats` | `13.4.2` | registry |
| `multipasta` | `0.2.7` | registry |
| `nanoassert` | `2.0.0` | registry |
| `nanoid` | `3.3.18` | registry |
| `napi-macros` | `2.2.2` | registry |
| `node-addon-api` | `7.1.1` | registry |
| `node-fetch` | `2.7.0` | registry |
| `node-gyp-build` | `4.8.4` | registry |
| `node-gyp-build-optional-packages` | `5.1.1` | registry |
| `node-gyp-build-optional-packages` | `5.2.2` | registry |
| `obuf` | `1.1.2` | registry |
| `p-queue` | `9.2.0` | registry |
| `p-timeout` | `7.0.1` | registry |
| `package-json-from-dist` | `1.0.1` | registry |
| `path-scurry` | `2.0.2` | registry |
| `pathe` | `2.0.3` | registry |
| `pathval` | `2.0.1` | registry |
| `pbkdf2` | `3.1.5` | registry |
| `pg` | `8.20.0` | registry |
| `pg-cloudflare` | `1.3.0` | registry |
| `pg-connection-string` | `2.12.0` | registry |
| `pg-connection-string` | `2.9.1` | registry |
| `pg-cursor` | `2.19.0` | registry |
| `pg-int8` | `1.0.1` | registry |
| `pg-numeric` | `1.0.2` | registry |
| `pg-pool` | `3.13.0` | registry |
| `pg-protocol` | `1.13.0` | registry |
| `pg-types` | `2.2.0` | registry |
| `pg-types` | `4.1.0` | registry |
| `pgpass` | `1.0.5` | registry |
| `picocolors` | `1.1.1` | registry |
| `picomatch` | `4.0.4` | registry |
| `possible-typed-array-names` | `1.1.0` | registry |
| `postcss` | `8.5.23` | registry |
| `postgres-array` | `2.0.0` | registry |
| `postgres-array` | `3.0.4` | registry |
| `postgres-bytea` | `1.0.1` | registry |
| `postgres-bytea` | `3.0.0` | registry |
| `postgres-date` | `1.0.7` | registry |
| `postgres-date` | `2.1.0` | registry |
| `postgres-interval` | `1.2.0` | registry |
| `postgres-interval` | `3.0.0` | registry |
| `postgres-range` | `1.1.4` | registry |
| `process-nextick-args` | `2.0.1` | registry |
| `progress-events` | `1.1.0` | registry |
| `pure-rand` | `6.1.0` | registry |
| `queue-microtask` | `1.2.3` | registry |
| `readable-stream` | `2.3.8` | registry |
| `redis-errors` | `1.2.0` | registry |
| `redis-parser` | `3.0.0` | registry |
| `reusify` | `1.1.0` | registry |
| `rimraf` | `6.1.3` | registry |
| `ripemd160` | `2.0.3` | registry |
| `rolldown` | `1.1.2` | registry |
| `run-parallel-limit` | `1.1.0` | registry |
| `rxjs` | `7.8.2` | registry |
| `safe-buffer` | `5.1.2` | registry |
| `safe-buffer` | `5.2.1` | registry |
| `safe-regex-test` | `1.1.0` | registry |
| `serialize-error` | `8.1.0` | registry |
| `set-function-length` | `1.2.2` | registry |
| `sha.js` | `2.4.12` | registry |
| `siginfo` | `2.0.0` | registry |
| `source-map-js` | `1.2.1` | registry |
| `split2` | `4.2.0` | registry |
| `stackback` | `0.0.2` | registry |
| `standard-as-callback` | `2.1.0` | registry |
| `std-env` | `3.10.0` | registry |
| `string_decoder` | `1.1.1` | registry |
| `strip-literal` | `3.1.0` | registry |
| `tinybench` | `2.9.0` | registry |
| `tinyexec` | `0.3.2` | registry |
| `tinyglobby` | `0.2.16` | registry |
| `tinyglobby` | `0.2.17` | registry |
| `tinypool` | `1.1.1` | registry |
| `tinyrainbow` | `2.0.0` | registry |
| `tinyspy` | `4.0.4` | registry |
| `to-buffer` | `1.2.2` | registry |
| `tr46` | `0.0.3` | registry |
| `ts-custom-error` | `3.3.1` | registry |
| `ts-log` | `2.2.7` | registry |
| `tslib` | `2.8.1` | registry |
| `type-fest` | `0.20.2` | registry |
| `typed-array-buffer` | `1.0.3` | registry |
| `uint8-varint` | `2.0.5` | registry |
| `uint8arraylist` | `2.4.9` | registry |
| `uint8arrays` | `5.1.1` | registry |
| `undici` | `8.10.0` | registry |
| `undici-types` | `6.21.0` | registry |
| `utf8-codec` | `1.0.0` | registry |
| `util` | `0.12.5` | registry |
| `util-deprecate` | `1.0.2` | registry |
| `uuid` | `11.1.1` | registry |
| `vite` | `8.1.0` | registry |
| `vite-node` | `3.2.4` | registry |
| `vitest` | `3.2.6` | registry |
| `web-encoding` | `1.1.5` | registry |
| `webidl-conversions` | `3.0.1` | registry |
| `whatwg-url` | `5.0.0` | registry |
| `which-typed-array` | `1.1.20` | registry |
| `why-is-node-running` | `2.3.0` | registry |
| `ws` | `7.5.11` | registry |
| `ws` | `8.21.0` | registry |
| `xtend` | `4.0.2` | registry |
| `yaml` | `2.9.0` | registry |
