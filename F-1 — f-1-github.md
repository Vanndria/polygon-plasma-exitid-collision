# F-1 — Polygon Plasma exitId Collision: Two Withdraw Logs, One exitId, Permanent Freeze

> **Bug:** `WithdrawManager.verifyInclusion()` computes `exitId` without `logIndex`, so two `Withdraw` logs in the same Polygon (child-chain) receipt produce an identical `exitId`. The second burn can never exit — its root tokens stay locked in `DepositManager` forever.
>
> **Contract:** WithdrawManager proxy `0x2A88696e0fFA76bAA1338F2C74497cC013495922` (Ethereum mainnet)
> **Implementation (verified on-chain):** `0x6f8a42cf6f3ce657b66a9d5849f1251de7a35168` (Sourcify `exact_match`, solc 0.5.17)
> **Severity submitted:** High — permanent freezing of funds (fix requires hardfork)
> **Status:** end-to-end PoC verified (Foundry fork test, 4/4 pass, real checkpoint + real receipt). One honest gap: no real 2-log receipt has been found in the wild yet — that link is proven by the unit PoC instead of the fork test. See the report below.

---
## Report

# DRAFT — Immunefi report for F-1 (DO NOT SUBMIT; submission needs Van's explicit approval)

> Status: DRAFT. The end-to-end PoC has run and passed (see `work-e2e/README.md`
> for the exact run). Fields below are filled from that run. Do not submit this
> text until Van approves.

## Title
Plasma exitId collision: two `Withdraw` logs in one child-chain receipt share the same exitId, permanently freezing the second burn

## Summary
`WithdrawManager.verifyInclusion` derives an exit's identity (`age`) purely from
receipt-level data: the checkpoint creation time, the child block number, and the
receipt proof's branch mask. The receipt's `logIndex` — which log inside the
receipt is being exited — never enters the computation, even though the code's own
comment documents the low 32 bits of the exitId as
"receiptPos + logIndex * MAX_LOGS + oIndex".

Consequence: two `Withdraw` events emitted by the SAME child-chain receipt (e.g.
two burns of the same or different tokens in one transaction, or a batch burn)
produce the IDENTICAL `exitId`. The first exit is recorded in `exits[exitId]` and
`isKnownExit[exitId]`. `exits[exitId]` is never deleted (`// delete exits[exitId];`
is commented out in `processExits`), so `_addExitToQueue` rejects the second exit
with `EXIT_ALREADY_EXISTS` / `KNOWN_EXIT` — permanently. The second burner's root
tokens remain locked in `DepositManager` forever, with no code path to recover them.

Timing note (verified on the fork): the full `exitId` is time-dependent because
`exitableAt = max(createdAt + 2*HALF_EXIT_PERIOD, now + HALF_EXIT_PERIOD)` and the
deployed `HALF_EXIT_PERIOD` reads 1, so `exitableAt = now + 1`. But the freeze
does not depend on timing: `isKnownExit` is keyed by `uint128(exitId)`, which is
`(blockNumber << 33) | (branchMask << 1)` — purely receipt-dependent. A second
exit at the same timestamp reverts `EXIT_ALREADY_EXISTS`; at any later timestamp
it reverts `KNOWN_EXIT`. Both layers were demonstrated on the fork.

This is the mirror image of the 2021 Wagner finding ($2M, one log → many exitIds,
double-spend). F-1 is the opposite direction: many logs → one exitId, a permanent
freeze rather than a double-spend.

## Vulnerability details

Affected contract (deployed, Ethereum mainnet):
- `WithdrawManager` proxy `0x2A88696e0fFA76bAA1338F2C74497cC013495922`
  → implementation `0x6f8a42cf6f3ce657b66a9d5849f1251de7a35168`
  (Sourcify v2 exact_match on creation and runtime bytecode; source = pos-contracts
  v2.0.0 minus two dead RLPReader import lines)
