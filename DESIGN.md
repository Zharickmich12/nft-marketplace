# NFT Marketplace: design note

One seller lists one NFT for a SOL price. One buyer pays and receives the NFT in the same transaction. Devnet only.

## 1. Scope (what must exist)

| Deck requirement | How this project covers it |
|---|---|
| On-chain logic | Listing PDA, authority checks, three instructions |
| Token operations | NFT (supply 1, 0 decimals) moved with checked transfers, vault token account, close accounts |
| Composability | CPIs to the Token Program and System Program, signed by the PDA |
| User interface | React frontend with wallet connect on devnet |
| Deployed program | Program ID on devnet with real transactions |
| README | Summary, architecture, runbook, trade-offs |

Not in scope for the base version: royalties, auctions, offers, partial payments, collection-verification checks, mainnet.

## 2. Instructions

| Instruction | Signer | What happens |
|---|---|---|
| `list(price)` | seller | creates the listing, creates the vault, moves the NFT from the seller into the vault |
| `buy` | buyer | buyer pays the price in SOL to the seller, vault sends the NFT to the buyer, vault and listing close |
| `cancel` | seller only | vault returns the NFT to the seller, vault and listing close |

## 3. Accounts and PDAs

- **Listing** (PDA): seeds `["listing", nft_mint]`. One NFT has one active listing at a time. No extra `seed` field is needed, because an NFT exists once.
  - Stores: `seller`, `nft_mint`, `price` (lamports, u64), `bump`.
- **Vault**: token account for `nft_mint` whose **authority is the Listing PDA**. Never the seller.
- Seller and buyer token accounts: associated token accounts for `nft_mint`. The buyer's is created with `init_if_needed`.

## 4. Authority model

- The vault authority is the Listing PDA, so the NFT can only leave through program logic (same rule as the swap program).
- `list`: the seller signs the transfer into the vault (`CpiContext::new`).
- `buy` and `cancel`: the Listing PDA signs the transfer out of the vault (`CpiContext::new_with_signer`, seeds `["listing", nft_mint, bump]`).
- `buy` SOL payment: the buyer signs a System Program transfer straight to the seller (`CpiContext::new`).
- `cancel`: `has_one = seller` and `seller: Signer`.

## 5. Checks (each one needs a test)

**`list`**
- `price > 0`, otherwise `InvalidPrice`.
- The mint has 0 decimals and the seller's token account holds exactly 1.

**`buy`**
- The `seller` account passed in equals `listing.seller` (`has_one = seller`), otherwise the SOL could go to the wrong wallet.
- The `nft_mint` passed in equals `listing.nft_mint` (`has_one = nft_mint`).
- The buyer has enough SOL for `price` (the transfer fails with a clear error).
- A second `buy` fails because the listing is closed.

**`cancel`**
- A non-seller cannot cancel.
- `cancel` after `buy` fails, and `buy` after `cancel` fails.

**Closing:** vault (`close_account`) and listing (`close = seller`) both send their rent to the seller.

## 6. Tests

- Happy paths: list → buy; list → cancel.
- All failure cases above, each asserting that nothing changed (NFT still in the vault or with the seller, SOL balances unchanged apart from fees).
- Reconciliation: after `buy`, the buyer holds the NFT, the seller received exactly `price` plus the recovered rent, and the NFT total supply is still 1.
- Run in memory (litesvm) and once end to end against devnet with the real NFTs.

## 7. Frontend

- Connect wallet (devnet).
- **Marketplace:** all active listings (name, image, price, seller) read from the program accounts plus the NFT metadata JSON.
- **My NFTs:** NFTs in the connected wallet with a price field and a List button.
- **Buy** button on other people's listings; **Cancel** button on your own.
- Transaction status shown as pending / confirmed / error, with readable error messages (not raw objects).
- The list refreshes after confirmation.
- Demo wallets: seller and buyer must be different wallets. Use the browser wallet for one and a script or second wallet for the other. Never import a CLI keypair into the browser wallet.

## 8. Build order

1. This design note.
2. Program: state, `list`, `cancel`, then `buy`. Tests after each one.
3. Deploy to devnet and run an end-to-end script.
4. Frontend: wallet, listings, then buy, list and cancel.
5. Harden failure paths and messages.
6. README.

## 9. Deliverables checklist

- [ ] Program deployed on devnet, program ID in the README, at least one real list and one real buy transaction
- [ ] Tests for success and failure cases
- [ ] Frontend that runs the full flow
- [ ] README: summary, architecture (accounts, PDAs, instruction flow), runbook (setup, deploy, run UI, run tests), trade-offs and next steps

## 10. Trade-offs (fill in at the end)

- SOL price only, no token prices.
- No royalties or collection check.
- One active listing per NFT.
