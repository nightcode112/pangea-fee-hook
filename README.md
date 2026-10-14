# Pangea Fee Hook

The Uniswap v4 hook used by every [Pangea](https://pangea.trade) launch pool. Pangea launches one
token on Ethereum, BNB Chain, Base, Robinhood Chain and Solana. On the four EVM chains each token
trades in a Uniswap v4 pool (native coin / token) that uses this hook.

This repository holds exactly the sources of the deployed contract: `src/venues/PangeaFeeHook.sol`
and the Uniswap v4-core files it imports, byte-for-byte as compiled (each file's keccak256 matches
the build's `metadata.json`). Building it reproduces the deployed creation and runtime bytecode.

## Deployments

Same address on all four chains (CreateX CREATE3, deployer-guarded salt):

**`0x898B9E0A40F30D00bE3098A5865Ad5D71a93a8cc`**

| Chain | Chain id | Verified source |
|---|---|---|
| Ethereum | 1 | [etherscan.io](https://etherscan.io/address/0x898B9E0A40F30D00bE3098A5865Ad5D71a93a8cc#code) · [Sourcify](https://repo.sourcify.dev/1/0x898B9E0A40F30D00bE3098A5865Ad5D71a93a8cc) |
| BNB Chain | 56 | [bscscan.com](https://bscscan.com/address/0x898B9E0A40F30D00bE3098A5865Ad5D71a93a8cc#code) · [Sourcify](https://repo.sourcify.dev/56/0x898B9E0A40F30D00bE3098A5865Ad5D71a93a8cc) |
| Base | 8453 | [basescan.org](https://basescan.org/address/0x898B9E0A40F30D00bE3098A5865Ad5D71a93a8cc#code) · [Sourcify](https://repo.sourcify.dev/8453/0x898B9E0A40F30D00bE3098A5865Ad5D71a93a8cc) |
| Robinhood Chain | 4663 | [explorer](https://explorer.mainnet.chain.robinhood.com/address/0x898B9E0A40F30D00bE3098A5865Ad5D71a93a8cc) · [Sourcify](https://repo.sourcify.dev/4663/0x898B9E0A40F30D00bE3098A5865Ad5D71a93a8cc) |

Constructor arguments per chain:

| Chain | `poolManager` | `deployer` | `devBuyer` (the chain's LaunchFeeCollector) |
|---|---|---|---|
| Ethereum | `0x000000000004444c5dc75cB358380D2e3dE08A90` | `0x7794De82aa21CE22863102fB07E319daf85dd1c5` | `0xF61Ed4347c4D02522e96657a606f49E8381B3dCA` |
| BNB Chain | `0x28e2Ea090877bF75740558f6BFB36A5ffeE9e9dF` | `0x7794De82aa21CE22863102fB07E319daf85dd1c5` | `0x0a6198D54206999f07508F8ADb804c1BCCAEf21D` |
| Base | `0x498581fF718922c3f8e6A244956aF099B2652b2b` | `0x7794De82aa21CE22863102fB07E319daf85dd1c5` | `0x326c4Cd74477aF800bc91C4B96ceF0ce17731dC0` |
| Robinhood Chain | `0x8366a39CC670B4001A1121B8F6A443A643e40951` | `0x7794De82aa21CE22863102fB07E319daf85dd1c5` | `0xDAA3ABEb28F4104b6Bf30cC6CBf742fE969C52E6` |

Example pool (the same pool id on all four chains, since the hook and the token share one address
everywhere): TEST3 / native, `0x30bcb7fdc604132bff0cef5b949b35f9c48239f7f2fef57f79ea42c41c5a73a7`.

## What it does

Hook permissions (`0x28CC`): `beforeInitialize`, `beforeAddLiquidity`, `beforeSwap`, `afterSwap`,
`beforeSwapReturnDelta`, `afterSwapReturnDelta`.

- **A fixed trading fee in the native coin, on buys and on sells.** Each pool is registered once
  with a fee between 0.3% and 5% (Pangea launches use 1%) and a fee destination (that token's own
  fee splitter: creator + platform). The fee is taken from the native side of every swap through
  the return deltas, so fees are always paid in ETH/BNB, never in the token. Exact-in and
  exact-out swaps pay the same rate. The pool's own LP fee is 0.
- **Nothing can change after registration.** No owner, no admin, no proxy, no pause; the fee and
  its destination are fixed for the pool's life.
- **The launch's first buy is fee-free**: swaps made by `devBuyer` (the chain's LaunchFeeCollector,
  which makes a launch's dev buy or opening buy) pay no fee.
- **Only the launch can add liquidity** (`beforeAddLiquidity`): the pool's single-sided launch
  position is the only liquidity, which is locked by the seeder contract (it has no code path that
  removes liquidity).
- **Swaps cannot end past the launch range's edge**, so the pool never leaves its liquidity.
- **No `hookData`** is read.

## Build

```
forge build
```

Compiler settings (also in `metadata.json`): solc 0.8.30, optimizer 200 runs, via-IR, EVM `cancun`.
`remappings.txt` is the full list the deployed build was compiled with, so the metadata hash and
therefore the bytecode match exactly.

## Licences

`src/venues/PangeaFeeHook.sol` is published as source for review and verification; all rights
reserved. The files under `lib/uniswap-v4-core/` are Uniswap Labs' v4-core sources under their own
SPDX licences (MIT, and BUSL-1.1 for `Position.sol`).
