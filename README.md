# TickHarberger

**Leases the right to price a pool's flow, one tick range at a time, under a Harberger tax.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://tick-harberger.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/TickHarbergerHook.sol`](src/hooks/TickHarbergerHook.sol)
- **Licence:** Apache-2.0

## How it works

An automated market maker charges the same fee everywhere, which is the same as saying it believes every price level is equally valuable to trade at. No venue with a human on it believes that. The flow near the current price is worth something quite different from the flow forty percent away, and it changes hour by hour, and a governance vote on a single pool-wide number is not a mechanism for discovering it.

Auction-managed AMMs answered this by selling the right to set the fee for the whole pool, which is a real improvement and still one number. This sells it per tick range. Somebody who believes the band the price is sitting in is worth more than the market thinks can buy exactly that band, and nothing else.

The lease is Harberger. A holder names their own price for the range and pays continuous rent on that number, and anybody may take the range from them at any moment by paying it. Naming a low price is cheap and loses the range; naming a high one keeps it and costs.

There is no auction to run, no round to wait for, and nobody who decides who wins: the holder's own valuation is both the tax base and the strike, which is what makes the two honest at once. Rent accrues only while a range holds the current price, which is the only time the right is worth anything. That is not a concession, it is what makes squatting self-defeating: holding a distant range costs nothing, and the valuation that makes it free to hold is also the price at which it is taken away the moment it becomes valuable.

Rent goes to the pool's liquidity providers by donation, so the people whose capital is being priced are the people paid for it. What the leaseholder earns is the fee they set, taken on top of the pool's own, which is the position they paid rent to occupy.

## Prior art

The am-AMM line of work (Adams, Milionis, Moallemi, Roughgarden) auctions the right to manage a whole pool, and v4 hooks implementing it exist. Harberger taxes on-chain go back to Radical Markets and appear in Wildcards, This Artwork Is Always On Sale and the partial-common-ownership NFT designs. Dynamic-fee hooks set one fee per pool from volatility or flow. Leasing the fee-setting right per tick range under a continuous self-assessed tax, so that price levels are priced separately and by whoever thinks they know better, is the contribution here.

## Where it does not help

Rent only accrues where the price actually is, so a pool that never moves pays rent on one range and leaves the rest free to hold. That is the intended incentive and it does mean the mechanism says nothing about ranges the price never visits. The lease is denominated in the pool's second currency, which must be an ERC-20, so a pool of native currency against nothing else cannot use it. Rent reaches providers through `donate`, which credits whoever is in range when it settles rather than whoever was in range while it accrued; settling often keeps that close, and `settleRent` is callable by anyone for exactly that reason. Finally, a leaseholder charging the maximum is still charging it, so `maxFeePips` is the real protection for traders and a pool that sets it carelessly has sold them.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    TickHarbergerHook.Config({
        rangeWidth: /* int24 */ 0,
        rentRateBps: /* uint32 */ 0,
        maxFeePips: /* uint24 */ 0,
        minDeposit: /* uint128 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `rangeWidth` | `int24` |  |
| `rentRateBps` | `uint32` | basis points (`10000` = 100%) |
| `maxFeePips` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `minDeposit` | `uint128` |  |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `CallbackNotPoolManager()` | Only the `PoolManager` may drive the unlock callback. Named distinctly because `BaseHook` declares its own. |
| `DepositTooSmall(uint128)` | The deposit is below the pool's minimum, which would leave the lease with no runway. |
| `FeeTooHigh(uint24)` | The fee asked for is above what this pool allows a leaseholder to charge. |
| `HookFeeTooLarge()` | Fee is higher than the maximum allowed fee. |
| `InvalidMaxFee()` | The ceiling on a leaseholder's fee must itself be below the protocol maximum. |
| `InvalidRangeWidth()` | A range narrower than the tick spacing, or not a whole number of them, cannot be a range. |
| `InvalidRentRate()` | A rent of zero is not a Harberger tax, and one above the whole valuation per day is confiscation. |
| `InvalidValuation()` | A valuation of zero would make the lease free to take and free to hold, which is not a lease. |
| `NothingToDo()` | There is nothing to settle or withdraw. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 4 of the fourteen:

- `afterInitialize`
- `beforeSwap`
- `afterSwap`
- `afterSwapReturnsDelta`

Mask: `0x10c4`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # TickHarberger
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # fees, harberger, auction, mev, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/tick-harberger
cd tick-harberger
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