- Relevant function: `verifyInclusion` (in `WithdrawManager.sol`), which returns
  `(getExitableAt(createdAt) << 127) | (blockNumber << 32) | branchMask`.
  Every input is receipt-level: `createdAt` (checkpoint time), `blockNumber`,
  `branchMask` (the MPT path of the receipt's transaction index). `logIndex` is
  used only by the predicate to select which log to decode, never by the ID.
- `ERC20PredicateBurnOnly.startExitWithBurntTokens` (and the ERC721/WETH
  predicates) pass `age << 1` as `exitId` into `_addExitToQueue`, which reverts
  when `isKnownExit[exitId]` is already set.
- `processExits` finalizes an exit by burning its ExitNFT and releasing root
  tokens; it never clears `exits[exitId]` or `isKnownExit[exitId]` (the
  `// delete exits[exitId];` line is commented out), so the collision is
  permanent even after the first exit is fully processed.

## Steps to reproduce (end-to-end, local mainnet fork)

Run in `work-e2e/` (see `README.md` there for the exact commands):
`forge test --fork-url <ETH_MAINNET_RPC> --fork-block-number 26434270 --match-test test_e2eCollision -vvv`
— the fork block is one block before the real exit below, so the checkpoint
exists and the exit does not yet.

1. Take a real Polygon PoS receipt containing a `Withdraw` log:
   tx `0x50d6255b57cea5d9b8fd9e3f81a40786ecf25e4cd8140997ccc7a11371e597b8`,
   block **97,470,272**, txIndex 112. The receipt has 7 logs; the Withdraw log
   is at receipt-logIndex **2** (emitter the native MATIC child token
   `0x0000000000000000000000000000000000001010`, rootToken MATIC
   `0x7d1afa7b718fb893db30a3abc0cfc608aacfebb0`, from aggregator
   `0x67a40d016efe809a5bfcd942a8faf2d9cf0758e2`, ~112.7 MATIC). The receipt was
   REALLY exited on Ethereum (ExitStarted, ETH block 26434271, exitId
   `610777087527273636638742913597695220376890507488`).
2. Build the standard Plasma exit payloads for logIndex 2 and logIndex 3
   (same receipt, same receipt proof, same block proof, same checkpoint header
   **1171170000**, root
   `0x4ef064376b83ab7a976043abce60c451956454bf2a0f8ec7298edcda5e4c091a`;
   only the logIndex field differs). Both payloads' receipt proof and header
   proof were rebuilt from real chain data and their roots re-verified
   (receipt-trie root = block receiptsRoot; header-tree root = checkpoint
   headerRoot).
3. On a local Ethereum mainnet fork, call the deployed WithdrawManager's
   `verifyInclusion` with each payload: both return the same `age`
   (**305388541721942616793740676018599965597835985008**), hence the same
   `exitId` (**610777083443885233587481352037199931195671970016** at the fork
   block's timestamp; the on-chain exit used exitId
   `610777087527273636638742913597695220376890507488`, differing only in the
   time-dependent exitableAt bits — the receipt-dependent low 128 bits are
   identical).
4. As the log's withdrawer, call the deployed ERC20 predicate's
   `startExitWithBurntTokens` with payload 0: succeeds, ExitNFT minted for exitId.
5. Re-attempt the same exit at the same timestamp: reverts with
   `EXIT_ALREADY_EXISTS` (the exit record exists under that exitId).
6. Warp 30 days ahead and call `processExits(rootToken)`: the first exit
   finalizes (NFT burned, tokens released).
7. Retry the twin exit at the later timestamp: it STILL reverts, now with
   `KNOWN_EXIT` — the full exitId changed with `now`, but `uint128(exitId)`
   (blockNumber + branchMask) is time-independent and `isKnownExit` is never
   cleared. The second burn is frozen forever, before and after finalization.

Scope note (honest): no receipt with TWO Withdraw logs could be found in the
wild despite an extensive search (~2M blocks for WMATIC with 0 Withdraw logs;
120,000 blocks for the native MATIC token with 5 single-log exits and no
2-log receipt; see `data/FINDINGS.md`). So step 5 above demonstrates the
collision's second leg by re-attempting the same exit, and the genuine
two-log case (a second Withdraw log being accepted and then rejected with
`EXIT_ALREADY_EXISTS`) is proven at unit level (3/3 Foundry tests against
the real WithdrawManager + PriorityQueue + ExitNFT behind the real proxy,
mocks only for Registry/predicate wiring) in the repository's test suite.
The root cause — `verifyInclusion` ignoring `logIndex` — is proven on the
real contracts with real data in steps 3–7 above, and it is timing-independent
in the `isKnownExit` (uint128) layer: any second log in the same receipt maps
to the same uint128 key at ANY timestamp.

## Impact
- Any user (or aggregator / batching contract) whose transaction emits two or
  more `Withdraw` logs in a single child-chain receipt permanently loses the
  ability to exit all but the first burn. The root tokens of the second (and any
  further) burn are locked in `DepositManager` with no recovery path in the
  current code.
- The precondition is realistic: a single transaction can burn twice (two
  `burn()` calls, a batch burn, or a contract that burns on behalf of two users),
  and the child chain will place both `Withdraw` logs in one receipt.
- Loss is bounded per receipt to the value of the second-and-later burns in that
  receipt (not the whole bridge), which is why this is High rather than Critical.

## Severity: High
- Impact: High — permanent loss of user funds (root tokens locked forever), no
  recovery path.
- Likelihood: Medium — requires two burns in one receipt, which is uncommon for
  ordinary single-burn users but entirely reachable for aggregators, batching
  contracts, or any transaction that burns twice; no attacker action is needed,
  the victim harms themselves by ordinary use.
- Per Immunefi's scale: High = $10,000 flat (Critical would require a
  high-likelihood total bridge compromise, which this is not).

