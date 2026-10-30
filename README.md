# Bull Games (Bull Gems) Mining Bot

Multi-account automation bot for **Bull Games** (gem.bullshot.io) — the mining & quest game on **Base Network** backed by **Bullshot** (memecoin launchpad, $7.7M raised from Animoca Brands, P2 Ventures, AWS).

Automates wallet auth, daily/weekly check-ins, quests, mining, and referral claims. **No captcha, no gas needed for farming.**

---

## Features

- **Wallet Signature Auth** — automatic login via `personal_sign` (no captcha, no OAuth)
- **Daily Miner Sign-in** — auto-claim daily check-in
- **Weekly Miner Check-in** — auto-claim weekly
- **Quest auto-claim** — TG/Discord/one-time quests (cooldown-aware retry)
- **Mining automation** — auto-start mining at best affordable location + auto-claim rewards per timer
- **Referral claim** — auto-claim pending referral bonuses
- **Multi-wallet** — one private key per line in `privateKey.txt`
- **Proxy support** — optional HTTP/SOCKS5 per wallet (rotate from `proxy.txt`)
- **401 auto-reauth** — re-login on session expiry
- **Cooldown handling** — honors server "verification in progress" wait
- **Colored logging** + per-wallet `logs.log`

---

## Requirements

| Item | Required? | Cost |
|------|-----------|------|
| Node.js v18+ | Yes | Free |
| EVM wallet(s) with private key | Yes | Free |
| Base network ETH (gas) | Only for trading/on-chain actions | ~$0 |
| X (Twitter) account | Only for X quests (optional) | Free |
| Proxy (optional) | No | Optional |

**Cost to farm: FREE** — mining, check-ins, and quest claims are all off-chain. No gas needed.

---

## Setup

```bash
cd bullgems-bot
npm install
```

1. Edit `config.json` (or copy from `config.json.example`).
2. Add your wallet private keys to `privateKey.txt` — one per line.
3. (Optional) Add proxies to `proxy.txt` — one per line (`http://ip:port` or `socks5://ip:port`).
4. Run:
```bash
node index.js
```

---

## Usage / Menu

```
1. Run daily (quests + mining + referral)   <- recommended
2. Auth only (login + profile check)
3. Mining only
4. Quests only
5. Check referral stats
```

### config.json options

| Option | Default | Description |
|--------|---------|-------------|
| `baseUrl` | gemapi.bullshot.io/api | Backend API |
| `referrerCode` | `0x7b9c8f8e` | Referral code sent on login |
| `delayMin` / `delayMax` | 3 / 8 | Seconds between wallets |
| `autoMining` | true | Auto start + claim mining |
| `autoQuests` | true | Auto claim intent quests |
| `autoClaimReferral` | true | Auto claim referral bonus |
| `miningLocation` | `auto` | Specific location id, or auto-pick |
| `timeout` / `maxRetries` | 20000 / 3 | HTTP timeout / retries |
| `proxyFile` | proxy.txt | Proxy list filename |

---

## Project Info

| Field | Value |
|-------|-------|
| Project | Bull Games (Bull Gems) |
| URL | https://gem.bullshot.io |
| Chain | Base (mainnet) |
| Backend | gemapi.bullshot.io/api |
| Token | $BULL (unannounced) |
| Airdrop | **NOT confirmed** — GEMS/XP conversion unannounced |
| Funding | $7.7M (Animoca, P2 Ventures, AWS, B1, Poolz, Becker, OneAM) |
| Active miners | ~794,000 |
| Referral | 10% of invited miners' claimed GEMS |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `Nonce failed ... timeout` | Transient rate-limit. Increase `timeout`, retry. Bot auto-retries. |
| `Session expired or invalid` | 401 → bot auto-reauths and retries. |
| `Verification in progress, wait N` | Server cooldown. Bot waits automatically. |
| `Please link your X (Twitter)` | X quests need a linked handle in Miner Center. Link once in browser. |
| `Banned` | Account flagged. Check ban reason in logs. |

---

## Disclaimer

Airdrop is **not confirmed**. GEMS have no guaranteed token conversion. Farm at your own discretion. This bot is for educational/automation purposes. Only use wallets you control.

---

**Layer Airdrop ID** — Contact: [@rmndkyl](https://t.me/rmndkyl)