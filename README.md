# Pangea Fee Hook and Liquidity Locker

The Uniswap v4 hook used by every [Pangea](https://pangea.trade) launch pool, and the liquidity
locker that holds every launch's liquidity. Pangea is a launchpad: each token it launches trades on
Ethereum, BNB Chain, Base, Robinhood Chain and Solana. On the four EVM chains each token trades in a
Uniswap v4 pool (native coin / token) that uses this hook, with its liquidity held by this locker.

This repository holds exactly the sources of the deployed contracts — `src/venues/PangeaFeeHook.sol`,
`src/venues/LiquidityLocker.sol` and the Uniswap v4-core files they import — byte-for-byte as compiled
(each file's keccak256 matches the build metadata in `metadata.json` / `metadata-locker.json`).
`forge build` reproduces the deployed creation and runtime bytecode of both.

## Deployments

Both are at the same address on all four chains (CreateX CREATE3, deployer-guarded salts):

| Contract | Address |
|---|---|
| PangeaFeeHook | **`0x5B58765D0b919F485bC36487Bada7f9f7fE6e8cc`** |
| LiquidityLocker | **`0xBdC307223630664e33132d64aDf0cc39c7a27e3D`** |

| Chain | Chain id | Verified source (hook) | Verified source (locker) |
|---|---|---|---|
| Ethereum | 1 | [etherscan.io](https://etherscan.io/address/0x5B58765D0b919F485bC36487Bada7f9f7fE6e8cc#code) · [Sourcify](https://repo.sourcify.dev/1/0x5B58765D0b919F485bC36487Bada7f9f7fE6e8cc) | [etherscan.io](https://etherscan.io/address/0xBdC307223630664e33132d64aDf0cc39c7a27e3D#code) |
| BNB Chain | 56 | [bscscan.com](https://bscscan.com/address/0x5B58765D0b919F485bC36487Bada7f9f7fE6e8cc#code) · [Sourcify](https://repo.sourcify.dev/56/0x5B58765D0b919F485bC36487Bada7f9f7fE6e8cc) | [bscscan.com](https://bscscan.com/address/0xBdC307223630664e33132d64aDf0cc39c7a27e3D#code) |
| Base | 8453 | [basescan.org](https://basescan.org/address/0x5B58765D0b919F485bC36487Bada7f9f7fE6e8cc#code) · [Sourcify](https://repo.sourcify.dev/8453/0x5B58765D0b919F485bC36487Bada7f9f7fE6e8cc) | [basescan.org](https://basescan.org/address/0xBdC307223630664e33132d64aDf0cc39c7a27e3D#code) |
| Robinhood Chain | 4663 | [explorer](https://explorer.mainnet.chain.robinhood.com/address/0x5B58765D0b919F485bC36487Bada7f9f7fE6e8cc) · [Sourcify](https://repo.sourcify.dev/4663/0x5B58765D0b919F485bC36487Bada7f9f7fE6e8cc) | [Sourcify](https://repo.sourcify.dev/4663/0xBdC307223630664e33132d64aDf0cc39c7a27e3D) |

Constructor arguments. Hook: `(poolManager, deployer, devBuyer, locker)`; locker: `(poolManager, deployer)`.
The deployer is `0x7794De82aa21CE22863102fB07E319daf85dd1c5` and the locker is `0xBdC3…7e3D` on every chain.

| Chain | `poolManager` | `devBuyer` (the chain's LaunchFeeCollector) | Launch factory |
|---|---|---|---|
| Ethereum | `0x000000000004444c5dc75cB358380D2e3dE08A90` | `0xF0973d5e539781C00524D071856dCFFF41Fa366b` | `0x5e234a65c40c85712C0953171172Eb4331C8d99C` |
| BNB Chain | `0x28e2Ea090877bF75740558f6BFB36A5ffeE9e9dF` | `0xC92E0925b6eD95E7E42882A1377a8EFd9440ef7E` | `0x6c25034dB757E19d0fe55FF194589eABD29873cA` |
| Base | `0x498581fF718922c3f8e6A244956aF099B2652b2b` | `0x9Db7a6e6E974fbeA0B379EeC60DbE17235bfa369` | `0x2fD03cf3D6B9b91F13310bdBB72840eAF51a8cB7` |
| Robinhood Chain | `0x8366a39CC670B4001A1121B8F6A443A643e40951` | `0xC92E0925b6eD95E7E42882A1377a8EFd9440ef7E` | `0x6c25034dB757E19d0fe55FF194589eABD29873cA` |

## What the hook does

Hook permissions (`0x28CC`): `beforeInitialize`, `beforeAddLiquidity`, `beforeSwap`, `afterSwap`,
`beforeSwapReturnDelta`, `afterSwapReturnDelta`.

- **A fixed trading fee in the native coin, on buys and on sells.** Each pool is registered once by
  the launch factory with a fee between 0.3% and 5% (Pangea launches use 1%) and a fee destination
  (that token's own fee splitter: creator + platform). The fee is taken from the native side of every
  swap through the return deltas, so fees are always paid in ETH/BNB, never in the token. Exact-in
  and exact-out swaps pay the same rate. The pool's own LP fee is 0.
- **Nothing can change after registration.** No owner, no admin, no proxy, no pause; the fee and its
  destination are fixed for the pool's life.
- **The launch's first buy is fee-free**: swaps made by `devBuyer` (the chain's LaunchFeeCollector,
  which makes a launch's dev buy or opening buy inside the launch transaction) pay no fee.
- **Only the locker can add liquidity** (`beforeAddLiquidity`): each pool's single-sided launch
  position, held by the locker, is its only liquidity.
- **Swaps cannot end past the launch range's edge**, so the pool never leaves its liquidity.
- **No `hookData`** is read.

## What the locker does

- **One locker per chain holds every launch's position**, each pool's position separate (the v4
  position salt is the pool id).
- **There is no function that removes liquidity.** The only liquidity writes are the one-time seed
  (positive liquidity, factory-only, once per pool) and fee collection (zero liquidity delta).
- **No owner, no admin, no proxy, no upgrade.**
- `collect(poolId)` is permissionless and pays that pool's fees only to that pool's own fee splitter,
  recorded at seeding. Fees never mix between pools.

## Build

```
forge build
```

Compiler settings (also in the metadata files): solc 0.8.30, optimizer 200 runs, via-IR, EVM `cancun`.
`remappings.txt` is the full list the deployed build was compiled with, so the metadata hash and
therefore the bytecode match exactly.

## Licences

`src/venues/PangeaFeeHook.sol` and `src/venues/LiquidityLocker.sol` (and the Pangea files they import)
are published as source for review and verification; all rights reserved. The files under
`lib/uniswap-v4-core/` are Uniswap Labs' v4-core sources under their own SPDX licences (MIT, and
BUSL-1.1 for `Position.sol`).
