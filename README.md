# Mint Precheck — Farcaster Mini App

A read-only mint/claim precheck tool for **BNB Smart Chain (56)** and **Base (8453)**,
built as a static site that also runs as a [Farcaster Mini App](https://miniapps.farcaster.xyz).

Paste a contract address (or any link/text containing one) and it reads the chain live,
in the visitor's own browser, and returns a verdict: is it really free, still open,
how much supply is left, what the per-wallet limit is, a 0-value simulation of the mint,
and basic risk flags. Adapted from the Daxin Check web app (the engine behind
@daxin_check_bot); the BSC probe logic is inherited from it.

**Live target URL (after launch):** https://georgeonweb3.github.io/mint-precheck-miniapp/

## How the checks work

- All reads are plain JSON-RPC `eth_call` / `eth_getCode` / `eth_getStorageAt` /
  `eth_gasPrice` against public endpoints, with failover across 4 endpoints per chain
  (BSC: Binance dataseeds + PublicNode; Base: mainnet.base.org, PublicNode, dRPC, 1RPC).
  Nothing is cached and there is no backend.
- Probe tables detect common view functions by selector: name/symbol/supply/owner/paused,
  13 price functions, 12 open/active functions, per-wallet limit functions,
  already-minted counters and whitelist checks (against one sample wallet), plus
  ERC-721/1155 interface detection.
- Mint/claim functions found in the contract bytecode are simulated with `eth_call`
  at 0 value from the sample wallet — a dry run only. A revert during simulation is
  information, not a transaction.
- Risk flags: EIP-1967 proxy (upgradeable), non-zero owner, paused state, and
  `setApprovalForAll` exposed outside a standard NFT interface.
- **Read-only, by construction:** the page contains no signing code, no private-key or
  seed handling, no `approve`/`setApprovalForAll` calls, and never connects a wallet for
  checks. The one exception is the optional in-Farcaster tip button, which only *opens*
  the Farcaster client's own send screen (`sdk.actions.sendToken`, 1 USDC on Base
  prefilled) — the user confirms there; this page cannot send anything itself.

## Farcaster integration

- `index.html` carries the `fc:miniapp` (and legacy `fc:frame`) embed meta tags, so the
  URL renders as a launchable card in feeds.
- The Mini App SDK (`@farcaster/miniapp-sdk` via esm.sh) loads as a module: it calls
  `sdk.actions.ready()`, shows the viewer's @username/FID from `sdk.context` when
  present, and enables the SDK tip button. If the SDK fails to load or there is no
  Farcaster context, the page runs standalone with zero breakage.
- `.well-known/farcaster.json` is the app manifest (spec checked 2026-11-10 against
  miniapps.farcaster.xyz). `requiredChains` lists only `eip155:8453` (Base) because
  that is the only chain the *wallet* ever touches (tips); BSC/Base reads are public
  RPC and need no wallet chain support.
- `.nojekyll` must stay in the repo root — without it, GitHub Pages' Jekyll build
  drops the `.well-known` directory and the manifest 404s.

## ⚠️ Launch steps that still need George (not done by the builder)

1. **Create and push the GitHub repo.** Create `georgeonweb3/mint-precheck-miniapp`,
   push this folder's contents to its root (including `.well-known/` and `.nojekyll`),
   then in repo Settings → Pages, publish from the `main` branch root. Confirm
   https://georgeonweb3.github.io/mint-precheck-miniapp/ loads and
   https://georgeonweb3.github.io/mint-precheck-miniapp/.well-known/farcaster.json
   returns the manifest JSON. If you host under a different domain/path instead,
   update every absolute URL in `index.html` (embed meta) and `farcaster.json` to match —
   the manifest domain must exactly match the hosting domain.
2. **Sign the manifest account association.** The `accountAssociation` object in
   `.well-known/farcaster.json` is intentionally EMPTY (`header`/`payload`/`signature`
   are blank placeholders — never accept a pre-filled one from anyone). Generate the
   real one with your own Farcaster account: Farcaster app → Settings → Developer
   Tools (enable Developer Mode) → Mini App Manifest Tool at
   farcaster.xyz/~/developers/mini-apps/manifest, enter the domain
   `georgeonweb3.github.io`, sign, paste the three values into the file, and push again.
   Until this is done the app still launches, but it will not show you as its
   verified author. (Farcaster Hosted Manifests exist as an alternative, but they
   need a server-side redirect, which GitHub Pages cannot do — the static file is
   the right route here.)
3. **Smoke-test inside Farcaster.** Open the live URL from the Farcaster app's
   developer preview / embed tool: confirm the splash clears (SDK `ready()`),
   your @username/FID line appears, a Base precheck runs (try the Pixel Foxes
   example chip), and the **Tip 1 USDC** button opens the client's send screen
   prefilled to your address. The tip call shape follows the current SDK types
   (CAIP-19 token string + `recipientAddress`) and was not testable from a
   desktop browser build environment — if the send screen does not open, that
   button is the only thing affected; the address copy row always works.
4. **Submit for indexing/discovery.** After steps 1–3: cast the app URL from
   @georgeagbe (the embed card is the discovery surface), and submit the app in
   the Farcaster app's Mini Apps section / developer tools if a submission entry
   is offered there. Discovery is driven by the signed manifest + casts; there is
   no separate app-store review queue in the current docs.
5. **Optional:** replace `icon.png` / `splash.png` / `embed.png` with designed art
   (spec: icon 1024×1024 PNG no alpha; splash 200×200; embed/OG 1200×630–1200×800,
   3:2 for the feed embed). The shipped images are simple generated placeholders
   that meet the format rules.

## Spec / ownership note (verified 2026-11-10)

Farcaster changed owners again on **2026-10-15** — Network
School (Balaji Srinivasan) acquired Farcaster from Neynar (which had taken over from
Merkle Manufactory in January 2026), with a rebrand to "Network State Protocol"
announced. The mini apps docs at miniapps.farcaster.xyz are still live and were
re-checked for this build: the manifest format (`miniapp` object, `version: "1"`,
`accountAssociation` signed via the Manifest Tool) and the embed/SDK flow are
**unchanged** from the pre-acquisition spec. One addition since older guides:
Farcaster Hosted Manifests (see step 2 for why the static file is used instead).

## Files

- `index.html` — the whole app (engine, UI, i18n EN/中文， Farcaster SDK wiring)
- `.well-known/farcaster.json` — manifest (accountAssociation pending George's signature)
- `icon.png` (1024×1024), `splash.png` (200×200), `embed.png` (1200×800) — placeholder art
- `.nojekyll` — required so GitHub Pages serves `.well-known/`
- `README.md` — this file
