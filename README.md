# Bull Games (Bull Gems) Mining Bot

Multi-account automation bot for **Bull Games** (gem.bullshot.io) — the mining & quest game on **Base Network** backed by **Bullshot** (memecoin launchpad, $7.7M raised from Animoca Brands, P2 Ventures, AWS).

Automates wallet auth, daily/weekly check-ins, quests, mining, **achievement claiming**, **equipment purchasing**, **upgrade purchasing**, **XP farming (P2P + swap)**, and referral claims. **No captcha, no gas needed for farming.**

---

## Features

- **Wallet Signature Auth** — automatic login via `personal_sign` (no captcha, no OAuth)
- **XP farm (naik level cepat)** — dua trik yang sudah diverifikasi live (2026-11-07):
  - **P2P Market**: beli listing termurah → **+75 XP per pembelian** (flat). Item yang dibeli otomatis di-relist lagi dengan harga sama, jadi wallet lain di fleet bisa membelinya → XP berputar tanpa GEMS hilang (GEMS cuma pindah wallet, seller dapat penuh tanpa fee).
  - **Swap bolak-balik**: `GEMS → iron-ore → GEMS` → **+5 XP per swap** (flat, tidak tergantung jumlah). Rate 1 GEMS = 4 ore dan 4 ore = 1 GEMS, jadi satu putaran (2 swap) = **10 XP tanpa loss**.
  - Progress level ditampilkan pakai rumus frontend: naik dari level n butuh `round(100 × n^1.5)` XP.
- **Achievement auto-claim** — klaim semua Miner Milestones yang sudah unlocked (one-time GEMS reward; swap farm otomatis membuka `ten_swaps`)
- **Equipment auto-buy** — beli tool di *Machinery & Mining Equipment* sesuai level, urut termurah (mining power bersifat **additif**)
- **Upgrade auto-buy** — beli *Mining Power / Mining Speed / Energy Capacity* selama GEMS cukup, cost dibaca ulang dari server tiap pembelian
- **Daily Miner Sign-in** — auto-claim daily check-in
- **Weekly Miner Check-in** — auto-claim weekly
- **Quest auto-claim** — TG/Discord/one-time quests (cooldown-aware retry)
- **Mining automation** — claim semua posisi ready, lalu **deploy terus-menerus sampai energy habis** (multi-position). Semua tipe lokasi didukung: cave, **Greenstone Quarry, Crystal Cavern, Ember Mine, Frozen Depths, Void Rift** — dipilih otomatis berdasarkan level + mining power + energy (bukan flag `isLocked` API yang tidak akurat).
- **Mining strategy** — `best` (GEMS/energy tertinggi), `fastest` (durasi terpendek), `spread` (rotasi tiap tier supaya dapat ore beragam: sapphire/ruby/diamond/void gem).
- **Referral claim** — auto-claim pending referral bonuses
- **Dry run** — lihat rencana belanja tanpa eksekusi
- **Multi-wallet** — one private key per line in `privateKey.txt`
- **Proxy support** — optional HTTP/SOCKS5 per wallet (rotate from `proxy.txt`)
- **401 auto-reauth** — re-login on session expiry
- **Cooldown handling** — honors server "verification in progress" wait
- **Colored logging** + per-wallet `logs.log`

---

## Urutan eksekusi per wallet

Dioptimalkan supaya GEMS yang baru didapat langsung dipakai:

```
1. Claim achievements   (GEMS gratis, tanpa cooldown)
2. Claim quests         (daily / weekly / oneTime, actionType=intent)
3. XP farm              (swap bolak-balik + beli listing P2P murah)
4. Claim posisi mining yang sudah ready
5. Belanja: equipment  ->  upgrade      (pakai GEMS hasil langkah 1-3)
6. Deploy mining        (pakai mining power / speed yang baru)
7. Claim referral
```

Urutan fase "earn" (1,2,3,4,7) diacak per wallet; XP farm selalu berada di dalam blok earn sehingga
tidak pernah bentrok dengan fase belanja.

---

## XP farm — naik level cepat

Dua sumber XP yang sudah diuji langsung ke backend produksi (2026-11-07):

