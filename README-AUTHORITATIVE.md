# Winzo authoritative-server test setup (SQLite)

## Files
- `index.html` — player app; existing game UI/animations preserved.
- `admin.html` — separate admin approval panel.
- `server.js` — authoritative Node/WebSocket server and SQLite persistence layer.
- `package.json` — server dependencies.
- `.devcontainer/devcontainer.json` — Codespaces setup.
- `.github/workflows/pages.yml` — GitHub Pages deployment.
- `.env.example` — environment variable template.
- `.gitignore` — keeps secrets and the SQLite database out of Git.

## Architecture

```text
GitHub Pages
   │ HTTPS / WSS
   ▼
Node.js authoritative server
   │
   ▼
SQLite (`winzo.sqlite`)
```

The browser never connects directly to SQLite. All game state, wallet operations, deposits, withdrawals, referrals, notifications, and admin decisions go through the authoritative WebSocket server.

## Codespaces test

1. Open the repository in GitHub Codespaces.
2. Set Codespaces port `8080` to **Public**.
3. Create `.env` from `.env.example` and set:
   - `BOT_TOKEN` — your Telegram bot token. For development, leaving it empty allows the DEV username login path.
   - `ADMIN_KEY` — a private, long random value used only by `admin.html`.
   - `ALLOWED_ORIGIN=*` for initial testing.
   - `SQLITE_DB_PATH=./winzo.sqlite`.
4. Run:

```bash
npm install
npm start
```

5. Test the server:

```bash
curl http://localhost:8080/health
```

6. The forwarded port URL is the server host. The WebSocket URL uses the same host with `wss://`.
7. Open the GitHub Pages player app with:

```text
?server=wss://YOUR-CODESPACE-8080.app.github.dev
```

8. Open `admin.html` with the same `?server=` parameter and enter the `ADMIN_KEY`.

## Money flow

- Deposit: player sends a request to the server; SQLite records it as `pending`.
- Admin approval: server credits the player's `play_wallet` exactly once.
- Withdrawal: server reserves the amount in `pending_withdrawal` immediately.
- Withdrawal approval: server moves the reserved amount out of `main_wallet`.
- Withdrawal rejection: server releases the reservation.
- The player browser cannot approve transactions.
- Deposit screenshots are compressed in the browser and stored with the pending transaction in SQLite for testing.

## Realtime game flow

- WebSocket is the single authoritative realtime channel.
- SQLite persists users, wallets, transactions, notifications, and the current room snapshots.
- A temporary WebSocket close does not automatically pull a player back into the active game.
- A disconnected human's active picks are refunded when the player reconnects; if the round finishes first, the disconnected picks are refunded during round reset.
- The refund is sent as a wallet update; no refund popup is required.
- Game rules remain 15/30 Birr rooms, numbers 1–200, up to 3 picks, 30-second selection, 2-second spinning, 5-second results, 80% payout, and 5% referral commission.