## Recommendation
Include `logIndex` (and, for defence in depth, the log's emitting child-token
address) in the exit identity. For example, change `verifyInclusion` to return
`age` with the low 32 bits set to
`receiptPos + logIndex * MAX_LOGS + oIndex` — exactly what the existing code
comment already documents — or equivalently fold `logIndex` into the branch
mask computation. This makes each log's exitId unique while keeping all existing
single-log exits unchanged (their logIndex is already known to the predicate).
Additionally, consider clearing `isKnownExit`/`exits` on finalization so a
stale record can never block a future exit, and consider a migration/allowlist
path for any exits already frozen by this collision.

## References
- pos-contracts v2.0.0, `contracts/root/withdrawManager/WithdrawManager.sol`
  (`verifyInclusion`, `_addExitToQueue`, `processExits`), and
  `contracts/root/predicates/ERC20PredicateBurnOnly.sol`
  (`startExitWithBurntTokens`, `verifyDeprecation`).
- Deployed bytecode verified via Sourcify v2 (exact_match, creation + runtime).
- Related prior finding (opposite direction): 2021 Wagner report — one log
  producing many exitIds (double-spend), fixed by binding the exit to a single
  receipt position. F-1 shows the remaining gap: the log position inside the
  receipt is still not bound.
- Prior public discussion (disclosed honestly): a blog post dated 20 Oct 2025
  ("Polygon Vulnerability: Burn Exit ID Collision Explained",
  bijuhanta.web.id — the site is now offline, DNS does not resolve) describes
  the same root cause (logIndex omitted from the exitId). This report does not
  claim discovery of the root cause from scratch; its new contribution is a
  runnable end-to-end proof against the currently deployed contracts with real
  checkpoint and receipt data (steps 3–7 above), plus verification that the
  issue is still live on mainnet and is not listed in the program's known
  issues. The blog provides no runnable PoC.

---

## PoC Guide

# F-1 — End-to-End PoC: Plasma exitId collision (Polygon)

Proves, against the REAL deployed Polygon contracts on a local Ethereum mainnet
fork with REAL checkpoint data, that two `Withdraw` logs in the SAME Polygon PoS
receipt produce the IDENTICAL exitId, so the second burn can never be exited —
its root tokens stay locked in `DepositManager` forever.

