# Idlescape — build plan

Derived from `docs/idlescape-1-idea-dump.md`. Mapped against the Lost City TS engine + RuneScript content that is already on disk.

## Where the code actually lives

| Path | What it is | Status |
|---|---|---|
| `Idlescape/Server/` | Lost City umbrella repo (`start.js` clones Engine-TS + Content + Client-TS/Java) | **bootstrap only** — no `content/`, no `engine/`, never run |
| `../2004scapeServer/` | the populated working copy: `content/`, `engine/`, `javaclient/`, plus `engine2/3/4` scratch copies | running dev tree, dirty git |
| `../2004scapeServer/content/` | all game logic: `scripts/*.rs2`, `*.if`, `*.loc/.npc/.obj`, `pack/` | the game |
| `../2004scapeServer/engine/src/` | TS server: login, hiscores, web, scripts VM | MIT |

Two things follow from that:

1. Idlescape should be a fork of `2004scapeServer`, not of `Idlescape/Server`. `Idlescape/Server` is a downloader.
2. Everything in this recording is **content work**, except Discord login, hiscores page, and the logout timer — those are engine work. That's the good news: ~80% of the pitch is RuneScript edits, testable in minutes.

## Decisions to make before writing code

1. **Webclient or Java client?** The recording literally says "a game you progress by leaving it open in your browser". `Client-TS` (web client) makes Discord OAuth a redirect + cookie, and lets the login screen be a normal web page. The Java client means OAuth has to bounce through the system browser and poll for a token. **Recommendation: Client-TS.** Note it is not in `2004scapeServer` (`git status` shows `webclient` deleted) — it needs cloning from `LostCityRS/Client-TS`.
2. **AFK vs offline progression.** "AFK and you get upgrades" can mean (a) the tab stays open and the character keeps skilling, or (b) you close the game and get XP for time offline. (a) is the recording; (b) is what actually makes idle games sticky and is maybe 100 lines (persist `logout_at`, grant XP on login). Don't build (b) silently — decide it.
3. **The "upgrades" are missing.** Nothing in this recording describes them. Logs→coins→iron axe is a linear ladder, not an idle loop. The retention loop needs: tool tiers, an upgrade shop with real costs, or prestige. This is the actual design hole.
4. **Licensing:** Content and Engine are MIT (© Lost City) — fine to fork and host, keep the notices.

## Phase 0 — get a running baseline (½ day)

