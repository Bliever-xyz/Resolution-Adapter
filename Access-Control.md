# Access Control — Architecture Reference

**Protocol:** Bliever Protocol  
**System:** `BlieverUmaAdapter` + `BlieverMarket`  
**Document Scope:** Every role, modifier, and access guard across both contracts.

---

## Overview

Access control is split across two contracts with different architectures:

- **`BlieverUmaAdapter`** uses OpenZeppelin `AccessControlUpgradeable` — explicit role identifiers, role membership, and a hierarchy. Multiple addresses can hold each role.
- **`BlieverMarket`** uses single-address guards — each privileged function is gated to exactly one stored address (`resolver`, `factory`). No role system; no membership lists.

---

## Visual Role Hierarchy & Membership

The following diagram illustrates the role hierarchy in the `BlieverUmaAdapter` and the address-based guards in the `BlieverMarket`.

![Access Hierarchy Diagram](https://private-us-east-1.manuscdn.com/sessionFile/RFp897btO1Cqct7PJ3MyE0/sandbox/A6mdFL2MNTTj9h0xNu3Ba5-images_1775923415149_na1fn_L2hvbWUvdWJ1bnR1L2FjY2Vzc19oaWVyYXJjaHk.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvUkZwODk3YnRPMUNxY3Q3UEozTXlFMC9zYW5kYm94L0E2bWRGTDJNTlRUajloMHhOdTNCYTUtaW1hZ2VzXzE3NzU5MjM0MTUxNDlfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyRmpZMlZ6YzE5b2FXVnlZWEpqYUhrLnBuZyIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5ODc2MTYwMH19fV19&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=UA6PSFtogkkt8QAFkF1N2y8m1AyYtI34VEt1amaFrWJTQ~szOQh7--J2BqcGw-nYppLRaiq22359rJ-HKUjiacT7mS8lcxIfTMEUsI-bXy~Cz-0jnrieFcdrB1eWw7D-2~LTcDnXNQkheVRcdWoMU40UFDffn590XiGReaED9YQQwlxS~CKLbAZAxff0nT0tzNaisYUW5gjYuL6uHnVJVgDTwJUoQT0hTgmc2Bnk0E756BNeY9~HWEuvj8K5lOsuBoSm1T3xiqWJxim-jozYpj0deR5zbtiClzGo7SpBFJGxbxI3vngsMKdar-cLNG970vYORQp2qRIydSHDmw8yvQ__)

---

## BlieverUmaAdapter — Role-Based Access Control

### Roles

| Role Identifier | `keccak256` key | Purpose |
| :--- | :--- | :--- |
| `DEFAULT_ADMIN_ROLE` | `0x00` (OZ default) | Upgrade proxy, grant/revoke roles, update OO address |
| `FACTORY_ROLE` | `keccak256("FACTORY_ROLE")` | Register questions (`initializeQuestion`) |
| `EMERGENCY_ROLE` | `keccak256("EMERGENCY_ROLE")` | Emergency intervention on questions and global pause |

### Role Permissions Matrix

The following matrix maps actors to their respective functional permissions.

![Access Matrix Diagram](https://private-us-east-1.manuscdn.com/sessionFile/RFp897btO1Cqct7PJ3MyE0/sandbox/A6mdFL2MNTTj9h0xNu3Ba5-images_1775923415149_na1fn_L2hvbWUvdWJ1bnR1L2FjY2Vzc19tYXRyaXg.png?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvUkZwODk3YnRPMUNxY3Q3UEozTXlFMC9zYW5kYm94L0E2bWRGTDJNTlRUajloMHhOdTNCYTUtaW1hZ2VzXzE3NzU5MjM0MTUxNDlfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwyRmpZMlZ6YzE5dFlYUnlhWGcucG5nIiwiQ29uZGl0aW9uIjp7IkRhdGVMZXNzVGhhbiI6eyJBV1M6RXBvY2hUaW1lIjoxNzk4NzYxNjAwfX19XX0_&Key-Pair-Id=K2HSFNDJXOU9YS&Signature=s3d4zREcZwaPhMtmDJz47W2skpn7i4fuST4BTXwtPW2Q~EDUMi1g0RFP4frzwv5d1qjHbPtV~5YOvg~Dfzwd8GKH4cgvnPSQEBxH6yrznKz0IYadKOOAsbZHVSxCi2cVKtnhlOjFAJtrU72Qi~sMSsJ2HzJXQjrytax-zTvyUWpn6TCnjww98yCSLaMdPCnNHsHGHsyQKDBs2IxAivD5louABmgdXjoDElWwb~heEEsA-5JqbfYQ65eb3pa60fCte0TYx03WzN3svrt9SPshIKO~EIPp3EVM6WmW6pjNvSy9ZUmAbzuSnLqV12BaWZzzY9Ku6m0pg0q7alQpnQ3Xjw__)

| Function | `DEFAULT_ADMIN` | `FACTORY` | `EMERGENCY` | Anyone |
| :--- | :---: | :---: | :---: | :---: |
| `initializeQuestion()` | | ✓ | | |
| `resolve(questionId)` | | | | ✓ (when ready) |
| `flag(questionId)` | | | ✓ | |
| `resolveManually()` | | | ✓ | |
| `reset(questionId)` | | | ✓ | |
| `pause()` (global) | ✓ | | | |
| `updateOptimisticOracle()` | ✓ | | | |
| `_authorizeUpgrade()` | ✓ | | | |
| `priceDisputed()` (callback) | | | | OO only¹ |
| `_tryResetQuestion()` | | | | self only² |

¹ `onlyOptimisticOracle` modifier — `msg.sender` must be in `_knownOracles`.  
² `BlieverUmaAdapter__OnlySelf` guard — `msg.sender` must be `address(this)`.

---

## Specialized Access Guards

### `onlyOptimisticOracle` Modifier
Applied to `priceDisputed()` only. Accepts callbacks from **any OO address ever assigned** to the adapter. This ensures that after an oracle upgrade, old questions' dispute callbacks are still accepted. `_knownOracles` is append-only to prevent stale callbacks from being permanently blocked.

### `BlieverUmaAdapter__OnlySelf` Guard
This function is `external` (required for Solidity `try/catch` to work) but gated so only the adapter itself can call it. This pattern is used inside `priceDisputed` to enable `try/catch` on internal operations safely.

### UUPS Upgrade Authorization
Only `DEFAULT_ADMIN_ROLE` can execute a proxy upgrade. The UUPS pattern requires the implementation contract itself to enforce upgrade authorization, ensuring that the upgrade cannot be performed even if the proxy's admin storage is manipulated.

---

## BlieverMarket — Address-Based Guards

### Stored Addresses

| Storage Field | Set By | Role |
| :--- | :--- | :--- |
| `resolver` | Factory (at clone `initialize`) | Only address that may call `resolve()` |
| `factory` | Factory (at clone `initialize`) | Only address that may call `pause()`, `unpause()`, `expireUnresolved()` |

These are set once in `initialize()` and never changed. The `resolver` is always `address(BlieverUmaAdapter)`, and the `factory` is always `address(BlieverMarketFactory)`.

### `tradingOpen` Modifier
Applied to `buy()` and `sell()`. Two independent conditions close trading:
1. `resolved == true` — market has been settled.
2. `block.timestamp >= tradingDeadline` — trading deadline reached.

---

## What Cannot Be Paused or Blocked

The following functions have NO access restrictions or pause checks beyond their basic pre-conditions:

| Function | Why it's unconstrained |
| :--- | :--- |
| `market.resolve()` | Markets must always be settleable — oracle results must land |
| `market.claim()` | Winners must always be able to claim — no admin can trap funds |
| `market.expireUnresolved()` | Factory must be able to close dead markets — only time-gated |
| `adapter.resolveManually()` | Emergency resolution must bypass all pauses |

---

## Role Security Properties

- **No single role can steal funds:** `DEFAULT_ADMIN_ROLE` can upgrade the proxy but cannot call `resolveManually()` without also holding `EMERGENCY_ROLE`. `EMERGENCY_ROLE` must wait 1 hour after flagging before resolving.
- **Immutable Resolver:** The resolver address in `BlieverMarket` is immutable post-deploy. An attacker who compromises the adapter proxy cannot retroactively change the `resolver` stored in deployed market clones.
- **Role Separation:** `pause()` (global) requires `DEFAULT_ADMIN_ROLE`, while `pauseQuestion()` (per-question) requires `EMERGENCY_ROLE`. These are distinct roles with distinct addresses.

---

## Error Reference

| Error | Contract | When Thrown |
| :--- | :--- | :--- |
| `NotResolver()` | `BlieverMarket` | `resolve()` called by non-resolver |
| `NotFactory()` | `BlieverMarket` | `pause/unpause/expireUnresolved()` called by non-factory |
| `TradingClosed()` | `BlieverMarket` | `buy/sell()` after deadline or post-resolution |
| `NotOptimisticOracle()` | `BlieverUmaAdapter` | `priceDisputed()` from unknown address |
| `BlieverUmaAdapter__OnlySelf()` | `BlieverUmaAdapter` | `_tryResetQuestion()` from non-self |
| `ZeroAddress()` | Both | Any initializer address param is `address(0)` |
| `AccessControl: account X is missing role Y` | `BlieverUmaAdapter` | OZ role check failure on any `onlyRole` function |
