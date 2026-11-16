# OPTIMJACK — The Chain That Broke a CDN

**A supply-chain attack on Optimizely's tag CDN (November 2026) — one chain, seven links: injection → payload → iOS exploit kit → infrastructure → money → remediation → attribution.**

> **Reading guide.** Every link in this chain is written as it was found. Facts carry a confidence tag:
> - ✅ **Confirmed** — verified against live data fetched as *data* (never executed), public blockchains, vendor pages, or third-party scan corpora.
> - ⚠️ **Unconfirmed / hypothesis** — plausible, but not proven. Kept as-is, not dressed up.
> - ❌ **Ruled out** — tested and shown false within the evidence we have.
>
> No fabrication. Where we don't know, the chain says so. Where we know, the chain says how.

---

## Link 1 — The trust chain breaks

A website that runs Optimizely loads JavaScript from `cdn.optimizely.com` and trusts it. That is the entire premise of a tag manager: the CDN is the quiet corner of the supply chain nobody audits — a few hundred bytes of third-party script, loaded on some of the busiest pages on the internet.

On **16 November 2026**, that trust broke.

Attacker JavaScript was found **appended inside Optimizely-served tag bundles** — inside the very files a site requests as its trusted vendor code. ✅ Confirmed by direct retrieval of the poisoned bundles and by public analysis (amirulcyber.com, 16 Nov 2026 — the primary public write-up we worked from).

The injection was not a separate malicious file slipped into a page. It was a **replacement of the trusted artifact itself**: sites that did everything right — pinned CDN, subresource integrity on their own assets, modern TLS — still shipped the attacker's code, because the trust anchor (the vendor CDN) was the compromised link.

> The story of this attack is one chain with seven links. Breaking any one of them would have changed the outcome. We walked each link to its end.

## Link 2 — The injection chain

The injected bundle code does something small and lethal: it appends a **loader** that fetches a single remote script.

```
Optimizely bundle (trusted) ──► appended loader ──► https://1688360.cc/track.js
```

✅ Confirmed. The loader bootstrap was captured in the live bundles; the remote script `track.js` was served by the C2 throughout the incident and was still being served (HTTP 200) at 22:54Z and 23:02Z on 16 November — hours after the purge of the bundles themselves.

The C2 host that served the loader is not a hardened criminal box. It is a **same-day registration**: `1688360.cc` was registered **05:23Z on 16 November** — the same hour the poisoning went live. ✅ Confirmed via registration/scan telemetry. It resolves to a Google Cloud IP (`136.113.106.136`, AS396982, `*.bc.googleusercontent.com` PTR) serving nginx with a TLS certificate issued at **05:04:54Z** — minutes after domain registration, minutes before the attack went live. The root page is a catch-all Chinese-language decoy titled `闲言碎语` ("chit-chat") — a placeholder that answers everything and explains nothing. ✅ Confirmed.

The workers inside the injected code **patch the exploit kit's default C2** (`k68k.xyz`) to the campaign domain `eil786npto9u5bq.icu` at runtime. ✅ Confirmed by decoding the served payloads. That one detail tells you the operators took an off-the-shelf kit and pointed it at their own infrastructure — they did not write the payload; they **deployed** it.

## Link 3 — The payload chain

`track.js` is not one thing. It is a **capability stack**, and this is what makes the campaign serious:

1. **Crypto address clipper** — the code watches clipboard and form content for wallet addresses and swaps them for attacker addresses (the wallet set in Link 5). ✅ Confirmed from live payload analysis.
2. **Card skimmer** — form-grabbing for payment details, served alongside `card.php` and `hit.php` exfil endpoints on the C2 (both answered 204 No Content when probed). ✅ Confirmed from C2 endpoint inventory.
3. **Screenshot / cookie exfil** — the C2 also serves `html2canvas.min.js`, a screenshot library — the exfil assembly line includes page capture. ✅ Confirmed from C2 file inventory.
4. **iOS second stage — an exploit kit.** This is the headline. The "skimmer" narrative in early coverage was incomplete: the chain does not stop at stealing a credit card. ✅ Confirmed from live payload retrieval.

### The iOS chain (DarkSword-style kit)

```
boot.js
  └─ A8 ─► B9 / C10 / D11 / D12          (obfuscated loader stages)
        └─ rce_loader.js / rce_loader_v5.js
              └─ rce_worker_18.{4,5,6,7}.js   (JSC + GPU RCE offsets)
                    └─ GhostWave harness
                          └─ post_exploit.js / native_bridge.js / nb_daemon.dylib
                                └─ PE into mediaplaybackd / SpringBoard
                                      └─ Coruna stage1 (cassowary / terrorbird /
                                         bluebird / jacurutu)  [iOS 15.2–17.2.1]
```

