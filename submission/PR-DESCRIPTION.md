# ERC-8415: Asynchronous Register Projection for NFTs

This PR resubmits ERC-8415 under its assigned number from a new account. The original PR, ethereum/ERCs#2006, was opened from `GiraffeTechnology`, an account that has since been permanently closed. Same author (Michael Yip), same proposal.

## Why this PR keeps the number 8415

ERC-8415 is the number assigned to this proposal in #2006; it was applied to the file in commit `c9bf1a5` ("Assign ERC-8415 and update discussions-to link"). This PR replaces #2006 and is not a new proposal, so it keeps that number. The Ethereum Magicians thread (`discussions-to`), the published interface identifiers and existing references to the proposal all use 8415. Changing the number would split one proposal's history across two numbers. If the editors still prefer a fresh number, I will rename the file and update `eip:` on request.

## What changed relative to #2006

Nothing. `ERCS/erc-8415.md` is byte-identical to the head of #2006 (`12e68aee54c6f94f6910d4db42f1d5677b506de1`), including the original `author` line. This PR is opened from the author's new account, `GiraffeJP`, because the account named in that line can no longer be used.

Discussion: https://ethereum-magicians.org/t/erc-8415-asynchronous-register-projection-for-nfts/29634

Implementation links are kept here rather than in the proposal, following the external-link rule:

- CC0 reference implementation: https://github.com/GiraffeJP/ERC-8415/blob/7d27bf46f7140c41783860886a013a8ca5ac0f10/reference/RegisterProjectionReference.sol
- Local-EVM test suite (no wallet, RPC, testnet asset or external signer needed): https://github.com/GiraffeJP/ERC-8415/blob/7d27bf46f7140c41783860886a013a8ca5ac0f10/test/protocol.cjs