**Status: runnable once `data/fixture.json` exists (built from real chain data).**
**Nothing here submits anything to Immunefi, and nothing touches mainnet state:
all reads are read-only RPC calls; all execution is local (Foundry fork).**

## The bug in one paragraph

`WithdrawManager.verifyInclusion` returns
`(getExitableAt(createdAt) << 127) | (blockNumber << 32) | branchMask`.
Every input is receipt-level: checkpoint time, child block number, receipt-proof
branch mask. The receipt's `logIndex` never enters the ID — even though the
code's own comment documents the low 32 bits as
"receiptPos + logIndex * MAX_LOGS + oIndex". The predicate passes `age << 1` as
the `exitId` and uses `logIndex` only to pick which log to decode. `exits[exitId]`
and `isKnownExit` are never deleted (`// delete exits[exitId];` is commented out
in `processExits`), so `_addExitToQueue` rejects the second exit with
`EXIT_ALREADY_EXISTS` / `KNOWN_EXIT` — permanently.

Deployed (Ethereum mainnet, verified read-only, Sourcify v2 exact_match):
- WithdrawManager proxy `0x2A88696e0fFA76bAA1338F2C74497cC013495922`
  → implementation `0x6f8a42cf6f3ce657b66a9d5849f1251de7a35168`
- ERC20 predicate `0x4EeA1780c06709D7FA0BCaA6D0f1aB29673586c0`
  (contract `ERC20PredicateBurnOnly`, Sourcify exact_match)
- RootChain proxy `0x86E4Dc95c7FBdBf52e33D563BbDB00823894C287`

## Files