| Aksi | XP | Biaya | Catatan |
|------|----|-------|---------|
| `POST /market/listings/{id}/buy` | **+75 XP** | harga listing | flat per pembelian, tidak tergantung harga/amount |
| `POST /trade/swap` | **+5 XP** | **0** (lossless) | flat per swap, tidak tergantung amount |

### Tips 1 — P2P Market beli yang murah

- Bot scan `/market/listings`, saring `status=active`, buang listing milik wallet sendiri,
  ambil harga termurah yang masih di bawah `maxBuyPrice`.
- Setiap pembelian = +75 XP. Listing 2 GEMS → 37.5 XP per GEMS, listing 0.3 GEMS → 250 XP per GEMS.
- Setelah dibeli, item **otomatis di-relist** dengan harga yang sama (`relist: true`).
  Wallet lain di fleet bisa membeli item itu → +75 XP lagi. GEMS hanya berpindah antar wallet,
  seller menerima penuh (tanpa fee) — jadi satu item bisa memutar XP berkali-kali.
- Listing bisa disambar pemain lain; bot akan skip dengan pesan `Market listing not found or already sold`.

### Tips 2 — Swap bolak-balik (gratis, tanpa batas)

- Pilih pasangan rate lossless dari `/trade/rates`: `GEMS → iron-ore` = 1 : 4, `iron-ore → GEMS` = 4 : 1.
- Satu putaran = 2 swap = **10 XP**, modal balik ke angka semula (tidak ada loss).
- Bot otomatis berhenti kalau hasil balik < 97% (rate berubah) supaya GEMS tidak drain.
- Kalau salah satu swap timeout, bot cek inventory dan menukar balik kelebihan asset (maks 1 leg),
  jadi ore yang sudah ada dari mining tidak ikut tertukar.

### Kurva level

Rumus dari bundle frontend: XP untuk naik dari level **n** ke **n+1** = `round(100 × n^1.5)`.

| Level | XP ke level berikutnya | Total XP kumulatif |
|-------|------------------------|--------------------|
| 10 | 3,162 | 11,106 |
| 15 | 5,809 | 31,998 |
| 22 | 10,319 | 85,702 |
| 30 | 16,432 | 189,029 |

Artinya: 10 putaran swap = 100 XP. Untuk 1 level di level 22 (±10,319 XP) butuh ±1,032 putaran swap,
atau ±138 pembelian P2P. Kombinasi keduanya paling cepat: swap jalan terus (gratis) sambil P2P
menyambar listing murah yang muncul.

---

## Mining — semua tipe lokasi

`GET /mining/locations` mengembalikan 8 lokasi. **Penting:** field `isLocked` dari API tidak bisa
dipercaya — Crystal Cavern, Ember Mine, dan Frozen Depths tetap `isLocked: true` walaupun level dan
power sudah cukup. UI game sendiri hanya mengecek `user.level >= levelRequired`, dan tombol Deploy
hanya butuh energy + mining power. Uji live: wallet level 14 / 360 PWR berhasil deploy ke ketiga
lokasi tersebut (HTTP 200, posisi dibuat). Bot memakai logika frontend itu.

| Lokasi | Lvl | Energy | Power | Durasi | GEMS | Ore | GEMS/E |
|--------|-----|--------|-------|--------|------|-----|--------|
| cave-* (3 lokasi) | 1 | 5 | 0 | 1m | 5-15 | — | 3.00 |
| greenstone-quarry | 1 | 10 | 0 | 10m | 8-15 | Iron Ore, Coal, Emerald | 1.50 |
| crystal-cavern | 4 | 20 | 25 | 30m | 25-45 | Copper Ore, Iron Ore, Sapphire | 2.25 |
| ember-mine | 8 | 35 | 60 | 1h 30m | 60-100 | Sulfur, Coal, Ruby | 2.86 |
| frozen-depths | 14 | 50 | 120 | 4h | 200-350 | Cryo Shard, Diamond, Obsidian | **7.00** |
| void-rift | 20 | 75 | 250 | 8h | 500-800 | Void Dust, Void Gem, Obsidian | **10.67** |

### Strategy (`miningStrategy`)