✅ Confirmed from live fetches of the staged payloads (files retrieved as data; **none executed**, no state-changing requests). The same file names and RCE-offset structure match public DarkSword reporting (Lookout, DarkReading). The trigger file `evil_43818.heic` was staged on **20 October** — almost a month before the Optimizely CDN was hit. ✅ Confirmed from payload inventory.

Two infrastructure quirks confirm the operators were **actively managing** the kit in real time:
- `5vobauua3ojpklx.icu/boot.js` was **pulled (empty)** while `eil786npto9u5bq.icu` still served — a live stage-rotation, visible hour-by-hour. ✅ Confirmed.
- The obfuscated `index.js` leaked a **brand-new C2 domain** (`l7bcsnwc7nzr3uo.icu`) that was **not** in the public article. ✅ Confirmed — our first novel finding beyond the blog.

One negative result, recorded honestly: deep deobfuscation (base64/hex) of `index.js`, the A8 loader and all DarkSword files surfaced **no additional hidden C2 hosts** — the obfuscated strings are BigInt anti-debug math, and the only domain swap is `k68k.xyz` → `eil786npto9u5bq.icu`. ❌ Ruled out: "more hidden C2s hiding in the obfuscation." The domain set stands as listed.

## Link 4 — The infrastructure chain

Pre-positioning was the tell. This was not an opportunistic hijack; the operator **built the rails before the train ran**:

| Date (2026) | Artifact | Note |
|---|---|---|
| 6 Oct | `netvigator.cc` (Spaceship) | cluster host, pre-positioned 41 days out |
| 20 Oct | `evil_43818.heic` | DarkSword trigger file staged |
| 28 Oct | `l7bcsnwc7nzr3uo.icu` (Dynadot) | telemetry host, 19 days before the CDN hit |
| 9 Nov | `5vobauua3ojpklx.icu` (Global Asset Domains) | iOS second-stage host |
| 12 Nov | first on-chain wallet activity | before public disclosure |
| 14 Nov | `moiubpagedocs.cc` (Dynadot) | cluster host |
| 16 Nov 05:23Z | `1688360.cc` + `eil786npto9u5bq.icu` | registered **same day**; poisoning went live same hour |
| 16 Nov ~15:00Z | public disclosure | |
| 16 Nov 17:56Z | payloads re-uploaded | attacker kept working *after* being exposed |
| 16 Nov 21:20Z | attacker files still being edited | |
| 16 Nov 21:29–21:38Z | bundles purged | remediation window closes |

✅ All confirmed from registration records, scan corpora and live retrieval timestamps.

**Victim surface.** Third-party scan corpus (urlscan) for `1688360.cc` grew past **230 scans** at ~9/hour during peak analyst coverage, and contacted-domain scans confirmed poisoned bundles reaching: `zoom.com`, `gitlab.com`, `smartsheet.com`, `scheels.com`, `klm.com`, `cnn.com`, `americangirl.com`, `hootsuite.com`, `intuit.com`, `managebuilding.com`, `loftliving.com`, `ow.ly`, `bbc.com` — plus cluster hosts `moiubpagedocs.cc` / `blogforge.vu`. ✅ Confirmed via scan corpus. The BBC observation is the cleanest pin on the exposure window: an Intuit marketing link scan at **19:10Z** showed the BBC page still contacting `1688360.cc` (25–32 unique IPs, 138–180 requests) — the poisoned bundle was still being served to a top-10 global news site **4 hours after disclosure**.

## Link 5 — The money chain

The skimmer wallets are not static bait. They are **live consolidation infrastructure**, and on-chain forensics showed the gang working in real time.

**ETH** — collector `0xC4026b06D82b0F0B363eda0642D61CfbDc17884e`:
- First activity **12 Nov** — before disclosure. ✅ Confirmed on-chain.
- **16 Nov 13:03Z**: swept **4,310.8 USDC** to sweep address `0x7005b45DE07e1b70DCD38eE3A708CFD88A6c7f8e`. ✅
- 10:00Z the same day: approved 4,310.8 USDC to DEX router `0xc1d13492285eb664951E201BF7C80c7c6318a1b5` (swap path — laundering via decentralized exchange). ✅
- 14 Nov: 299.9 USDC to `0xfb6a6A1cDc9B01dea50aab8E85Da9Ab012E9B3Cf`. ✅