- Copy `2004scapeServer` → `Idlescape/game/` (or fork the three repos properly under your own GitHub org, and point `start.js`'s `repoOrg` at it).
- `bun install` in `engine/`, `bash start.sh`, log in with a throwaway account, confirm you can chop a tree and sell a log. Everything after this is incremental and hot-reloadable.

## Phase 1 — spawn experience (1–2 days)

All in `content/scripts/tutorial/`:

- **Kill the tutorial, keep the map.** `tutorial/configs/tutorial.varp` + `tutorial/scripts/tutorial.rs2` drive the step chain (`tut_chatbox_steps.rs2`, `tutorialstep.rs2`). Set the tutorial varp to "complete" at login so no step ever fires; leave the island geometry alone.
- **Spawn point** → inside the Tutorial Island house, not Lumbridge. One line in the login/spawn config + `%tutorial` varp default.
- **Retext the two NPCs** in `tutorial/configs/tutorial.npc` (`welcome` tutor → "Welcome to Idlescape — a game you progress by leaving it open in your browser"). Delete the net-giver's dialogue entirely.
- **Bank booth upstairs in the spawn house.** Bank booths work anywhere a `bank` loc exists — add a loc to the map or spawn one with `loc_add`. No engine change.
- **Blocked fence.** `tut_doors_and_gates.rs2` already handles that gate; give it its own loc id (so the real gate isn't affected) and make `oploc1` print `"You cannot go here until you're past 300 total level."` and return. Text only, exactly as recorded.
- **Turn off tutorial music/overlay** (`interface_welcomescreen`), remove the "new user / existing user" screen.

## Phase 2 — the idle loop (2–3 days) ← the actual game

**Woodcutting / logs**
- `content/scripts/skill_woodcutting/scripts/woodcut.rs2`: `get_logs` depletes with `deplete_chance` and `respawnrate` from `trees.dbtable`. For Idlescape trees, add a `woodcutting_trees` row with `deplete_chance=0` and never call `loc_change(stump)` — infinite tree.
- **Logs stack:** `obj` config `stackable=yes` on logs (and the custom log tiers). Then the `inv_freespace(inv) < 1` guard in `woodcut.rs2` must be skipped for stackables, or you get "inventory too full" at 28 stacks — that guard appears twice (`attempt_cut_tree`, `cut_tree`).
- **No logout timer / no random events:** `content/scripts/macro events/` → empty the `macro_*` labels and drop the `afk_event` varp check in `woodcut.rs2`. Engine side, the socket idle/`NullClientSocket` timeouts: verify and disable in `engine/src/server/ClientSocket.ts`.
- Result: click tree once, come back in an hour to 60k logs. This is the money shot for the pitch — build it first.

**Sell X (and Buy X)**
- `content/scripts/shop/scripts/shop.rs2` has three buttons wired: `inv_button2/3/4` on `shop_template_side:inv` → sell 1/5/10. Add the "X" op (button 5) → `p_countdialog` → handle `resume_countdialog` → `@sell_item($obj, $amount)`. Copy the pattern verbatim from `content/scripts/interface_bank/scripts/bank_deposit.rs2` (it's ~30 lines). Same for the shop's own panel to get Buy X.
- Needs a client-side opname change so the inventory right-click menu shows "Sell X" (`Client-TS` shop interface config, or `Client.java` inventory options for the Java client).
- Guard: `inv_total(inv, $obj)` already clamps the amount, so typing 30000 is safe.

**Shop NPC**
- One `shop` config (`content/scripts/shop/configs/`) sold by the welcome tutor: bronze axe 0gp, iron axe 20k, tinderbox 100k, buys logs at 1gp.
- Caveat: shop prices come from the global `oc_cost` per obj × per-shop `shopbuy`/`shophaggle` multipliers. Setting `oc_cost` per item leaks into every other shop and into alchemy. With one shop in the world that's fine; the moment you add a second shop you need per-shop prices instead of global costs. Flag it now, don't discover it in month two.
- tinderbox `oc_cost=100000`: cheap, brutal, and it's what makes shared fires the social mechanic.

**Firemaking / shared fires**
- Burning already exists: `content/scripts/skill_firemaking/scripts/firemaking.rs2` (`firemaking_success` → `loc_add(coord, loc_2732, delay)` then `world_delay` → ashes).
- **Feeding a fire:** add `[oplocu, fire]` → if `last_useitem` is a log: `stat_advance(firemaking, oc_param($log, productexp))`, consume the log, `loc_change(loc_2732, $new_delay)`. `LOC_CHANGE` takes a duration and re-arms the lifecycle (`engine/src/engine/script/handlers/LocOps.ts:60`), so this is one opcode, not a new system. Loop it (`p_oplocu` re-trigger) so it keeps eating logs until you run out.
- **The ashes trap:** the current code spawns ashes via `world_delay($delay)` once, at light time. If the fire gets refuelled, that timer is stale and ashes appear under a burning fire. Replace with a watcher loop: `world_delay(2); if (loc_find(coord, loc_2732) = 0) { obj_addall(coord, ashes, 1, 100); return; }` — it follows every refuel for free.
- Sharing is already free (fires are world locs). The "interesting mechanic" = 100k tinderbox, so nobody buys one, everyone huddles around the one guy who did. Cheap to build, great for a trailer.

## Phase 3 — login (1–2 days)

- **Discord OAuth** in the Bun web server: `engine/src/web.ts` already runs a `Bun.serve` on `WEB_PORT` and a management server on `WEB_MANAGEMENT_PORT`. Add three routes:
  - `GET /auth/discord` → 302 to Discord with `state` nonce (cookie).
  - `GET /auth/discord/callback` → exchange code, fetch `id`+`username`, `INSERT ... ON CONFLICT` into an `account` row (`engine/src/db/`), mint a one-time session token, 302 back to the game.
  - `GET /auth/session` → returns the token for the client to send on the WS handshake.
- Server-side verify in `engine/src/server/login/LoginServer.ts` — it already bcrypt-compares passwords; add a branch: if the presented secret is a live one-time token, accept and consume it. Keep the password path for your own admin accounts.
- One button on the client welcome screen instead of username/password, plus the "new user / existing user" removal (Java client: `javaclient/src/main/java/deob/Client.java` ~line 4098, 11.6k-line deob; webclient: the login page markup).
- ~200 LoC total. Do this **last** — it's the easiest part and it blocks nothing.

## Phase 4 — competing (1–2 days)

- The engine already writes `hiscore` + `hiscore_large` on logout (`LoginServer.ts:45`) — total level and per-skill XP are being recorded today, for free.
- Add a public `/hiscores` route to `web.ts` (~100 LoC + HTML): overall total level, per-skill, top 100.
- Add the **"time played / logs chopped"** style counters if you want idle-game-flavoured boards; those need a wealth/event table or a new `stat` column on logout.

## Phase 5 — ship it (1–2 days)

- Linux VPS, Caddy in front (TLS for the webclient + OAuth callback), systemd units for login/world/web, `world.json` per world, nightly `db.sqlite` backup to R2/S3.
- Prometheus is already wired (`/prometheus` on the management port) — point Grafana at it and you get player count, ticks, logins for free. Fits the "observability first" rule: if a session mysteriously stops gaining XP, you want the tick/zone metrics before you want to guess.
- Discord: a bot that assigns a role to anyone with an account (same OAuth, `guilds.join`) — nice-to-have, and it makes the Discord login feel intentional.

## Order of work (why this order)

1. Baseline running.
2. Infinite trees + stacking logs + no logout timer → **this is the game**. If this isn't fun to leave open for 20 minutes, nothing downstream matters.
3. Sell X + shop + shared fires → the economy and the social hook.
4. Spawn polish (no tutorial, bank, blocked fence).
5. Hiscores page.
6. Discord login.
7. Deploy.

## Open questions

1. Webclient or Java client? (recommend webclient)
2. AFK-with-tab-open, or XP while logged out?
3. What are the "upgrades"? Tool tiers, an upgrade shop, prestige? This is the missing half of the pitch.
4. Do you want idle-time caps / soft anti-bot rules, or is "leave it running" the whole point?
5. Is the fence a real 300-total-level gate at launch or genuinely text-only?

## Rough totals

| Phase | Work |
|---|---|
| 0 | ½ day |
| 1 spawn | 1–2 days |
| 2 idle loop | 2–3 days |
| 3 login | 1–2 days |
| 4 hiscores | 1–2 days |
| 5 deploy | 1–2 days |

≈ 7–10 focused days to something a stranger can log into, chop forever, feed a fire, and see their name on a leaderboard. The parts that actually decide whether Idlescape works are #2 and the missing upgrade loop — spend the time there, not on OAuth.
