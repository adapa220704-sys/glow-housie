# Glow Housie — Netlify ready

## Publish by drag and drop

1. Sign in to your Netlify account, then open **https://app.netlify.com/drop**.
2. Drag the complete **netlify-ready-game.zip** into the Drop zone. If your browser asks for a folder, unzip it and drag the complete project folder containing `netlify.toml` and `package.json`.
3. Wait for Netlify's automatic Vite build and Function deployment. Open the resulting site URL and share it with friends.

No GitHub, terminal commands, environment-variable configuration, API keys or database setup are required. **Stay signed in and upload the entire source project.** Netlify's signed-in Drop flow detects/builds source projects. Dropping only `dist/` or dropping while signed out publishes static files without the required backend build.

Netlify config is already at the project root. It publishes `dist/` and discovers the single standard Function at `netlify/functions/game.ts`. Blobs storage and its runtime credentials are supplied automatically by Netlify. If your Netlify team has private-by-default enabled, use its normal site visibility setting to make the game accessible to friends.

## Design and gameplay preserved

The existing React arcade HUD, original avatar art, tactile paper/metal ticket skins, ink-slash/splatter effects, animated caller, history ticker, podium, shop, daily quests and leagues are retained. Private rooms support 2–8 players. The host gets a valid unique 3×9 ticket and also plays. Players manually mark any ticket number; called numbers never highlight or auto-mark their tickets. Claims remain server-verified. Early Five gives 10 points, each row 20, Full House 50. Full House ends a round. Highest current-round score wins; ties share the lead. Rematches reset current-round scores and generate new tickets.

Online matchmaking and solo play against three server-controlled bots are included. Auto-Call uses persisted server deadlines; any player can pause/resume the room. Private/bot hosts can select intervals or call manually. Direct invite links use `/?room=ABCD` and open a nickname-only join prompt. Persistent guest currency, goals, owned/equipped cosmetics and ranking points retain the existing rules. Guest profiles belong to this browser's cookie. Optional Email Login links guest progress and restores the same coins and unlocks on another device; guests who clear cookies lose access unless they linked an email profile.

## Email Login and Daily Rewards

Use **Sign in with Email** on the main menu, then choose **Create Profile** for the first visit. Existing guest coins, avatars, skins and stats are kept. Return with the same email and password on any device to restore that player profile. Guest play stays available without registration.

Email addresses are trimmed, normalized and lowercased. Their SHA-256 hash is the actual blob key in the automatically created `glow-housie-email-profiles` store. Each account JSON contains its profile, coin balance, unlocked avatar IDs and `last_daily_reward_timestamp`. Passwords require 8–128 characters and are stored only as salted PBKDF2-SHA256 hashes (210,000 iterations); no plaintext passwords are stored or returned. Sign-in uses rotating HttpOnly/SameSite cookies, secure on HTTPS, with 30-day server-checked sessions. Five incorrect attempts lock the account for 15 minutes.

This is lightweight, self-contained game authentication: email ownership is not verified, no email is sent, and there is no email-based password reset. Keep the chosen password. No external authentication provider, API key or manual database setup is involved.

A glowing **Claim Daily Reward** button appears in Player HQ and the room lobby. Signed-in players receive a cryptographically random **50–200 coins** on their first claim and again after a rolling **24 hours**. The timestamp is a server-clock millisecond value; submitted client timestamps/balances are ignored. The button displays a server-calibrated HH:MM:SS countdown while recharging. This daily bonus is separate from the existing UTC-reset gameplay quests.

The coin grant and timestamp update share the existing strong-consistency conditional transaction. Two devices claiming the same account simultaneously can award only once. Confirmed profile revisions are then saved to the email-hash blob using conditional writes. Delayed requests cannot overwrite newer saved profiles; if a mirror write is interrupted, the canonical wallet remains committed and the next authenticated request repairs the account snapshot.

## Serverless architecture

All frontend requests go **directly** to `/.netlify/functions/game`. The JSON `channel` selects game, profile or matchmaking operations inside that one Function. There is no custom `config.path`, game API redirect, websocket, SSE server, Cloudflare Worker, Durable Object, D1 database or long-running backend process.

Clients request short snapshots about once per second. Calls, bot reactions and online start/end deadlines advance against the server clock when a request arrives. Disconnected rooms catch up on the next request while within the 24-hour room lifetime. Without requests, there is no continuously running timer. Claim validation, host permissions, random numbers, rewards and purchases remain authoritative on the server.

