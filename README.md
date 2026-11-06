# ERC-8415: Asynchronous Register Projection for NFTs

**Proposal status:** Draft<br>
**ERC number:** 8415<br>
**License:** CC0-1.0

ERC-8415 lets an NFT follow an asset whose ownership is decided by an external register (a warehouse register, land registry, custody ledger or securities register) without pretending the chain is that register. The register stays authoritative. The chain records what the register has confirmed, when it took effect, whether that answer is settled, and whether a change is still in progress.

This repository contains the working draft, Solidity interfaces, a reference implementation and local-EVM tests. It is a standards proposal repository, not a product repository, and it does not represent an adopted Ethereum standard.

- [Upstream proposal: ethereum/ERCs PR #2006](https://github.com/ethereum/ERCs/pull/2006)
- [Upstream proposal text pinned for this README](https://github.com/GiraffeTechnology/ERCs/blob/12e68aee54c6f94f6910d4db42f1d5677b506de1/ERCS/erc-8415.md)
- [Ethereum Magicians discussion](https://ethereum-magicians.org/t/erc-8415-asynchronous-register-projection-for-nfts/29634)
- [ERC8415-Kit: integration components, SDKs and examples](https://github.com/GiraffeTechnology/ERC8415-Kit)

## Why this exists

Most real-world asset tokens today treat the token as the asset: move the token and you move the right. For many assets that is not how ownership works. Title to goods in a warehouse, a registered security or a parcel of land passes when the register records the change, and registers do not update in the same block as a transfer.

The common workaround is to place the asset in an intermediate company and tokenize the company's shares. The buyer then holds a claim on that company rather than on the asset, and the register that legally decides ownership is no longer the one the token follows.

ERC-8415 takes the other path: keep the asset in its register, let the token trade, and make the register's confirmed state readable on chain with explicit timing and finality.

### A worked example

The example below is illustrative. It describes technical coordination only and implies no relationship with any exchange, warehouse operator or registry.

Trader A holds a warehouse receipt for 25 tonnes of copper and sells it to fund B. B later sells to C.

| Step | On chain (`ownerOf`) | Register (`holderAsOf`) | What a consumer can tell |
| --- | --- | --- | --- |
| A lists the receipt | A | A | The projection's entry for A is the starting point |
| A sells to B; a registration change is requested | B | A | A gap is open: a change is in progress and not yet admitted |
| B resells to C before the register catches up | C | A | Trading is not blocked; the register still names A |
| The register records B | C | B from its effective time | The A to B change is admitted; instants before it are now final |
| The register records C | C | C from its effective time | The B to C change is admitted |

Two things matter in this sequence:

- **The token and the register can disagree, and that is expected.** `ownerOf` is the tradeable position. `holderAsOf(tokenId, t)` is who the register recognised at time `t`. An application that pays out, releases escrow or accepts collateral reads the second, not the first.
- **A late or failed registration is visible, not hidden.** While a change is pending, `openGapOf` says so. If a registration never arrives, the projection keeps naming the last confirmed holder and leaves recent instants non-final, instead of reporting a transfer that did not happen.

## What the standard adds

For any token, a consumer can ask three separate questions:

| Question | Function | Answer |
| --- | --- | --- |
| Who did the register recognise at time `t`? | `holderAsOf(tokenId, t)` | The holder of the entry covering `t` |
| Is that answer settled? | `isFinalAsOf(tokenId, t)` | `true` only when a later entry has closed the interval containing `t` |
| Is a change in progress right now? | `openGapOf(tokenId)` | The open settlement ID, or zero |

These are kept apart on purpose. A record-date payout needs a final answer for a past instant. A release of funds may also need to know that no change is in flight. Collapsing them into one boolean would force every application into the same policy.

## Where it sits among RWA standards

| Layer | Question it answers | Examples | Relation to ERC-8415 |
| --- | --- | --- | --- |
| Eligibility | Who may hold or receive the token | ERC-3643, ERC-1450, ERC-7943 | Composable: eligibility decides who may buy; ERC-8415 says whether the register has confirmed them |
| Asset state | What state the asset is in, and who confirms it | ERC-8392, **ERC-8415** | Complementary: both separate an expected delay from a real failure |

ERC-8415 does not replace eligibility or compliance standards, and an asset can use both. It also does not replace intermediate-company or custodian-issued token structures where those suit the asset; it gives assets that already live in an authoritative register another option.

## Design principles

### 1. Token state and register state are separate

```text
Tradeable token position  !=  Register-confirmed historical state
```

The token remains transferable while register synchronisation is pending.

### 2. External registers keep their boundaries

The proposal does not move registers on chain and does not replace legal or institutional sources of truth. It makes admitted register history queryable under an explicit verification profile, whose authorities and trust assumptions remain part of the security boundary.

### 3. Historical semantics are explicit

The proposal separates historical resolution (`holderAsOf`), finality (`isFinalAsOf`) and unresolved transitions (`openGapOf`, when the optional settlement interface is supported).

## Interfaces

The [reference contract](reference/RegisterProjectionReference.sol) implements ERC-721 and both interfaces below on the same contract. Consumers discover each interface independently through ERC-165.

| Interface | ERC-165 identifier | Surface |
| --- | --- | --- |
| [`IRegisterProjection`](interfaces/IRegisterProjection.sol) | `0x6309e170` | `currentEntry`, `entryAt`, `entryAsOf`, `holderAsOf`, `isFinalAsOf`, `entryCount`, `registerId` |
| [`IProjectionSettlement`](interfaces/IProjectionSettlement.sol) | `0xf4a7d71b` | `settlement`, `openGapOf`, `settlementPeriod`, `verificationProfile`, `isSettlementAuthority`, `beginSettlement`, `finalizeSettlement`, `cancelSettlement` |

Projection entries are append-only, versions start at one, effective times strictly increase, and record commitments cannot repeat within a token. A new entry links the previous commitment and closes the previous interval at the new entry's `effectiveAt`. The standard does not require commitment or registry-reference uniqueness across different tokens.

### Historical resolution and finality

`entryAsOf(tokenId, t)` selects the entry covering `t`; `holderAsOf` returns its holder. Queries before the first entry revert. For an existing token, `isFinalAsOf` returns `false` before the first entry and is `true` exactly when:

```text
firstEntry.effectiveAt <= t < latestEntry.effectiveAt
```

An instant at or after the latest effective time remains provisional. Admitting a later confirming entry with the same holder can make earlier instants final; merely closing or cancelling a gap cannot. All instants use Unix seconds.

This is finality of the projection's historical answer. It is not a block-depth test, an assertion that `ownerOf` matches the registered holder, or proof of legal title.

### Optional settlement

The proposal permits projection-only implementations. Settlement conformance additionally requires projection conformance and every settlement and proof requirement; advertising the settlement interface alone is not sufficient.

The reference implementation uses an immutable registrar as settlement authority, independently of ERC-721 ownership:

- `beginSettlement` opens a gap with an unused nonzero ID, a nonzero snapshot, an expected holder and a future deadline within the 30-day settlement period. Opening another gap supersedes the earlier one without changing the projection.
- Anyone can relay a valid proof to `finalizeSettlement`. Successful admission atomically appends an entry and closes the gap. Failed admission reverts all effects.
- Only the recorded initiator can cancel an open gap, strictly after its deadline. Cancellation leaves projection history unchanged.
- `openGapOf` returns zero when no gap is open; `settlement(unknownId)` reverts. A gap contests instants at or after its `openedAt`, while historical finality is determined independently.
- Ordinary ERC-721 transfers do not update the projection and remain possible while a gap is open.

Settlement here records proof-verified admission. It does not implement escrow, payments, refunds, recovery of transferred NFTs or application entitlement rules.

### Reference verification profile

The reference uses a fixed validator quorum, EIP-712 signatures and tagged Keccak Merkle membership proofs. Proofs bind the destination chain and contract, token, settlement, expected holder, snapshot, linked commitments, registry reference, version and effective time. Remote heights increase per token and consumed proofs cannot be reused. Initialisation is an administrator-authorised `mint`, and subsequent admissions use settlement proofs.

This is a demonstration profile, not a production light client or a security audit. Its validator set and registrar are immutable, validator equivocation across tokens is not detected, and repeated supersession is not bounded. The profile caps future effective times at 30 days ahead. Read the [reference contract's trust assumptions](reference/RegisterProjectionReference.sol) and [security considerations](SECURITY.md) before adapting it.

### Consumer integration

Applications should re-read and validate the required holder, finality and gap conditions on chain within the same transaction as the dependent action. Off-chain reads can inform a UI but do not guarantee the state when a later transaction executes; two transactions in the same block do not close that window.

The [ERC8415-Kit](https://github.com/GiraffeTechnology/ERC8415-Kit) contains integration components, SDKs and examples. Its [RecordDateClaim example](https://github.com/GiraffeTechnology/ERC8415-Kit/blob/main/contracts/examples/RecordDateClaim.sol) and [pinned local-EVM tests](https://github.com/GiraffeTechnology/ERC8415-Kit/blob/93035d2c1384dbb6b974a9680326fbff623c2d9b/tests/onchain/recordDateClaim.onchain.cjs) demonstrate same-transaction checks and one-time record-date payouts. Their payout and gap-rejection policies are application choices, not extra ERC requirements.

## Scope and non-goals

This proposal does not:

- determine legal ownership;
- replace government or institutional registers;
- establish jurisdiction-specific rights;
- make the blockchain the legal source of truth;
- define application-specific entitlement, escrow or payment rules.

A proof establishes inclusion in an accepted remote state. It does not independently prove the factual correctness of the underlying register.

## Repository structure

```text
ERC-8415/
├── ERCS/erc-8415-asynchronous-register-projection.md   spec text, ethereum/ERCs form
├── EIPS/eip-8415.md                                    spec text, ethereum/EIPs form
├── interfaces/
│   ├── IRegisterProjection.sol                         projection queries
│   └── IProjectionSettlement.sol                       settlement lifecycle
├── reference/RegisterProjectionReference.sol           reference implementation
├── test/protocol.cjs                                   invariant tests
├── watchtower/                                         non-normative addon, separate toolchain
├── scripts/                                            lint, verification and build scripts
├── RATIONALE.md                                        why the design is shaped this way
├── COMPARISON.md                                       how it differs from adjacent standards
├── SECURITY.md                                         trust assumptions and attack surface
└── README.md
```

The local draft is kept in two places, `ERCS/` and `EIPS/`. Both are byte-identical copies of the head of upstream PR #2006 (`12e68ae`), which is the authoritative text. They are not synchronised automatically; later upstream revisions must be copied in by hand. The resubmission copy under `submission/` differs only in the author handle.

### The `watchtower/` directory

[`watchtower/`](watchtower/README.md) is a separate, non-normative freshness addon, not a required ERC-8415 interface. Its registry accepts EIP-712 signed watchtower attestations, enforces strictly monotonic sequencing per asset feed, honours block-scoped signing-key rotation, and classifies the recorded head as `STALE`, `FRESH_PENDING` or `FRESH_FINAL` (or `UNKNOWN` when no head is available).

`FRESH_FINAL` means a fresh attestation has reached the configured block-age threshold; it does not prove consensus finality or the ERC's historical `isFinalAsOf` condition. The current API exposes the attestation type hash and EIP-712 domain, not a raw type-string getter. `rotateKey` and `revokeKey` are separate calls; there is no atomic rotate-and-revoke operation.

It carries its own Foundry and Hardhat setup and is built and tested from its own directory. The root `compile` and `test` do not reach into it; the repository-wide `lint`, `check-imports` and `secret-scan` checks do cover it.

## Building and verifying

The root CI uses Node 20 and the committed npm lockfile. From the repository root:

```sh
npm ci
npm run verify:all
```

`verify:all` runs the following sequence. The first interface compilation needs access to Hardhat's Solidity compiler distribution unless the compiler is already cached.

| Step | What it checks |
| --- | --- |
| `lint` | SPDX identifier and pragma on Solidity; LF endings, no tabs or trailing whitespace, and trailing newline in checked source, configuration and Markdown files; ASCII-only Solidity |
| `compile` | Compiles `interfaces/` with solc 0.8.26 and flattens the artifacts |
| `verify:constants` | Recomputes each interface ID from the compiled ABI and compares it against the frozen value |
| `verify:secret-scan` | Checks files against the publication allowlist and scans for private-key headers and GitHub token patterns; not an exhaustive secret detector |
| `verify:imports` | No absolute imports, and no relative import escapes the package |
| `test` | Compiles the reference contract with Hardhat's bundled solc and runs its local-EVM invariant tests |
| `build` | Writes `dist/abi/`, the local `EIPS/eip-8415.md` copy, and `dist/manifest.json` with interface IDs and SHA-256 hashes of the generated files |

CI runs the same sequence on every push to `main` and every pull request. `verify:constants` checks the frozen IDs above by XOR of each interface's function selectors, excluding inherited `supportsInterface`.

To run only the reference-contract tests after `npm ci`:

```sh
npm test
```

This uses `hardhat test --no-compile`; the test file compiles the standalone reference itself. The suite runs locally without a wallet, RPC endpoint, testnet funds or external signer. Passing it does not validate a production deployment or the separate watchtower implementation.

To build and test the watchtower addon with its own locked dependencies and solc 0.8.28:

```sh
cd watchtower
npm ci
npm run compile
npm test
```

See [watchtower/README.md](watchtower/README.md) for its alternative Foundry workflow. The root CI does not run either watchtower test suite.

## Feedback

Feedback is welcome on:

- historical query semantics and finality definitions;
- projection gap handling and settlement authority;
- verification profiles and attestation trust;
- what lending, escrow and agent tooling need to read before acting;
- interoperability with eligibility and asset-state standards.

Discuss the draft in the [Ethereum Magicians thread](https://ethereum-magicians.org/t/erc-8415-asynchronous-register-projection-for-nfts/29634), or report reproducible implementation issues in this repository with the commit, command, and expected versus observed behaviour.

## License

[CC0-1.0](LICENSE).
