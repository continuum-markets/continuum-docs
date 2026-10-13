# Devnet program rollout: completed

Release source `cd2ae7a2c0bb8fec8618abd7dc147652a082f0081a314d082368ac4574858e14` is deployed analytics commit `29960efb4d9db88c2bfc2ead42f0a2b8af52b847` plus exactly five reviewed CLP/Oracle files. The working-tree corporate-action changes are excluded, including from the mint-redeem dependency. The detached release source is `/tmp/continuum-performance-release-source/programs`; `performance-only.patch` and `release-boundary.json` record the boundary. Existing business events are retained.

Artifacts (v1.45, SBFv2, unchanged release profile):

| Program | Reviewed artifact | SHA-256 | Bytes |
|---|---|---|---:|
| CLP | `/tmp/continuum-performance-release-sbf/optimized/clp.so` | `0830f8c0acba63aa6fd26090462d5ba7e82922362dffd72c2ce3a57b01a5c56e` | 807,384 |
| Oracle, devnet feature | `/tmp/continuum-performance-release-sbf/optimized/oracle.so` | `f168f19075c3b6829ba53b3c977067f9587b784c2528e58f0d4bf3e1a6135e02` | 307,680 |

The configured signer `/Users/nafironato/.config/solana/id.json` exposes public key `8xhPK1rGK4neoKo4fT8FxHQDZRiCnSYW9hvEdaVTxCni`. Both live upgrade authorities and all 15 selected config admins match. Balance observed at finalized slot 509015415: 7.435820120 SOL. The peak required upload buffer is approximately 4.102348920 SOL, recovered by successful upgrade; verify balance and current rent again immediately before upload. No authority changes or mainnet operations are planned.

`release-coverage.md` and `evidence/svm-coverage.json` record 115 clean-release entrypoints, 167 scenarios and 835 signed local transactions with five stable replays. Eight exact-account CLMM comparisons pass. The allocation unit test passes. All 15 selected public configs at finalized slot 509016557 decode, match their PDA/admin and existing Market authority, and resize successfully against the candidate Oracle in LiteSVM with the actual Rent sysvar. Signature verification was deliberately disabled for this public-data simulation; no private signer was loaded. The existing configured signer's public identity was checked separately. Every original data byte is preserved; added bytes are zero; all 15 need zero rent top-up at the observed rent schedule.

The following sequence was authorized and completed; [deployment.json](deployment.json) records finalized signatures, slots, hashes and all account outcomes. Both tracked buffers are closed with zero balances. These commands document the completed run; use the checkpointed wrappers to reconcile/resume, not blind re-execution.

Executed order:

1. Recheck source/binary hashes, devnet genesis, current upgrade authorities, signer balance, and both live program hashes against the backups. Stop on unrelated changes. Confirm the selected configs still meet the plan; keep the three missing-Market and one older-schema accounts excluded.
2. Extend Oracle's existing program allocation by **11,992 bytes**, as a separate transaction. Existing program-data funding covers the observed rent requirement. Record signature and finalized allocation before proceeding.
3. Upload/upgrade Oracle with the persistent prepared buffer key. Verify deployed binary prefix equals the reviewed artifact and all remaining allocation bytes are zero. Verify upgrade authority is unchanged and buffer closure/refund is accounted for.
4. Submit one config resize as a canary using `oracle-migration-plan.json`. Preflight on the upgraded live program, journal the signed transaction/signature before sending once, confirm it, and verify size 1,781 plus unchanged original bytes (account for independent keeper updates). On unknown confirmation, reconcile the recorded signature and live state; do not blindly resend.
5. Repeat those preflight/receipt/post-state checks sequentially for the remaining 14 configs. Checkpoint each completed address; skip already verified completed work on resume.
6. Upload/upgrade CLP using its persistent buffer key. No program extension is required. Verify exact binary prefix/zero allocation tail, unchanged authority, buffer disposition, and a small authorized devnet smoke transaction through the normal app.

The explicit CLI commands use public RPC, existing authority, one signing iteration and tracked buffers. The coordinator must execute through a wrapper that captures raw CLI output in a private 0600 file and publishes only allowlisted receipt/error fields; never copy a recovery phrase into evidence.

```sh
solana program extend 5vxiCrDpFnQ2W5QtgZBC66K2XTC19bjVBjinGYYBsadC 11992 \
  --url https://api.devnet.solana.com --keypair /Users/nafironato/.config/solana/id.json \
  --commitment finalized --use-quic --output json

solana program deploy /tmp/continuum-performance-release-sbf/optimized/oracle.so \
  --program-id 5vxiCrDpFnQ2W5QtgZBC66K2XTC19bjVBjinGYYBsadC \
  --buffer /tmp/continuum-performance-rollout/private/oracle-buffer.json \
  --upgrade-authority /Users/nafironato/.config/solana/id.json \
  --fee-payer /Users/nafironato/.config/solana/id.json --keypair /Users/nafironato/.config/solana/id.json \
  --url https://api.devnet.solana.com --commitment finalized --use-quic --max-sign-attempts 1 --output json

solana program deploy /tmp/continuum-performance-release-sbf/optimized/clp.so \
  --program-id 8xauDRjw9XRyk4FE3hW1JKjD8nC87gfr59Xig1dJqLES \
  --buffer /tmp/continuum-performance-rollout/private/clp-buffer.json \
  --upgrade-authority /Users/nafironato/.config/solana/id.json \
  --fee-payer /Users/nafironato/.config/solana/id.json --keypair /Users/nafironato/.config/solana/id.json \
  --url https://api.devnet.solana.com --commitment finalized --use-quic --max-sign-attempts 1 --output json
```

Rollback binaries are `/tmp/continuum-performance-rollout/{oracle,clp}-before.so`, with public hashes in `program-preparation.json`. The CLP backup exactly matches the live analytics deployment dump. Program rollback restores the prior binary through a separately checkpointed upgrade; the extra program allocation and safely enlarged config accounts need not be shrunk. Do not shrink account state or reverse successful migrations. A failed or interrupted CLI call requires signature/program/buffer reconciliation before any resume or authorized buffer cleanup.

The repository preflight passes with the scoped IDL check supplied separately in `idl-compatibility.json`. Existing instruction/account layouts remain identical; Oracle adds `realloc_oracle_config`. Fresh CLP IDL additionally exposes the already-existing `SwapExecuted` event type, with no changed existing type. The generic deployment script is not used because it rebuilds and also deploys mint-redeem. Mint-redeem, registry and faucet are outside this deployment scope.