| Nilai | Perilaku | Kapan dipakai |
|-------|----------|---------------|
| `best` (default) | GEMS per energy tertinggi yang **terjangkau** | farming GEMS jangka panjang |
| `fastest` | durasi terpendek dulu | butuh putaran cepat / achievement jumlah mining |
| `spread` | rotasi tiap tier (cave → greenstone → crystal → ember → frozen → void) | kumpulkan ore beragam untuk craft/refine/quest |

Lihat semua lokasi + status kelayakan wallet tanpa deploy:
```bash
node index.js --locations            # tabel lengkap + pilihan tiap strategy
node index.js --locations --wallet=3
```

Contoh output:
```
status  lokasi                    lvl  energy  power  durasi   GEMS        ore
READY   crystal-cavern            4    20      25     30m      25-45       Copper Ore/Iron Ore/Sapphire
READY   ember-mine                8    35      60     1h 30m   60-100      Sulfur/Coal/Ruby
energy kurang  frozen-depths      14   50      120    4h       200-350     Cryo Shard/Diamond/Obsidian
butuh lvl 20   void-rift          20   75      250    8h       500-800     Void Dust/Void Gem/Obsidian
```

### Hasil claim — terukur live (2026-11-08, wallet multiplier x2.6)

| Lokasi | Energy | Waktu | GEMS base | GEMS diterima | Ore per claim | XP |
|--------|--------|-------|-----------|---------------|---------------|-----|
| cave | 5 | 1m | 6 | 15.6 | — | kecil |
| greenstone-quarry | 10 | 10m | 15 | 39 | 16 Iron Ore, 5 Coal | **+25** |
| crystal-cavern | 20 | 30m | 28-40 | 72.8-104 | 22-29 Copper Ore, 13 Iron Ore, 3 Sapphire | **±60** |

Temuan penting:
- **Multiplier berlaku pada GEMS**, bukan ore. Wallet contoh punya x2.6 (daily/weekly/one-time quest bonus), jadi angka di tabel adalah `rewardRanges × multiplier`.
- **Claim mining memberi XP juga**, bukan cuma GEMS — sekitar 1.7-1.8 XP per base GEM (greenstone base 15 → +25 XP; 2× crystal base 40+28 → +120 XP). Jadi tier tinggi = GEMS besar + ore mahal + XP besar sekaligus.
- Ore ikut masuk inventory dan bisa dijual di P2P. Nilai pasar ore per claim crystal ±58 GEMS (Copper 0.6, Iron 0.25, Sapphire 3.5), jadi total nilai satu claim crystal ±130-160 GEMS untuk 20 energy.
- Bot sekarang mencatat ore di log dan ringkasan: `mining claim OK: ... (+104 GEMS @x2.6 | 29x Copper Ore, 13x Iron Ore, 3x Sapphire)` dan `ore=Copper Ore x51, Sapphire x6` di summary.

