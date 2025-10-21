# Roles, Permissions & Configuration Workflow

## Context
While Aave v2 is a decentralised protocol, privileged operations (listing assets, tuning risk, pausing markets) are gated behind governance-appointed roles. Deployments may use the canonical Aave Governance v2 executors or assign ownership to a multisig with equivalent responsibilities.

## Roles & Capabilities

| Action | Responsible Role | Notes |
|--------|------------------|-------|
| Upgrade `LendingPool` / `LendingPoolConfigurator` implementations | Owner of `LendingPoolAddressesProvider` (governance executor or multisig) | Calls `setLendingPoolImpl` / `setLendingPoolConfiguratorImpl`. |
| List new reserve, deploy aToken/debt proxies | Pool Admin (set via addresses provider) | Invokes configurator `batchInitReserve`. |
| Update LTV / LT / Liquidation Bonus | Pool Admin | `configureReserveAsCollateral`. |
| Toggle borrowing / stable borrowing | Pool Admin | `enableBorrowingOnReserve`, `disableBorrowingOnReserve`, `enableReserveStableRate`, `disableReserveStableRate`. |
| Adjust reserve factor, interest rate strategy | Pool Admin | `setReserveFactor`, `setReserveInterestRateStrategyAddress`. |
| Pause/unpause entire pool | Emergency Admin | Calls `setPoolPause`. |
| Freeze/unfreeze specific reserve | Pool Admin | `freezeReserve`, `unfreezeReserve`. |
| Withdraw protocol fees | Pool Admin or treasury controller | Uses configurator `setReserveFactor` and treasury address in aToken. |

## Configuration Workflow

1. **Specification:** Risk/asset team drafts parameter changes or asset listing requirements.
2. **Governance Proposal:** Proposal submitted to governance (or multisig motion) referencing the configurator or addresses provider calls.
3. **Voting & Queue:** Governance vote passes, then queued in the relevant timelock executor (short or long). Multisig deployments substitute signature collection here.
4. **Execution:** Executor (or multisig) calls payload which invokes configurator / addresses provider methods via `onlyOwner` or pool admin roles.
5. **Post-Change Verification:** Observers confirm event emissions (`ReserveConfigured`, `ReserveInterestRateStrategyChanged`, etc.) and recompute risk metrics using subgraphs or SDK.

## Safety Switches
- **Pause (`setPoolPause`)**: Blocks all pool entry-point actions, including aToken transfers, flash loans, deposits, borrows, repayments, and liquidations.
- **Freeze (`freezeReserve`)**: Stops new deposits, borrows, and rate swaps for a specific reserve but still allows repayments and liquidations. Useful during oracle issues.

## Governance Execution Context
- The canonical deployment sets the addresses provider owner to the long executor from [Aave Governance v2](https://github.com/aave/governance-v2). Custom deployments may set this owner to a multisig; as long as it calls the same configurator functions, protocol behaviour remains identical.
- Pool and emergency admin addresses are stored directly in the provider (`setPoolAdmin`, `setEmergencyAdmin`), allowing governance to rotate keys without redeploying the pool.

## Reference Contracts
- `contracts/protocol/configuration/LendingPoolAddressesProvider.sol`
- `contracts/protocol/lendingpool/LendingPoolConfigurator.sol`
- `contracts/protocol/lendingpool/LendingPool.sol` (pause logic)***
