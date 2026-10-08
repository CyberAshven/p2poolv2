# P2Poolv2 for Bitcoin Cash: plan

Status: plan only, no code yet. Written 2026-10-08 from design discussions
with the maintainer of [Pickaxe](https://github.com/CyberAshven/pickaxe-miner).
Audience: contributors and AI agents starting the BCH port. Read
[AGENTS.md](../AGENTS.md) and `docs/architecture/` before changing code.

## Goal

A peer-to-peer Bitcoin Cash mining pool built from P2Poolv2: miners
coordinate without an operator, and every block's coinbase pays the miners
directly. Pickaxe will run it for its users the way Gupaxx runs P2Pool for
Monero: the user's own node plus a local share chain, behind one setup
screen. Pool operators may use the same software too, while the aim is to
move most BCH hashrate to the share chain.

## Decisions so far

1. **Base.** Fork P2Poolv2 (Rust, MIT/Apache-2.0) and port the Bitcoin
   specifics to BCH. Take what is needed from P2Pool v1 for BCH
   ([jtoomim/p2pool](https://github.com/jtoomim/p2pool), and
   [BitcoinCash1/p2poolBCH](https://github.com/BitcoinCash1/p2poolBCH), a fork
   of frstrtr's): both are GPL-3.0, so copying their code (not just their
   ideas) changes this fork's license; see open questions.
2. **Miner protocol: Stratum V2 first.** Encryption with an authenticated
   server key, binary framing, standard and extended channels, and Job
   Declaration, added the way ckpool added SV2 beside SV1. P2Poolv2's current
   stratum server is SV1; nothing about it needs to stay hardcoded. SV1
   remains only for firmware that cannot speak SV2 (Pickaxe already
   translates SV1 devices). Prior art:
   [average-gary/sv2-p2pool](https://github.com/average-gary/sv2-p2pool), an
   SV2 pool with P2Poolv2's share chain as its backend, "usable on testnet4".
3. **Job Declaration: both modes.** Full-Template and Coinbase-only, because
   miners and pools are in transition. On BCH, declared jobs must keep
   canonical transaction order (CTOR), or they declare invalid blocks.
4. **Payouts: direct in the coinbase, no ledger, no custody.**
   - PPLNS: a share enters a sliding window (the "conveyor belt"; Monero's
     P2Pool uses 2,160 share-chain blocks, about 6 hours). While the share is
     in the window, it earns a slice of every block found.
   - Payouts appear in the miners' wallets with the block and become
     spendable after 100 blocks (BCH consensus, about 17 hours).
   - A miner who stops keeps earning from blocks found while their shares
     are in the window; then the shares slide out. Nothing is held, kept or
     expired.
   - Dust: each share is worth a slice of a full block reward, far above
     BCH's dust limit, so nothing accumulates.
   - Small miners' variance: a main and a "mini" share chain with lower
     share difficulty, as Monero's P2Pool has (main, mini, nano), plus uncles
     (P2Poolv2 already has them).
   - Optional later: a keyless covenant payout tree in one coinbase output
     (BCH introspection can check the spending transaction's outputs), only
     if the number of coinbase outputs becomes a problem.
5. **Merge mining.** Reserve one coinbase commitment output whose hash covers
   every token the miner chose to merge-mine (AuxPoW-style); each token's
   covenant checks a short proof up to it. The share chain fixes the payouts;
   each share's token choice is free within a size budget. This is what lets
   Pickaxe users choose their tokens.
6. **Templates from the miner's own node.** BCHN over local JSON-RPC
   (`getblocktemplate`, or `getblocktemplatelight` with `submitblocklight`),
   or Knuth (its C API in-process, or its Stratum V2 Template Distribution
   once complete). Any local inter-process link works; Cap'n Proto is not
   required by the Stratum V2 specification.
7. **Interface with Pickaxe: Stratum V2 is the boundary.** The P2Pool node
   acts as an SV2 pool and job declarator; Pickaxe connects the devices (SV2
   directly, SV1 through its adapter) and the user's node, and can start the
   P2Pool node in the background. Pickaxe's default will be the most
   decentralized option: the local share chain plus the user's own node.
8. **Not adopted.** DATUM-style designs, where an operator sets the coinbase
   outputs and holds small balances, keep the central infrastructure this
   project wants to remove.

## BCH porting checklist

Found by reading P2Poolv2's code and docs; each item needs a short design
check before code.

- **Segwit removal.** Witness commitment, wtxid handling and witness data in
  compact blocks: `p2poolv2_lib/src/shares/` (`compact_block.rs`,
  `share_commitment.rs`, `transactions/coinbase.rs`, `handle_stratum_share.rs`,
  `share_block/`) and `p2poolv2_lib/src/node/` (`emission_worker.rs`,
  `messages.rs`).
- **Share chain addresses.** Today bech32m over a Taproot (P2TR) key
  (`docs/architecture/address-format.md`, `p2poolv2_wallet`). BCH has no
  Taproot: choose a BCH form (for example CashAddr over a Schnorr key).
- **Share trading.** The atomic-swap scripts for selling shares
  (`docs/atomic-swap/`) need BCH scripts, or the feature waits while the mini
  share chain covers small miners.
- **Block rules.** CTOR (lexicographic transaction order), no block weight
  (BCH's adaptive block size limit, ABLA), BIP34 height, ASERT on the main
  chain, 100-block coinbase maturity, CashAddr payout scripts.
- **Node RPC.** BCHN's `getblocktemplate` has no segwit fields; the light
  calls are optional; BCHN supports ZMQ `hashblock`.
- **Networks.** BCH mainnet, chipnet and regtest as one code path with a
  network switch; new work is tested on chipnet first.
- **Share window and difficulty.** Tune the PPLNS window and share interval
  for BCH's 10-minute blocks; keep every share above dust; define the main
  and mini share chains. Share difficulty already uses ASERT
  (`docs/difficulty_adjustment/`).
- **Stratum V2 server.** On the reference crates (`stratum-core`), beside or
  instead of SV1, with both Job Declaration modes.

## Open questions

- License: if v1 code is copied, relicense the fork as GPL-3.0 or AGPL-3.0
  (AGPL-3.0 matches Pickaxe).
- PPLNS window length, share interval and share difficulty for the main and
  mini share chains.
- Keep P2Poolv2's share trading on BCH, or rely on the mini share chain.
- Merge-mining commitment format and size budget (Stratum V2's
  `CoinbaseOutputConstraints` already declares coinbase space).
- Binding each miner's payout address to their shares over SV2 (sv2-p2pool
  lists it as open, its ADR 0002).
- Whether the covenant payout tree is needed at all; prove it within BCH's VM
  limits first.
- Pool operators running this software beside the share chain.

## First steps

1. Read `AGENTS.md` and `docs/architecture/`; build and run the tests on
   Bitcoin signet unchanged.
2. Write a design check for each porting item.
3. Build a BCHN regtest and chipnet harness.
4. Implement in small pull requests, each verified on regtest and chipnet.

## Sources

- P2Poolv2: https://github.com/p2poolv2/p2poolv2 ; docs, including the
  comparison with Stratum V2 and DATUM: https://github.com/p2poolv2/docs
- sv2-p2pool: https://github.com/average-gary/sv2-p2pool ; its design notes:
  https://github.com/average-gary/wiki/tree/main/topics/sv2-p2pool-integration
- Stratum V2 specification: https://stratumprotocol.org/specification/03-protocol-overview/ ;
  Job Declaration: https://stratumprotocol.org/specification/06-job-declaration-protocol/
- Coinbase payouts extension (open): https://github.com/stratum-mining/sv2-spec/pull/203
- ckpool (Stratum V2 pool, solo, proxy and Job Declaration server): https://github.com/ckolivas/ckpool
- Monero P2Pool: https://github.com/SChernykh/p2pool ; Gupaxx:
  https://github.com/Cyrix126/gupaxx
- P2Pool v1: https://github.com/jtoomim/p2pool ,
  https://github.com/BitcoinCash1/p2poolBCH
- OCEAN DATUM: https://ocean.xyz/docs/datum
- Pickaxe: https://github.com/CyberAshven/pickaxe-miner (Stratum V2 server,
  SV1 adapter and BCH templates in `src/stratum_v2/`)
