# Running Idlescape

## 1. Start the server

```sh
cd engine && bun run src/app.ts
```

Requires [bun](https://bun.sh). The first boot packs `../content` into `data/pack` and takes ~15s; later boots are ~5s.

Ports: `8888` web, `8898` management, `43594` game, `43500` login, `45099` friend, `43501` logger.

## 2. Discord login setup

Discord is the only login method, and it is disabled until you create an application:

1. https://discord.com/developers/applications → New Application.
2. OAuth2 → Redirects → add `http://localhost:8888/discord/callback`, then **Save Changes** at the bottom of the page. Adding the row is not enough — unsaved rows are silently dropped.
3. OAuth2 → copy the Client ID and Client Secret.
4. Edit `engine/data/config/world.json`:

```json
"discord": {
    "enabled": true,
    "clientId": "<client id>",
    "clientSecret": "<client secret>",
    "redirectUrl": "http://localhost:8888/discord/callback"
}
```

5. Restart the server.

Until step 4 is done, `GET /discord` answers `503 Discord login is not configured.` and the login button does nothing.

**The redirect URI must match character for character.** Discord compares the `redirect_uri` we send against that list and refuses with `Invalid OAuth2 redirect_uri` on its own page if they differ (you never reach our server, so nothing is logged). At boot the server prints the URI it will send, so compare against that line:

```
INFO  Discord login: enabled, redirect URI http://localhost:8888/discord/callback
```

## 3. Play

Open http://localhost:8888/rs2.cgi. Without a session cookie that URL redirects straight to Discord, so the client is never handed out to someone who cannot log in. After you authorise, the callback sets a session cookie and the client logs in by itself.

The title screen has one **Login** button. It does not navigate anywhere: the page already has a session, so the button just reconnects with the token from `/discord/session`. If the session has gone stale, the client sends you to `/discord` instead.

The first sign-in creates an account named after your Discord display name and drops it in the starting hall.

### Authentication is only real when the login server is on

`environment.login.enabled` (the `login` block in world.json) **must be `true`**. With it off, the world's login thread skips authentication entirely: it accepts any 1-20 character username/password and serves `data/players/<profile>/<username>.sav` directly. Nothing verifies the Discord token, no `account_login` row is written, and a stray client could connect to the game port as any name. The easyStartup worker still binds port 43500 either way, so the only symptom is that `Login:` lines never appear in the log.

With it on, every login goes through the login server, which is what makes the Discord token single-use and writes `account_login`/hiscores.

### The login key

The engine owns the RSA keypair in `engine/data/config/private.pem` and serves the public half at `/publickey`; the web client fetches it at startup rather than carrying a build-time copy. To rotate it:

```sh
cd engine && bun run tools/server/rsa.ts   # rewrites data/config/{private,public}.pem
# restart the server
```

No client rebuild, no constants to paste. The three checks the world makes on a login are all logged now (`Login rejected (client out of date)`, `(cache out of date)`, `(rsa mismatch)`), so a rejected login says which one it was instead of only telling the player to reload.

There is also a fixed engine bug in `src/io/Packet.ts` worth knowing about: node-forge's jsbn reads a byte array as **two's complement**, so `new BigInteger(bytes)` produced a *negative* number whenever the top bit was set. Roughly half of all ciphertexts are like that, so `rsadec` decrypted half of all logins to garbage and the world answered "RuneScape has been updated!". Both `rsaenc` and `rsadec` now prepend a zero byte to force an unsigned read.

## 4. Manual test checklist

- Login with Discord lands you in the starting hall (not Lumbridge) holding a bronze axe.
- Opening /rs2.cgi while signed out sends you to Discord; with a session it loads the game.
- Left-click a tree once: the character keeps chopping on its own and logs stack in one slot.
- The tree never turns into a stump and chopping never stops at 28 logs.
- Right-click the Idle Guide → **Trade**: the shop offers Buy 1/5/10/X and Sell 1/5/10/X.
- Sell logs for 1gp each; buy a small fishing net (5gp), tinderbox (100gp) and iron axe (56gp).
- Use the net on the fishing spot next to the pond to catch raw shrimp; use one on a fire to cook it.
- Light a fire with a tinderbox, then use logs on the fire: *"The fire burns brighter."* and it stays lit.
- When a fire finally goes out it leaves ashes.
- Click the fence: *"You cannot go here until you're past 300 total level. Your total level is N."* It opens for real at 300.
- Leave the game alone for 5+ minutes: no logout, no random event, chopping continues.
- Reload the page: you are logged back in where you left off.

## 5. Hosting on labs.lostcity.rs

The callback path does not change, only the origin. Add a second Redirects entry to the **same** Discord app and keep the localhost one, so local development still works:

```
http://localhost:8888/discord/callback
https://labs.lostcity.rs/discord/callback
```

Discord expects HTTPS for anything that is not localhost, so if the portal refuses the `http://` localhost row once you have a real domain in there, that is why.

On the server, use environment variables instead of world.json, so the secret is not sitting in a file that the setup API hands out:

```sh
DISCORD_ENABLED=true
DISCORD_CLIENT_ID=1567923625015644190
DISCORD_CLIENT_SECRET=...
DISCORD_REDIRECT_URL=https://labs.lostcity.rs/discord/callback
```

These override the `discord` block in world.json (`migrateFromLegacyEnv`), so a systemd `EnvironmentFile` or Docker env is enough — no code change per host.

What already works once you switch to HTTPS:

- The session cookie flips to `Secure` on its own, because `DiscordAuth.setCookie` keys off whether `redirectUrl` starts with `https:`.
- No CORS or proxy configuration is needed: the client is served from the same origin, and the cookie is `SameSite=Lax`, which is fine for the top-level redirect back from Discord. The redirect URI is always taken from config, never from the request, so TLS termination in Caddy in front of the app does not change what we send to Discord.

One thing the code does not protect you from: the management port (`8898`) serves `/setup/config`, which dumps the whole world config — including `discord.clientSecret` if it is in world.json. Bind that port to localhost or firewall it. Using the env vars above keeps the secret out of that response entirely.

## 6. Rebuilding

```sh
cd engine && bun run build          # re-pack ../content into engine/data/pack
cd webclient && bun run build       # bundle the web client into webclient/out
cp webclient/out/client.js webclient/out/client.js.map \
   webclient/out/ondemandworker.js webclient/out/ondemandworker.js.map \
   webclient/out/tinymidipcm.wasm engine/public/client/
```

`engine/public/client` is what the server serves, so if the web client is not rebuilt and copied, play changes will not show up. The build no longer needs `LOGIN_RSAN`/`LOGIN_RSAE`; those were a build-time copy of the engine's key.

Idlescape content deliberately differs from upstream, so the packer's checksum verification (`build.verify`) defaults to `false` in this fork. Turning it back on makes `bun run build` fail with `checksum mismatch!` on any edited config.