The site-wide `glow-housie-v1` Blobs store uses **strong consistency** for every read. One versioned arcade snapshot keeps rooms, guest profiles, the queue and reward receipts in the same atomic write. Initial creation uses `onlyIfNew`; updates use `onlyIfMatch` with the read ETag. Conflicts discard the stale snapshot and retry the full action. Missing ETags fail closed: they never degrade to an unconditional write. Wallet grants and claims therefore cannot split across records. This single-snapshot design suits a small party-game deployment; it serializes concurrent writes across its rooms.

Guest cookies are HttpOnly/SameSite=Lax and Secure on HTTPS; only their SHA-256 digest is stored. Room bearer tokens allow the same-device reconnect. Public room views omit other players' tokens, tickets and marks. Both site-wide stores persist across deploys; they are initialized automatically by the same standard Function. Earned rewards are applied before expired rooms are cleaned up.

## Included files

- `netlify.toml`, `package.json`, `package-lock.json`, `index.html`, `vite.config.ts`, `tsconfig.json`: automatic deployment/build configuration.
- `netlify/functions/game.ts`: the only discoverable function source file.
- `server/service.ts`, `server/accounts.ts`: request routing, atomic wallet transactions, email/password hashing and conditional profile saving, outside the functions directory.
- `src/`: complete frontend, shared rules, original game/economy engine and bundled fonts.
- `public/`: static assets.
- `dist/`: verified prebuilt production frontend, provided for inspection; upload the whole project.
- `release/functions/game.zip`: independently packaged standard Netlify API v2 function, with all dependencies bundled and no `node_modules`.
- `release/function-source/game.mjs`: bundled function code from the exact source.
- `tests/`, `scripts/`: reproducible validation and packaging code. Netlify runs the configured build automatically; these require no user-run commands.
- `verification/results.json`: packaged-function validation results.

## Verification completed

The tests run three complete rounds through API requests with two players in the same private room, generating real tickets and drawing all 90 numbers in every round. They check host permissions, unique tickets, private snapshots, manual marking, pause/resume, claim scoring, rematches, state synchronization, final winner, coins, daily rewards and duplicate-grant prevention.

| Round | Pavan | Teja | Winner |
|---|---:|---:|---|
| 1 | 80 | 40 | Pavan |
| 2 | 20 | 100 | Teja |
| 3 | 100 | 20 | Pavan |

**Final-round winner: Pavan, 100–20.** Aggregate test totals are 200–160, recorded for verification only; no tournament scoring change was introduced. The final wallet check is 570 coins for Pavan (including one collected daily reward) and 460 for Teja.

The same three-round scenario is repeated by importing the code extracted from the packaged function ZIP and using the real Netlify Blobs SDK against Netlify's local `BlobsServer`. That emulator omits GET ETags in SDK 11.1.4, so the test harness mirrors the server's confirmed PUT ETags onto GET responses to reproduce the documented production read contract. Production code uses the actual platform headers and contains no emulator, fixture store or test credentials.

Email/daily tests verify profile restoration on a second device, guest migration, hashed email keys, salted password storage, failed-login lockouts, duplicate email handling, logout, concurrent daily claims, client timestamp tampering and the exact 24-hour boundary. The real-SDK packaged-function integration repeats email login and both daily eligibility checks.

Other tests cover 10,000 valid tickets, concurrent conditional writes, competing claims, purchases, matchmaking, skin/identity spoof prevention, bot timing and missing-ETag rejection. Ticket-rendering tests confirm all five skins remain identical before/after calls. The production frontend and the standard API v2 packaged function are built and verified. No live Netlify deployment or visual browser QA was performed here.

## Netlify references

- Signed-in source drag/drop: https://docs.netlify.com/start/quickstarts/netlify-drop-quickstart/
- Functions configuration: https://docs.netlify.com/build/functions/configuration/
- Strong consistency, conditional writes and automatic Blobs context: https://docs.netlify.com/build/data-and-storage/netlify-blobs/

## Mobile ticket fit
The complete 3×9 ticket uses nine flexible `minmax(0, 1fr)` columns, square cells, 12/14/16px responsive numbers, 2px mobile gaps and compact 8px outer horizontal padding. Ticket cards and wrappers are full width with a 28rem cap, centered and overflow-hidden. There is no fixed minimum grid width, swipe prompt or pink hover ring. The no-assistance tests still cover every skin, with width-boundary checks from 320 to 768px.

## Crew chat and reactions
Use the floating chat button to open the room drawer. Messages are plain text (1–200 characters), with the last 100 retained for the room's 24-hour lifetime. Senders come from authenticated room seats. A compact emoji picker sends six shared reactions; fresh events float along the screen edges, honor reduced-motion preferences, and do not replay when joining. All updates use the existing one-second Function polling and atomic, strongly consistent Blobs writes. Chat and reaction buttons float outside the ticket grid; all nine ticket columns still fit without swiping. No WebSockets or extra setup are needed.
