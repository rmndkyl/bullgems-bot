# Bull Gems Mining Bot (Showcase)

Automation bot for **Bull Games / Bull Gems** (`gem.bullshot.io`) — mining & quest game on **Base Network**, backed by Bullshot (memecoin launchpad, **$7.7M** raised from Animoca Brands, P2 Ventures, AWS, B1, Poolz).

Wallet-signature auth (no captcha, no gas for farming) + full daily loop: **achievements → quests → mining → auto-spend on equipment & upgrades → deploy → referral**.

> This repository is a public showcase (features + setup guide). Full source is private.

---

## What it does

| Feature | Detail |
|---|---|
| **Wallet signature auth** | `POST /auth/nonce` → `personal_sign` → `POST /auth/login` → JWT. No captcha, no OAuth |
| **Auto-claim achievements** | Reads Miner Milestones, claims every unlocked milestone (one-time GEMS) |
| **Auto-buy equipment** | Buys mining gear by level eligibility, cheapest first — mining power is **additive** per owned item |
| **Auto-buy upgrades** | Mining Power / Speed / Energy Capacity, re-reads server cost every purchase, respects caps |
| **Mining automation** | Claims every finished position, then deploys repeatedly until energy hits 0 (multi-position) |
| **Quest automation** | Daily / weekly / one-time `actionType: intent` quests, cooldown-aware retry |
| **Referral claim** | Auto-claims pending referral bonuses (10% of invitees' GEMS) |
| **Multi-wallet** | One private key per line, sequential with jitter delay |
| **Proxy support** | HTTP / SOCKS5, rotated per wallet |
| **401 auto-reauth** | Re-signs and retries transparently |
| **Dry run** | Prints the exact shopping plan without spending a single GEM |

---

## Execution order (why it's built this way)

```
1. Claim achievements      free GEMS, no cooldown
2. Claim quests            daily / weekly / one-time
3. Claim finished mining   positions whose timer expired
4. Spend: equipment → upgrades   funded by steps 1-3
5. Deploy mining           now with the improved power/speed
6. Claim referral
```

Spending happens *after* earning and *before* deploying, so every run mines with the best stats the wallet can afford.

---

## Economy: why Mining Power is the default target

Live values from `/mining/locations`:

| Location | Lvl | Energy | Power req | Duration | GEMS | GEMS/energy |
|---|---|---|---|---|---|---|
| cave-* (Greenstone) | 1 | 5 | 0 | 1 min | 5-15 | 3.00 |
| crystal-cavern | 4 | 20 | 25 | 30 min | 25-45 | 2.25 |
| ember-mine | 8 | 35 | 60 | 90 min | 60-100 | 2.86 |
| **frozen-depths** | 14 | 50 | **120** | 4 h | 200-350 | **7.00** |
| **void-rift** | 20 | 75 | **250** | 8 h | 500-800 | **10.67** |

Caves are the best early game, but `frozen-depths` and `void-rift` pay **2-3.5x more per energy** — they only unlock through mining power. Power upgrades cost a flat 250 GEMS for +5 PWR with no cap, so they are the shortest path upward.

Energy Capacity is deliberately **not** bought by default: it raises the storage ceiling, not the energy income, and the bot spends energy every run.

---

## Setup

```bash
git clone <private-repo>   # buyers only
cd bullgems-bot
npm install
```

1. Copy `config.json.example` → `config.json`.
2. Put wallet private keys in `privateKey.txt`, one per line.
3. (Optional) Put proxies in `proxy.txt`, one per line.
4. Run.

```bash
node index.js                      # interactive menu
node index.js --auto               # full run, no menu
node index.js --auto --dry-run     # preview purchases only
node index.js --wallet=3           # single wallet
```

### Menu

```
1. Run daily  (achievements + quests + mining + auto upgrade + auto buy)
2. Auth only
3. Mining only
4. Quests only
5. Check referral stats
6. Auto upgrade + buy equipment only
7. Dry run (shopping plan, no purchases)
```

### Key config options

| Option | Default | Meaning |
|---|---|---|
| `autoAchievements` | true | Claim unlocked achievements |
| `autoUpgrade` | true | Buy upgrades while GEMS allow |
| `autoBuyEquipment` | true | Buy shop equipment while GEMS allow |
| `upgradeTypes` | `["miningPower"]` | Purchase priority order; add `miningSpeed` / `energyCapacity` |
| `maxUpgradesPerRun` | 10 | Upgrade purchase cap per run |
| `maxEquipmentPerRun` | 6 | Equipment purchase cap per run |
| `gemsReserve` | 0 | GEMS the bot must never spend |
| `maxDeploysPerRun` | 50 | Mining deploy safety cap |

---

## Requirements

| Item | Needed | Cost |
|---|---|---|
| Node.js 18+ | Yes | Free |
| EVM wallet private key(s) | Yes | Free |
| Base ETH | No (off-chain farming) | $0 |
| X account | Only for X quests | Free |
| Proxy | Optional | Optional |

---

## Verified behaviour (live test)

- Achievements: 3-4 claimed per wallet, **+300 to +400 GEMS** on first run
- Equipment: Iron Pickaxe then Iron Drill bought → mining power **2 → 4 → 10** (proves additive stacking)
- Upgrades: Mining Power I x7 → power **60 → 95**, cost fixed at 250, no cap
- Full run on one wallet: 20 claims (+295.8 GEMS), 20 deploys, energy 100 → 0, power 2 → 35

---

## Known limits

- **X (Twitter) quests** need a linked handle in Miner Center — semi-manual by design, server rejects otherwise.
- **Referral claim** only fires when a pending bonus exists.
- Airdrop is **not confirmed**; GEMS have no announced token conversion. Farm at your own risk, only with wallets you control.

---

**Layer Airdrop ID** — [@rmndkyl](https://t.me/rmndkyl)