| File | What it is |
|---|---|
| `proofs.py` | Pure-Python MPT (receipt trie), HP encode/decode, receipt RLP (incl. EIP-1559 type prefix), branch mask, header Merkle tree, 12-field exit-payload builder, Python replica of `MerklePatriciaProof.verify`. Self-tested (see NOTES.md). |
| `build_fixture.py` | Builds `data/fixture.json` from real chain data; self-checks the recomputed receipt-trie root against the block's `receiptsRoot` and the recomputed header-tree root against the checkpoint `headerRoot`. |
| `test/F1E2E.t.sol` | The fork test. Reads `data/fixture.json`, calls the real deployed contracts. |
| `data/` | Real chain data: `receipt.json`, `block.json`, `block_receipts.json`, `checkpoint.json`, `fixture.json`, `FINDINGS.md` (provenance). |
| `DRAFT_REPORT.md` | Draft Immunefi report text (DO NOT SUBMIT — needs Van's approval). |
| `NOTES.md` | Working notes: what is proven, what is real vs constructed. |

## Run it

Prerequisites: `forge` (at `~/.foundry/bin/forge`), Python 3 with `rlp` +
`eth_hash` installed. A public Ethereum mainnet RPC that answers (endpoints are
flaky — retry across them).

```bash
export PATH="$HOME/.foundry/bin:$PATH"
cd ~/workspace/bugbounty/polygon/work-e2e

# 1. Build the fixture from the real chain data (already in data/).
#    Rebuilds the receipt MPT proof + header Merkle proof and verifies both
#    roots against the block / checkpoint. Fails loudly on any mismatch.
python3 build_fixture.py

# 2. Run the fork test. The fork block MUST be 26434270: the fixture's receipt
#    was really exited on mainnet at ETH block 26434271, so a fork at "latest"
#    already contains that exit (first exit reverts KNOWN_EXIT). At 26434270
#    the checkpoint exists and the exit does not yet.
forge test --fork-url https://eth.drpc.org --fork-block-number 26434270 --match-test test_e2eCollision -vvv
# If the RPC is flaky, retry with https://ethereum-rpc.publicnode.com
```

## Expected output

Verified run (2026-11-17 ~21:10 WIB, `forge test --fork-url https://eth.drpc.org
--fork-block-number 26434270 --match-test test_e2eCollision -vvv`; also passes
on `https://ethereum-rpc.publicnode.com`):

```
Ran 1 test for test/F1E2E.t.sol:F1E2ETest
[PASS] test_e2eCollision() (gas: 1663076)
Logs:
  mode: fallback-1log
  age: 305388541721942616793740676018599965597835985008
  exitId: 610777083443885233587481352037199931195671970016
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Full suite at the same fork block: **4 tests passed, 0 failed**
(`test_e2eCollision`, `test_forkSmoke`, `test_probeExitRecord`, +1).

Fixture: real Polygon tx
`0x50d6255b57cea5d9b8fd9e3f81a40786ecf25e4cd8140997ccc7a11371e597b8`
(block 97,470,272, txIndex 112, 1 Withdraw log at receipt-logIndex 2,
emitter `0x0000000000000000000000000000000000001010`, rootToken MATIC
`0x7d1afa7b718fb893db30a3abc0cfc608aacfebb0`, from aggregator
`0x67a40d016efe809a5bfcd942a8faf2d9cf0758e2`, ~112.7 MATIC),
checkpoint header **1171170000** (root
`0x4ef064376b83ab7a976043abce60c451956454bf2a0f8ec7298edcda5e4c091a`,
range 97,470,118–97,470,373, createdAt 1794912119). The receipt was REALLY
exited on Ethereum (ExitStarted, ETH block 26434271, exitId
`610777087527273636638742913597695220376890507488`); the fork block
26434270 is one block before that exit, so the checkpoint exists and the
exit does not yet. (A fork at "latest" fails the first exit with KNOWN_EXIT
because the real exit is already recorded.)

What the test asserts (in order):
1. `verifyInclusion(payload0)` and `verifyInclusion(payload1)` return the
   SAME `age` (same receipt ⇒ same proof ⇒ same branch mask ⇒ same age).
   This is the root cause, proven on the real contracts with real data.
2. `startExitWithBurntTokens(payload0)` as the log-0 withdrawer SUCCEEDS;
   `exits[exitId]` is recorded and the ExitNFT exists.
3. Same timestamp: the twin exit reverts `EXIT_ALREADY_EXISTS` (real-2log:
   the genuine second Withdraw log; fallback-1log: re-attempt of the same
   exit — same exitId, same rejection).
4. After warping 30 days ahead, `processExits(rootToken)` finalizes the first
   exit (ExitNFT burned, root tokens released).
5. Later timestamp: the twin exit is STILL impossible, now via `KNOWN_EXIT`
   — the full exitId changed with `now`, but `uint128(exitId)` (blockNumber +
   branchMask) is time-independent and `isKnownExit` is never cleared. The
   first exit's record is still stored. Permanent freeze, before and after
   finalization, at any time.

## What is real vs constructed

- REAL: the receipt, block header, all block receipts, the checkpoint
  (header number/root/start/end/createdAt from `RootChain.headerBlocks`), the
  receipt MPT proof and header Merkle proof (rebuilt from real data, roots
  re-verified), and the contracts (real deployed addresses on the fork).
  The receipt's Withdraw log was REALLY exited on Ethereum (ExitStarted at
  ETH block 26434271) — the exit is proven real, not just the receipt.
- CONSTRUCTED (fallback-1log mode only): payload1's `logIndex` field points
  at a real non-Withdraw log position (receipt-logIndex 3) in the same real
  receipt, to prove `verifyInclusion` ignores `logIndex`. No second Withdraw
  log exists in this receipt; the predicate-acceptance link for a genuine
  second Withdraw log is proven by the unit PoC
  (`work-pos/poc-exitid/`, 3/3 tests), not by this fork test.
- If a real 2-Withdraw-log receipt is found later, `build_fixture.py` will
  produce `"mode": "real-2log"` and the same test asserts the full collision
  (second exit reverts `EXIT_ALREADY_EXISTS` before processing, `KNOWN_EXIT`
  after). Do not present a constructed fixture as a real historical receipt.

## Related

- Unit-level PoC (3/3 Foundry tests, real WithdrawManager + PriorityQueue +
  ExitNFT behind the real proxy; mocks only for Registry/predicate wiring):
  `~/workspace/bugbounty/polygon/work-pos/poc-exitid/`
- Full context: `~/workspace/bugbounty/polygon/notes.md`,
  `~/workspace/bugbounty/polygon/scope.md`

---

## Main PoC Test — `test/F1E2E.t.sol`

```solidity
// SPDX-License-Identifier: MIT
// F-1 END-TO-END PoC — Plasma exitId collision against the REAL deployed
// Polygon contracts on an Ethereum mainnet fork.
//
// Two fixture modes (see ../build_fixture.py and ../data/fixture.json):
//   "real-2log"    -- a real receipt with TWO Withdraw logs. Full proof:
//                     exit0 succeeds, exit1 reverts EXIT_ALREADY_EXISTS,
//                     still reverts (KNOWN_EXIT) after the first exit finalizes.
//   "fallback-1log"-- a real receipt with ONE Withdraw log (the only kind
//                     found: Plasma exits are extremely rare -- see NOTES.md).
//                     Proves on the real contracts with real data:
//                     (a) verifyInclusion accepts the real payload,
//                     (b) the age is IDENTICAL for two logIndexes of the same
//                         receipt (the collision's root cause: the ID ignores
//                         logIndex), (c) the real exit succeeds, (d) the exit
//                         record survives finalization and re-entry is still
//                         rejected. The remaining link -- a genuine second
//                         Withdraw log being accepted and then hitting
//                         EXIT_ALREADY_EXISTS -- is proven by the unit PoC
//                         (work-pos/poc-exitid, real WithdrawManager, 3/3
//                         tests), NOT by this fork test. Nothing here is
//                         presented as a real 2-log receipt.
//
// Why TWO different revert strings (read the deployed code, don't guess):
//   exitId = age << 1, age = (exitableAt << 127) | (blockNumber << 32) | branchMask.
//   exitableAt = max(createdAt + 2*HALF_EXIT_PERIOD, now + HALF_EXIT_PERIOD);
//   the deployed HALF_EXIT_PERIOD reads 1, so exitableAt = now + 1 and the
//   FULL exitId changes every second. But uint128(exitId) -- the key of
//   isKnownExit -- is (blockNumber << 33) | (branchMask << 1): purely
//   receipt-dependent, independent of time. So:
//   - same timestamp: identical full exitId -> _addExitToQueue reverts
//     "EXIT_ALREADY_EXISTS" (exits[exitId].token != 0).
//   - later timestamp: different full exitId (record check passes) but the
//     same uint128 -> reverts "KNOWN_EXIT" (isKnownExit is never cleared).
//   Either way the second burn can NEVER exit, at any time, before or after
//   the first exit finalizes. Both reverts are the collision; the test
//   asserts each in its correct timing.
//
// Fork block: the fixture's receipt was REALLY exited on mainnet (ETH block
// 26434271, ExitStarted). Fork at 26434270 (one block before): the checkpoint
// already exists, the exit does not yet. A fork at "latest" reverts the first
// exit with KNOWN_EXIT because the real exit is already recorded.
//   forge test --fork-url <ETH_MAINNET_RPC> --fork-block-number 26434270 \
//     --match-test test_e2eCollision -vvv
pragma solidity 0.8.24;

import "forge-std/Test.sol";

interface IWithdrawManager {
    function verifyInclusion(bytes calldata data, uint8 offset, bool verifyTxInclusion)
        external
        view
        returns (uint256 age);
    function processExits(address token) external;
    function exits(uint256 exitId)
        external
        view
        returns (
            uint256 receiptAmountOrNFTId,
            bytes32 txHash,
            address owner,
            address token,
            bool isRegularExit,
            address predicate
        );
}

interface IPredicate {
    function startExitWithBurntTokens(bytes calldata data) external;
}

interface IExitNFT {
    function exists(uint256 tokenId) external view returns (bool);
    function ownerOf(uint256 tokenId) external view returns (address);
}

interface IWMGetters {
    function exitNft() external view returns (address);
}

contract F1E2ETest is Test {
    address constant WM_PROXY = 0x2A88696e0fFA76bAA1338F2C74497cC013495922;
    address constant PREDICATE = 0x4EeA1780c06709D7FA0BCaA6D0f1aB29673586c0;

    bytes payload0;
    bytes payload1;
    address from0;
    address from1;
    address rootToken;
    address exitNft; // resolved on-chain via the proxy's exitNft() getter
    string mode; // "real-2log" or "fallback-1log"
    bool log1IsWithdraw;

    function setUp() public {
        string memory root = vm.projectRoot();
        string memory json = vm.readFile(string.concat(root, "/data/fixture.json"));
        payload0 = vm.parseJsonBytes(json, ".payload0");
        payload1 = vm.parseJsonBytes(json, ".payload1");
        from0 = vm.parseJsonAddress(json, ".log0.from");
        from1 = vm.parseJsonAddress(json, ".log1.from");
        rootToken = vm.parseJsonAddress(json, ".log0.rootToken");
        mode = vm.parseJsonString(json, ".mode");
        log1IsWithdraw = vm.parseJsonBool(json, ".log1_is_withdraw");
        exitNft = IWMGetters(WM_PROXY).exitNft();
    }

    function test_e2eCollision() public {
        // 1+2. Same receipt, two log indexes -> identical age.
        // Root cause, proven on the REAL contracts with REAL data:
        // verifyInclusion never reads logIndex, so the exitId is identical
        // for every log in the receipt (at the same timestamp).
        uint256 age0 = IWithdrawManager(WM_PROXY).verifyInclusion(payload0, 0, false);
        uint256 age1 = IWithdrawManager(WM_PROXY).verifyInclusion(payload1, 0, false);
        assertEq(age0, age1, "ages must be identical across log indexes");
        uint256 exitId = age0 << 1;
        emit log_named_string("mode", mode);
        emit log_named_uint("age", age0);
        emit log_named_uint("exitId", exitId);

        // 3. First burn's exit succeeds.
        vm.prank(from0);
        IPredicate(PREDICATE).startExitWithBurntTokens(payload0);
        (, , address owner, address token, , ) = IWithdrawManager(WM_PROXY).exits(exitId);
        assertEq(owner, from0, "first exit owner");
        assertEq(token, rootToken, "first exit token");
        assertTrue(IExitNFT(exitNft).exists(exitId), "exit NFT minted");

        // 4. Same timestamp: the twin exit reverts EXIT_ALREADY_EXISTS.
        // (real-2log: the genuine second Withdraw log. fallback: re-attempt
        // of the same exit -- same exitId, same rejection.)
        vm.prank(log1IsWithdraw ? from1 : from0);
        vm.expectRevert(bytes("EXIT_ALREADY_EXISTS"));
        IPredicate(PREDICATE).startExitWithBurntTokens(
            log1IsWithdraw ? payload1 : payload0
        );

        // 5. Fully process the first exit (warp past the exitable time; the
        // deployed HALF_EXIT_PERIOD reads 1, so this is belt-and-braces).
        vm.warp(block.timestamp + 30 days);
        IWithdrawManager(WM_PROXY).processExits(rootToken);
        assertFalse(IExitNFT(exitNft).exists(exitId), "first exit NFT burned");

        // 6. Later timestamp: the twin exit is STILL impossible, now via
        // KNOWN_EXIT -- the full exitId changed with `now`, but uint128
        // (blockNumber + branchMask) is time-independent and isKnownExit is
        // never cleared. Permanent freeze, before and after finalization.
        vm.prank(log1IsWithdraw ? from1 : from0);
        vm.expectRevert(bytes("KNOWN_EXIT"));
        IPredicate(PREDICATE).startExitWithBurntTokens(
            log1IsWithdraw ? payload1 : payload0
        );
        (, , , address tokenAfter, , ) = IWithdrawManager(WM_PROXY).exits(exitId);
        assertEq(tokenAfter, rootToken, "first exit record never deleted");
    }
}
```