Kesimpulan: `best` (tier tertinggi yang eligible) tetap pilihan terbaik — cave hanya menang tipis di GEMS/energy mentah, tapi kalah total karena tanpa ore dan XP jauh lebih kecil.

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
1. Run daily  (achievements + quests + xp farm + mining + auto upgrade + auto buy)   <- recommended
2. Auth only  (login + profile check)
3. Mining only
4. Quests only
5. Check referral stats
6. Auto upgrade + buy equipment only (no mining/quests)
7. Dry run    (lihat rencana belanja, tidak ada pembelian)
8. XP farm    (swap bolak-balik + beli listing P2P murah)
9. XP swap only    (5 XP/swap, lossless, tanpa belanja)
10. XP market only (beli listing murah @75 XP per pembelian)
11. Lihat daftar lokasi mining + status kelayakan (lvl/power/energy)
```

### CLI flags (non-interactive, untuk scheduler / cron)

```bash
node index.js --auto              # full run tanpa menu
node index.js --xp-only           # XP farm saja (swap + P2P)
node index.js --xp-swap-only      # swap bolak-balik saja (gratis)
node index.js --xp-p2p-only       # beli listing murah saja
node index.js --xp-swaps=100      # override jumlah swap run ini
node index.js --xp-buys=20        # override jumlah pembelian run ini
node index.js --shop-only         # achievements + upgrade + buy equipment saja
node index.js --mining-only
node index.js --mining-only --strategy=spread   # rotasi semua tier lokasi (ore beragam)
node index.js --strategy=fastest                # durasi terpendek dulu
node index.js --locations                        # tabel lokasi + status kelayakan, tanpa deploy
node index.js --quests-only
node index.js --auth-only
node index.js --dry-run           # gabung dengan flag lain: preview rencana
node index.js --wallet=3          # jalankan hanya wallet #3 (1-based)
node index.js --no-upgrade --no-buy --no-xp   # matikan sebagian fitur
```

Contoh aman sebelum run besar:
```bash
node index.js --auto --dry-run --wallet=1
node index.js --xp-only --wallet=1 --xp-swaps=6 --xp-buys=2
```

> Catatan: `--dry-run` membatalkan pembelian P2P, swap, equipment, dan upgrade. Claim achievement,
> quest, dan mining tetap dijalankan karena tidak merugikan (menambah GEMS/XP).

---

### Jadwal otomatis (Windows)

Dua wrapper sudah tersedia di folder bot:

| File | Isi | Log |
|------|-----|-----|
| `run-daily.bat` | `node index.js --auto` (semua wallet) | `daily-run.log` |
| `run-xp.bat` | `node index.js --xp-only` (semua wallet) | `xp-run.log` |

Sudah didaftarkan di Task Scheduler (jalan tiap hari 08:00, hanya saat user login):

```bat
schtasks /Create /TN "LYID-BullGems-Daily" /TR "\"C:\Users\raman\LYID-BOTS\bullgems-bot\run-daily.bat\"" /SC DAILY /ST 08:00 /F
```

Cek / ubah / hapus:
```bat
schtasks /Query /TN "LYID-BullGems-Daily" /FO LIST
schtasks /Change /TN "LYID-BullGems-Daily" /ST 20:00
schtasks /Delete /TN "LYID-BullGems-Daily" /F
```

Argumen bisa ditambahkan lewat wrapper, mis. `run-daily.bat --wallet=3 --dry-run`.
Catatan: `--auto` menjalankan 37 wallet berurutan (±2-3 jam termasuk XP farm), jadi beri waktu sebelum run berikutnya.

### config.json options

| Option | Default | Description |
|--------|---------|-------------|
| `baseUrl` | gemapi.bullshot.io/api | Backend API |
| `referrerCode` | `0x0bd665c7` | Referral code sent on login |
| `delayMin` / `delayMax` | 3 / 8 | Seconds between wallets |
| `autoMining` | true | Auto start + claim mining |
| `autoQuests` | true | Auto claim intent quests |
| `autoAchievements` | true | Auto claim achievement yang unlocked |
| `autoUpgrade` | true | Auto beli upgrade (Mining Power / Speed / Energy) |
| `autoBuyEquipment` | true | Auto beli equipment di shop |
| `autoClaimReferral` | true | Auto claim referral bonus |
| `autoXpFarm` | true | Nyalakan fase XP farm di run harian |
| `xpFarm.mode` | `both` | `both` / `swap` / `p2p` |
| `xpFarm.maxSwaps` | 40 | Batas swap per run (2 swap = 1 putaran = 10 XP) |
| `xpFarm.swapAsset` | `auto` | `auto` = pilih pasangan rate lossless (iron-ore); bisa isi `emerald`, `iron-ore`, dll |
| `xpFarm.swapAmount` | 1 | GEMS per leg swap (harus bulat) |
| `xpFarm.maxBuys` | 10 | Batas pembelian listing P2P per run (75 XP masing-masing) |
| `xpFarm.maxBuyPrice` | 3 | Harga listing maksimum yang mau dibeli (GEMS) |
| `xpFarm.budgetGems` | 30 | Budget maksimum untuk P2P per run |
| `xpFarm.gemsReserve` | 20 | GEMS yang disisakan untuk upgrade/mining (tidak dipakai XP farm) |
| `xpFarm.relist` | true | Relist item hasil beli dengan harga sama supaya wallet lain bisa beli (XP berputar) |
| `xpFarm.targetLevel` | 0 | Berhenti farm kalau level ini tercapai (0 = tanpa batas) |
| `upgradeTypes` | `["miningPower"]` | Jenis upgrade yang dibeli, **urutan = prioritas**. Bisa tambah `"miningSpeed"`, `"energyCapacity"` |
| `maxUpgradesPerRun` | 6 | Batas pembelian upgrade per run |
| `maxEquipmentPerRun` | 6 | Batas pembelian equipment per run |
| `maxEquipmentPrice` | 35000 | Harga equipment maksimum yang mau dibeli |
| `includeZeroPowerEquipment` | false | Ikut beli equipment +0 PWR (mis. Rusty Pickaxe) |
| `gemsReserve` | 0 | GEMS yang **tidak boleh** dipakai belanja |
| `tokenCache` | true | Simpan JWT di `tokens.json`, login hanya kalau token hampir habis (JWT berlaku 72 jam) |
| `tokenMinHoursLeft` | 6 | Ambang minimal sisa umur token sebelum bot login ulang |
| `humanize.jitterSigma` | 0.55 | Lebar sebaran jeda (log-normal). Makin besar makin banyak jeda panjang |
| `humanize.pauseChance` | 0.35 | Peluang berhenti lama sekali per run (20-90 detik) |
| `humanize.deployStopChance` | 0.4 | Peluang berhenti deploy di 70-90% energy, bukan selalu 0 |
| `humanize.randomPhaseOrder` | true | Urutan fase diacak per wallet (deterministik per wallet) |
| `miningLocation` | `auto` | `auto` = pilih lokasi GEMS/energy terbaik; atau isi id lokasi spesifik |
| `miningStrategy` | `best` | `best` (GEMS/energy) / `fastest` (durasi terpendek) / `spread` (rotasi tiap tier, ore beragam) |
| `maxDeploysPerRun` | 50 | Batas deploy per run (safety, cegah loop tak berujung) |
| `timeout` / `maxRetries` | 20000 / 3 | HTTP timeout / retries |
| `proxyFile` | proxy.txt | Proxy list filename |

---

## Anti-detection / humanisasi

Backend Bull Gems tidak pakai captcha dan API-nya tidak challenge request (Cloudflare hanya di depan,
`/mining/stats` balas 200 tanpa auth). Jadi yang berisiko bukan "ketahuan bot" gaya WAF, tapi
**pola perilaku** yang bisa dibaca dari log server. Empat hal yang sudah diperbaiki:

| Sebelum | Sesudah |
|---|---|
| Login ulang tiap run (nonce → sign → login tiap kali) | JWT di-cache 72 jam di `tokens.json`, login ~1x per 2.5 hari |
| Header: hanya UA Chrome, tanpa `sec-ch-ua` / `sec-fetch-*` | Header lengkap & konsisten (Chrome 131 + `sec-ch-ua`, `accept-language`, `sec-fetch-site/mode/dest`, `priority`) |
| Jeda seragam 0.7–1.4 detik | Log-normal (kebanyakan cepat, sesekali lambat), plus pause 20–90 detik sesekali |
| Deploy selalu sampai energy 0, urutan fase sama untuk semua wallet | 40% run berhenti di 70–90% energy, urutan fase diacak per wallet (12 kemungkinan) |

`tokens.json` berisi bearer token aktif → **wajib gitignored** (sudah masuk `.gitignore`).
Matikan humanisasi kalau butuh run cepat: `humanize.pauseChance: 0`, `humanize.randomPhaseOrder: false`.

Yang **belum** ditangani dan tetap jadi risiko terbesar: semua wallet berjalan dari satu IP.
Kalau mau serius soal clustering, proxy per wallet (1 wallet = 1 IP sticky) lebih menentukan
daripada semua poin di atas.

---

## Strategi ekonomi (kenapa default-nya begini)

GEMS per energy tiap lokasi (dibaca live dari `/mining/locations`):

| Lokasi | Lvl | Energy | Power req | Durasi | GEMS | GEMS/E (max) |
|--------|-----|--------|-----------|--------|------|--------------|
| cave-* (Greenstone) | 1 | 5 | 0 | 1 mnt | 5-15 | 3.00 |
| crystal-cavern | 4 | 20 | 25 | 30 mnt | 25-45 | 2.25 |
| ember-mine | 8 | 35 | 60 | 90 mnt | 60-100 | 2.86 |
| **frozen-depths** | 14 | 50 | **120** | 240 mnt | 200-350 | **7.00** |
| **void-rift** | 20 | 75 | **250** | 480 mnt | 500-800 | **10.67** |

Kesimpulan: **Mining Power adalah target utama.** Cave memang paling efisien di awal, tapi `frozen-depths` (7.0 GEMS/E) dan `void-rift` (10.67 GEMS/E) butuh power 120 / 250. Upgrade power = 250 GEMS untuk +5 PWR (bisa diulang tanpa batas, harga tetap), jadi ini jalur tercepat ke tier atas.

Equipment bersifat **additif** (terbukti live: Iron Pickaxe +2 lalu Iron Drill +6 → power 2 → 4 → 10), jadi bot membeli semua tier yang terjangkau urut dari termurah. Level lock tiap item: `iron-pickaxe 1`, `iron-drill 3`, `copper-turbo-drill 6`, `ruby-laser 10`, `cryo-plasma-drill 15`, `void-annihilator 20`.

Energy Capacity default **tidak** dibeli: kapasitas tidak menambah energy, hanya batas simpan — dan bot menghabiskan energy tiap run. Aktifkan manual lewat `upgradeTypes` kalau ingin menabung energy.

---

## API yang dipakai (terverifikasi live)

| Method | Path | Body | Catatan |
|--------|------|------|---------|
| GET | `/achievements` | — | `{achievements:[{id,title,goal,reward,progress,unlocked,claimed,claimable}]}` |
| POST | `/achievements/claim` | `{achievementId}` | return `{success, rewardGems, user}` |
| GET | `/shop/items` | — | `[{id,name,type,price,effect:{miningPower}}]` |
| POST | `/shop/purchase` | `{itemId}` | auto-equip; power additif |
| GET | `/upgrades` | — | `[{id,type,increment,cost,isCapped}]` |
| POST | `/upgrades/{id}` | — | repeatable; harga power tetap 250 |
| GET | `/trade/rates` | — | `{rates:{"GEMS_iron-ore":4,"iron-ore_GEMS":0.25,...}, assetValues:{...}}` |
| POST | `/trade/swap` | `{fromAssetId,toAssetId,amount}` | **+5 XP**, return `{success,received,user}` |
| GET | `/market/listings` | — | `[{id,sellerWallet,assetId,amount,price,status}]` — `price` = total, bukan per unit |
| POST | `/market/listings/{id}/buy` | — | **+75 XP** ke buyer, seller dapat GEMS penuh (no fee) |
| POST | `/market/listings` | `{assetId,amount,price}` | bikin listing baru |
| POST | `/market/listings/{id}/cancel` | — | batal listing |

### Bukti uji XP (2026-11-07)

| Skenario | Hasil |
|---|---|
| Swap 1 GEMS → 4 iron-ore | `+5 XP`, gems 79.1 → 78.1 |
| Swap 4 iron-ore → GEMS | `+5 XP`, gems 78.1 → 79.1 (kembali persis) |
| 8 swap beruntun tanpa jeda | semua `200`, `+40 XP`, gems tidak berubah |
| Beli listing coal 1 @ 2 GEMS | `+75 XP`, gems −2 |
| Wallet A list coal @2 → Wallet B beli | B `+75 XP` / −2 GEMS, A `+2 GEMS` / 0 XP (no fee) |
| Run bot `--xp-only --xp-swaps=6 --xp-buys=2` | `+180 XP`, 6 swap + 2 beli, 2 item berhasil direlist |

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
| `equipment: semua tier ... sudah dimiliki` | Normal — tidak ada item baru yang eligible di level itu. |
| `upgrade: tidak ada yang terjangkau` | GEMS tidak cukup untuk upgrade berikutnya. Normal. |
| Belanja terlalu boros | Naikkan `gemsReserve`, atau turunkan `maxUpgradesPerRun` / `maxEquipmentPerRun`. |
| `Banned` | Account flagged. Check ban reason in logs. |

---

## Disclaimer

Airdrop is **not confirmed**. GEMS have no guaranteed token conversion. Farm at your own discretion. This bot is for educational/automation purposes. Only use wallets you control.

---

**Layer Airdrop ID** — Contact: [@rmndkyl](https://t.me/rmndkyl)
