# Core Components

## What and Why
The Aave v2 market is composed of modular contracts with narrowly scoped responsibilities. Understanding which contract handles which concern is crucial when auditing flows or designing integrations.

## Contract Catalogue

| Component | Contracts | Responsibilities | How It Fits in Flows |
|-----------|-----------|------------------|----------------------|
| `LendingPool` (proxy + implementation) | `LendingPool.sol` | Handles end-user actions: deposit, withdraw, borrow, repay, rate swap, flash loan, liquidation. Maintains reserve state and enforces validation logic. | Entry point for all user-triggered flows; updates reserve indexes before mint/burn operations. |
| `LendingPoolAddressesProvider` | `LendingPoolAddressesProvider.sol` | Registry for pool, configurator, oracle, lending-rate oracle, pool admin, emergency admin, collateral manager. | Governance/multisig updates these addresses; other contracts read them to locate canonical components. |
| `LendingPoolConfigurator` (proxy + implementation) | `LendingPoolConfigurator.sol` | Admin interface to list reserves, configure LTV/LT/bonus, toggle borrowing, deploy aTokens and debt tokens, upgrade implementations. | Called only by pool admin role after governance execution; its changes feed into future pool interactions. |
| `LendingPoolCollateralManager` | `LendingPoolCollateralManager.sol` | Executes liquidation mechanics via `delegatecall` from the pool, enforcing close factor and bonus application. | Invoked during `liquidationCall` flow to compute seized collateral and burn debt tokens. |
| `AToken` | `AToken.sol` (+ proxies per reserve) | Interest-bearing ERC-20 minted on deposit, burned on withdrawal. Stores reserve treasury address for protocol fee minting. | Receives mint/burn instructions from pool; scaled balances track depositor share. |
| `StableDebtToken` | `StableDebtToken.sol` (+ proxies per reserve) | Tracks principal and fixed borrow rate per user; supports rebalancing when utilisation is extreme. | Minted on stable borrow, burned on repay or liquidation; feeds health factor calculations. |
| `VariableDebtToken` | `VariableDebtToken.sol` (+ proxies per reserve) | Tracks scaled variable debt using borrow index. | Minted on variable borrow, burned on repay or liquidation. |
| Interest Rate Strategy | `DefaultReserveInterestRateStrategy.sol` (or custom) | Computes liquidity and borrow rates based on utilisation and slope parameters. | Queried by `LendingPool` whenever reserve state updates are required. |
| Price Oracles | `AaveOracle.sol` + Chainlink feeds | Provides asset prices in ETH; supports fallback oracle. | Used in health factor, liquidation eligibility, and borrowing power checks. |
| Flash Loan Adapters | `FlashLoanReceiverBase.sol` implementers | Provide hooks for integrations to use flash liquidity. | Called during flash loan operations; must repay plus premium within transaction. |
| DEX Swap Adapters (optional) | `UniswapLiquiditySwapAdapter`, `UniswapRepayAdapter`, `ParaSwapLiquiditySwapAdapter` | Route collateral or borrowed funds through Uniswap/ParaSwap during refinancing, repay, or liquidation helper flows. | Invoked via flash-loan callbacks; must settle flash debt back to the pool within transaction. |
| Incentives Controller (optional deployment) | `AaveIncentivesController` | Distributes liquidity mining rewards (e.g., stkAAVE) based on index accounting. | aTokens and debt tokens call `handleAction` to update reward state after user interactions. |

## Data Structures

- **`ReserveData`** (`DataTypes.sol`): stores configuration bitmask, liquidity/borrow indexes, current rates, token addresses, interest rate strategy, and an id. Updated on every state change.
- **`ReserveConfigurationMap`**: 256-bit packed structure encoding LTV, liquidation threshold, liquidation bonus, decimals, active/frozen flags, borrowing/stable toggles, and reserve factor.
- **`UserConfigurationMap`**: bitmask that tracks, per reserve, whether the user is using it as collateral and/or has an outstanding borrow.
- **Scaled Balances:** `AToken` and `VariableDebtToken` expose `scaledBalanceOf`. Multiply by the current index to derive real balance: `balance = scaledBalance * liquidityIndex / RAY`.

## How It Works
1. **Address Discovery:** Front ends and contracts query the addresses provider to locate the current pool, configurator, and oracles. Governance (or a multisig) is the provider owner and can swap implementations.
2. **Reserve Lifecycle:** When a new asset is listed, configurator deploys token proxies, initialises indexes to 1e27, sets risk parameters in the configuration map, and stores the interest rate strategy address in `ReserveData`.
3. **Runtime Operations:** Each user action enters the pool, which fetches the reserve state, validates conditions via `ValidationLogic`, updates indexes/rates via `ReserveLogic`, and calls token contracts to mint/burn as needed.
4. **Liquidations:** The pool delegates to the collateral manager to compute how much debt can be covered (fixed 50% close factor) and the collateral to seize with the liquidation bonus applied.
5. **Adapters:** Optional swap adapters receive flash liquidity, execute trades via external routers (Uniswap or ParaSwap), and return funds before the flash loan closes—allowing users to deleverage or migrate positions atomically without native DEX support in the core pool.

All components are designed for composability: external applications can rely on proxy-address stability while upgrades are governed through the addresses provider ownership.***
