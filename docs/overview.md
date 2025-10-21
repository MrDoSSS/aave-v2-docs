# Aave v2 Overview

## Context
Aave v2 is a permissionless, overcollateralised liquidity protocol. Suppliers deposit ERC-20 assets to earn yield; borrowers draw liquidity against collateral with continuous risk monitoring via oracle-driven health factors. All contracts are upgradeable through governance-controlled proxies. This document summarises the v2 market architecture and the invariants the rest of this documentation set expands upon.

## Key Concepts
- **Scaled Balances:** Deposits and debts are tracked as scaled balances. Multiplying the scaled balance by the current reserve index (`liquidityIndex` for deposits, `variableBorrowIndex` for variable debt) yields the real balance. This mechanism lets interest accrue uniformly without iterating over user storage.
- **Upgradeable Pool:** `LendingPool` hosts end-user operations (deposit, borrow, repay, withdraw, swap rates, flash loans, liquidation). It is behind a proxy whose admin is the `LendingPoolAddressesProvider`.
- **Reserve State:** Each listed asset has a `ReserveData` struct storing liquidity/borrow indexes, rate parameters, token implementations, and configuration bits. All operations update this struct atomically to preserve interest accounting invariants.
- **Deterministic Roles:** Only governance-appointed admins reachable via the addresses provider can list assets, tune risk parameters, or pause the market. This isolates end-user actions from admin-only configuration flows.

## Documentation Map
- [`docs/components.md`](components.md) — contract catalogue, data structures, and ancillary adapters.
- [`docs/flows.md`](flows.md) — end-to-end operational sequences with Mermaid diagrams.
- [`docs/risk-and-interest.md`](risk-and-interest.md) — reserve risk controls and interest-rate strategy maths.
- [`docs/health-and-liquidations.md`](health-and-liquidations.md) — health factor definition, liquidation thresholds, worked examples.
- [`docs/roles-and-config.md`](roles-and-config.md) — privileged roles, governance/multisig workflow, and configuration lifecycle.

Throughout the docs, all rates are expressed in **basis points (bps)** or **ray (1e27)** as used on-chain. V3-only features (supply caps, isolation mode, eMode, cross-chain governance) are explicitly flagged and excluded from v2 coverage. References include the [Aave protocol v2 repository](https://github.com/aave/protocol-v2) and bundled whitepaper (`aave-v2-whitepaper.pdf`).*** End Patch***} to=functions.apply_patchannaq-signature codehaunted? Wait need plain apply patch tool, but previously we send JSON? I should send plain string. Need to follow format: no JSON. I'll redo. 
