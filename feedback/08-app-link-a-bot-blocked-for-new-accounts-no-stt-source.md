# App/docs: a new account cannot "Link a bot" at all — the app's key has no STT, the wallet has 0 STT, and there is no STT faucet — so the documented "simplest route" to bot gas dead-ends
Severity: normal
Area: app | docs

The developer page says under **Get test funds**:

> STT for gas. The simplest route is through the app: sign in at app.testnet.dreamdex.io, then Link a
> Bot with your key's address and use Send it … for gas or Top up.

and the Link a Bot guide says "The app signs this change with its own key, so there is no wallet
prompt" and "Leave Send it … for gas on to send the bot gas from this wallet". For a freshly created
account (email login, new Somnia Account wallet) neither works:

- the **Link a bot** form shows `Send it 5 STT for gas — From Wallet 1 (0.00 STT)` and, in red,
  `Can't add a bot yet: the app's key that sends this change has no STT.`; **Add bot** is disabled;
- the gas panel for the app's own key says `This app's key is nearly out of gas. 0.00 STT left` and
  `TOP IT UP — Send STT to this key`, i.e. it expects the user to source STT elsewhere;
- **Add funds** only shows the wallet's deposit address ("Deposit USDC, ETH & more"); there is no
  faucet in the app, `https://testnet.somnia.network` serves the Shannon network (50312), not Hideki
  (50383), and the RPC host has no `/faucet` endpoint (`POST /faucet` answers JSON-RPC
  `invalid request`).

Together with report 01 (paper USDso cannot be self-claimed) a developer who follows the developer
page on a new account ends up with no STT and no USDso, and the page never says that both must be
requested from the team.

## Steps to reproduce
1. Open https://app.testnet.dreamdex.io, **Connect** → **Log in with email or socials** → e-mail OTP
   (Privy). A new Somnia Account wallet is created (ours: `0x0e7E8DF6Eac2E14165b4fA4CB2BD4e445d0E3817`).
2. Open the wallet panel (top-right address) → click the wallet name → **Link a bot**.
3. Read the form before entering anything, or paste any bot address.
4. In the wallet panel, click the yellow "This app's key is nearly out of gas" banner.
5. Open the **Add funds** tab.

## Expected
Per the developer page, a new account can link a bot and send it gas from the app; or the page tells
the developer where testnet STT comes from (faucet URL or "ask in Discord").

## Actual
- Link a bot: `Perps — Perp margin: 0.00 USDso … Send it 5 STT for gas — From Wallet 1 (0.00 STT).
  Can't add a bot yet: the app's key that sends this change has no STT.` Add bot disabled
  (screenshot `evidence/link-a-bot-no-stt.png`).
- Gas panel: `Paid by this app's key — This app's key is nearly out of gas. 0.00 STT left … TOP IT UP:
  Send STT to this key (0x2277…49c6) … Three payers are tried in order: a relayer, then this app's
  key, then your login wallet.` The relayer evidently does not cover the bot-link change.
- Add funds: deposit address only, no faucet.

## Evidence
- Screenshot `evidence/link-a-bot-no-stt.png` (2026-10-19 ~11:08 UTC, account created minutes
  earlier).
- Panel texts quoted verbatim from the DOM (`document.querySelector('[role=dialog]').innerText`).
- Faucet probes: `curl -X POST https://api.hideki2.infra.testnet.somnia.network/faucet` →
  `{"jsonrpc":"2.0","id":"unknown","error":{"code":-32600,"message":"invalid request"}}`;
  `https://testnet.somnia.network/` mentions only Shannon.

## Suggested fix
Have the relayer pay for the operator-grant transaction (the app already signs it with its own key),
or seed each new app key / wallet with a little STT, or at minimum put the real STT route on the
developer page ("ask in #… on Discord" / a faucet link) next to the Link a Bot sentence.

## Environment
- App: app.testnet.dreamdex.io as served 2026-10-19 ~11:00 UTC (Chrome 155, Linux); network Tokyo/Hideki (50383)
- `@somnia-chain/markets-sdk` 0.32.1 (not involved)
- Wallet: app wallet `0x0e7E8DF6Eac2E14165b4fA4CB2BD4e445d0E3817`; intended bot key `0x2A11e7388D1D2F9690f2636846d44b478f61bF08`