**TRX** — collector `THvs6p56XkUiMkJeSc51cAP64u2RsTYqxa`:
- **Wallet created 2026-03-15** — eight months before the Optimizely campaign. ⚠️ Unconfirmed whether the same operators ran a longer skimmer/clipper operation under this wallet set; the wallet predates the campaign, which is at least strong circumstantial evidence.
- **16 Nov 15:45Z**: swept **546.8 USDT** to sweep address `TD64NDwZfQB6UASW2Xz6hWrz1T12p1pxL7`. ✅
- Took in 350 + 175 + 100 + 16 USDT from four distinct addresses the same day. ✅

**BTC** — accumulator `19g9rZcPeiz8rUvyWpTRyXXhPnjsWrc3kW`:
- ~0.085 BTC (~8.5M sats, 7 UTXOs) from multiple sources, inflows confirmed all day 16 Nov (blocks 976113→976180). ✅
- **No sweep yet** as of the last check — the consolidation engine had not touched it. ✅

**TON** — uninitialized, zero balance (the public article listed its raw address). ✅ Confirmed.

### The chain stayed live after disclosure

Hours after the bundles were purged and the story went public, the money engine kept running (checks 23:03Z, 16 Nov):
- ETH sweep `0x7005b45...` had moved **0.829 ETH to HitBTC's hot wallet** (`0x80787af194C33b74a811f5e5c549316269d7Ee1A`) at **19:40Z** — an exchange deposit, i.e. a cash-out path — and took in ~2.2 ETH in dust inflows at 22:56–23:01Z. ✅ Confirmed on-chain.
- TRX sweep `TD64NDwZ...` received **~13,031 USDT from five distinct senders** within ~10 minutes (including a single **11,550 USDT** transfer at 22:55Z). ✅ Confirmed on-chain.
- BTC accumulator gained another 0.0023 BTC at 22:37Z. ✅

The pattern: **sweep → DEX-swap or exchange deposit → fresh consolidation addresses**. The wallet set we know is only the visible crust; fresh sweep addresses to watch: `0x7005b45DE07e1b70DCD38eE3A708CFD88A6c7f8e`, `0xfb6a6A1cDc9B01dea50aab8E85Da9Ab012E9B3Cf`, `TD64NDwZfQB6UASW2Xz6hWrz1T12p1pxL7`. ✅ Confirmed.

## Link 6 — The remediation chain

Optimizely's response, measured from the outside:

- **19:10Z** — poisoned bundle still served to BBC visitors (urlscan). 
- **21:29Z / 21:38Z** — the two named bundles (`js/5113954737848320.js` rev 1796, `js/6395285578973184.js` rev 29) were **republished clean**. ✅ Confirmed by direct data-only fetch at 22:20Z: **zero occurrences** of `1688360` in the served bundles.
- **Loader purge window ≈ 19:10Z → 21:29Z (~2.5 hours)**. ✅ Confirmed. Caches may still have served stale copies after that — sites had to re-verify, not assume.
- **Status-page incident 31955513** ("Recommendations tracking/reporting", 16:29Z → 22:08Z) is the only public trace of the incident on Optimizely's status page. ✅ Confirmed.
- **No public security advisory** as of 23:03Z 16 Nov — the official Security Announcements center listed only unrelated advisories (PHP RCE 22 Oct, .NET Brotli 2 Nov, Node.js 27 Oct). ✅ Confirmed by direct fetch. ⚠️ This is a vendor-posture observation, not a criticism: advisories take time, and the purge was the substance.

**Sites affected must not stop at "bundle is clean."** The exposure window means any visitor between ~19:10Z and ~21:29Z on 16 Nov may have executed the loader. That is the remediation boundary for incident response teams: **assume compromise of anything that ran the bundle in the window**, rotate, inspect for the wallet-clipper side effects, and check iOS devices for the second stage.

## Link 7 — The attribution chain

What the evidence supports, and what it does not:

- ✅ **Confirmed**: the C2 root page is Chinese-language; the `JIUJIU888` directory marker on the iOS second-stage host is a **Chinese-language operator handle**. These are markers, not proof of nationality or state sponsorship — the DarkSword kit itself is sold as a commercial spyware/exploit service.
- ⚠️ **Unconfirmed**: whether this campaign is run by a state-linked group, a commercial spyware reseller, or a criminal gang. The clipper/skimmer money chain (Link 5) is criminal, not espionage — but the iOS RCE capability is the kind of thing states buy. We do not know how they meet.
- ⚠️ **Unconfirmed**: connection to Optimizely's **23 Feb 2026 breach by ShinyHunters** (vishing, business-contact data). Different actor profile (data-extortion MO vs JS supply-chain + RCE gang), different entry method. **Do not conflate.** Whether any prior access or stolen credentials resurfaced in this tag injection is unknown and should be asked of Optimizely directly.
- ❌ **Ruled out**: hidden second C2 family inside the obfuscation (Link 3). The panel hunt found **no clearnet admin panel** on the known C2 hosts — the panel is on unreported infrastructure, onion-only, or the operation is API-driven without a panel. Unresolved. ⚠️

## The IOC chain (for defenders)

**Domains**
```
1688360.cc            primary C2 / exfil (GCP AS396982, 136.113.106.136)   ✅
eil786npto9u5bq.icu   iOS second-stage host (JIUJIU888 marker)             ✅
5vobauua3ojpklx.icu   iOS second-stage host (boot.js, later pulled)        ✅
l7bcsnwc7nzr3uo.icu   telemetry C2 leaked from obfuscated index.js         ✅
moiubpagedocs.cc      cluster host                                         ✅
netvigator.cc         cluster host (since 6 Oct)                           ✅
blogforge.vu          cluster host                                         ⚠️
k68k.xyz              kit default C2 (patched at runtime)                  ✅
```

**Host / IP**
```
136.113.106.136       C2 IP (GCP, ptr *.bc.googleusercontent.com)          ✅
```

**Wallets (all live consolidation infrastructure as of checks)**
```
0xC4026b06D82b0F0B363eda0642D61CfbDc17884e   ETH collector   ✅
0x7005b45DE07e1b70DCD38eE3A708CFD88A6c7f8e   ETH sweep (→ HitBTC) ✅
0xfb6a6A1cDc9B01dea50aab8E85Da9Ab012E9B3Cf   ETH recipient    ⚠️
THvs6p56XkUiMkJeSc51cAP64u2RsTYqxa           TRX collector (created Mar 2026) ✅
TD64NDwZfQB6UASW2Xz6hWrz1T12p1pxL7           TRX sweep        ✅
19g9rZcPeiz8rUvyWpTRyXXhPnjsWrc3kW           BTC accumulator  ✅
0x80787af194C33b74a811f5e5c549316269d7Ee1A   HitBTC hot wallet (cash-out sink) ✅
```

**File / path markers** — `track.js`, `card.php`, `hit.php`, `html2canvas.min.js` (C2), `boot.js`, `rce_loader.js`, `rce_loader_v5.js`, `rce_worker_18.{4,5,6,7}.js`, `post_exploit.js`, `native_bridge.js`, `nb_daemon.dylib`, trigger `evil_43818.heic`. ✅

> Full live-payload SHA256 hashes and raw fetch evidence are preserved in the originating case's evidence store (`/home/jeje/.panoptes` — tool-output, graph, IOC ledger), not reproduced here.

## Detection chain — what to look for

- **Network**: any client contacting `1688360.cc`, `eil786npto9u5bq.icu`, `5vobauua3ojpklx.icu`, `l7bcsnwc7nzr3uo.icu`, `moiubpagedocs.cc`, `blogforge.vu`, or `netvigator.cc`; any page that loaded `cdn.optimizely.com` bundles and then contacted those domains.
- **Web**: SRI mismatch on third-party tag bundles; unexpected `track.js`/`card.php`/`hit.php` requests; clipboard-watcher behavior (address swap) — the clipper signature.
- **Endpoint (iOS)**: `mediaplaybackd`/`SpringBoard` anomalies, `nb_daemon.dylib` presence, unexpected `native_bridge.js`, non-Apple signed processes.
- **Wallet**: inbound transactions to the listed addresses; clipboard addresses matching the wallet set.

## Postscript — the chain is still moving

At the time of writing (23:03Z, 16 Nov), the C2 was still serving `track.js`, the wallets were still consolidating, and the vendor had not yet published. This repository will be updated as the chain moves: a 24-hour scheduled re-check tracks C2 liveness, wallet sweeps, exchange deposits, and advisory status. The chain does not end because the bundles got cleaned — it ends when the money stops moving and the panel is found. It hasn't ended.

---

*Compiled by PANOPTES — passive-first investigation of the OPTIMJACK/Optimizely CDN supply-chain incident, 16 November 2026. Primary public source: amirulcyber.com OPTIMJACK write-up (16 Nov 2026). All live data retrieved as data, never executed. Confidence legend: ✅ confirmed · ⚠️ unconfirmed/hypothesis · ❌ ruled out.*
