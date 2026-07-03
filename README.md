# MMM-NowPlayingOnSpotify (v2.0.0, 2026-07-03)

A module for the [MagicMirror²](https://github.com/MagicMirrorOrg/MagicMirror) project displaying the song currently playing on Spotify.

This is a maintained fork of [raywo/MMM-NowPlayingOnSpotify](https://github.com/raywo/MMM-NowPlayingOnSpotify) (MIT). It adds handling for Spotify's refresh-token expiry (effective 20 July 2026), a self-managed token store, and an integrated re-authorization flow — so a dead token no longer silently blanks the display.

## What it does

After you set up a Spotify app and authorize it once, the module shows the track you are currently listening to, on which device, and optionally the album cover.

Since 20 July 2026 Spotify expires user refresh tokens after **6 months**. When that happens this module:

- detects the `invalid_grant` error, stops hammering the API, and shows a clear **red re-authorization banner** on the mirror (instead of pretending nothing is playing);
- warns **proactively** (about two weeks before the hard expiry) with a smaller banner above the cover art, showing the same re-authorization URL so you can renew early — the music keeps playing;
- lets you re-authorize from any computer via a short SSH tunnel — no config editing, no token copy-paste, and the mirror recovers automatically without a restart.

## Preconditions

- A MagicMirror² instance (Node.js **>= 18**, for native `fetch`)
- A Spotify account

This module has **no runtime dependencies**.

## Installing

```bash
cd ~/MagicMirror/modules
git clone https://github.com/rkorell/MMM-NowPlayingOnSpotify.git
cd MMM-NowPlayingOnSpotify
npm install
```

## Step 1 – Create a Spotify app

Create an app in the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard). Note the **Client ID** and **Client Secret**.

Add this exact **Redirect URI**:

```
http://127.0.0.1:8888/callback
```

> ⚠️ **This is effectively the only value that works — take it literally.**
>
> Spotify accepts `http` redirect URIs *only* for the loopback address `127.0.0.1`. Watch out for the trap: a LAN IP or hostname over `http` may be **accepted in the dashboard yet rejected at the actual redirect** as *insecure* — and `localhost` is not allowed at all (it must be the numeric `127.0.0.1`).
>
> In practice the only part you might ever change is the **port** (`8888`), and then only consistently in both the redirect URI and `config.js`. The single real alternative would be an `https://` URI with a valid certificate — which on a headless mirror means running a reverse proxy plus certificate management, wildly disproportionate for a twice-a-year login. So for all practical purposes: use `http://127.0.0.1:8888/callback`.

## Step 2 – Configure the module

```javascript
{
  module: "MMM-NowPlayingOnSpotify",
  position: "top_right",

  config: {
    showCoverArt: true,
    clientID: "<YOUR_CLIENT_ID>",
    clientSecret: "<YOUR_CLIENT_SECRET>",
    redirectURI: "http://127.0.0.1:8888/callback"
  }
}
```

You no longer paste tokens into `config.js`. Tokens are obtained in Step 3 and stored by the module itself in `tokens.json` (git-ignored — it holds secrets, never commit it).

**Migration from v1.x:** if your `config.js` still contains `accessToken` and `refreshToken`, they are picked up once and moved into the token store automatically. After the first successful run you **should remove** `accessToken` and `refreshToken` from `config.js` — only `clientID`, `clientSecret` and `redirectURI` need to stay.

## Step 3 – Authorize (once) and re-authorize (every 6 months)

Both use the same flow. Because the redirect URI is a loopback address, the browser doing the login must reach the mirror's `127.0.0.1:8888` — bridge it from another computer with a one-line SSH tunnel:

1. When authorization is needed, the mirror shows a red banner.
2. On your computer, open an SSH tunnel to the mirror and keep it open:
   ```
   ssh -L 8888:127.0.0.1:8888 pi@<mirror-ip>
   ```
3. In a browser on that computer, open `http://127.0.0.1:8888` and click **Log in with Spotify**.
4. Spotify redirects back through the tunnel, the token is stored, and the mirror recovers on its own — no restart, no file editing.

Alternatively, run a browser directly on the mirror's own desktop and open `http://127.0.0.1:8888` there.

That's it — twice a year, one login.

## How the tokens work (and why `refreshToken` is the important one)

Spotify uses two tokens, and their names are misleading:

- **`accessToken`** — short-lived (about **1 hour**). It is the token actually sent with each API request, but it is disposable: the module treats any stored access token as already expired and mints a fresh one on startup, then roughly every hour. You never supply a lasting access token — its value is throwaway state.
- **`refreshToken`** — long-lived (now **6 months**). This is the credential that actually keeps the module working: it is exchanged for new access tokens over and over. When it expires, everything stops until you re-authorize — which is what this whole module is about.

So despite the naming — *access* sounds primary, *refresh* sounds like a mere helper — the **`refreshToken` is the one that matters**. Both are obtained during authorization and managed for you in `tokens.json`; you never edit them by hand. This is also why the `accessToken` that used to sit in a v1.x `config.js` was effectively irrelevant: it was replaced by a refresh on every start, so it was safe to leave stale — and safe to delete now.

## Configuration options

| Option | Description |
|--------|-------------|
| `clientID` | **REQUIRED** string – the Client ID of your Spotify app. |
| `clientSecret` | **REQUIRED** string – the Client Secret of your Spotify app. |
| `redirectURI` | **REQUIRED** string – the redirect URI whitelisted in your Spotify app, and the port the local auth server listens on. In practice this is fixed to `http://127.0.0.1:8888/callback` (loopback only — see the callout in Step 1); the **port** is the only part you would realistically change. |
| `showCoverArt` | Optional boolean – show the album cover. Default `true`. |
| `updatesEvery` | Optional integer – display update interval in seconds. Default `1`. Lower is more responsive; higher relieves the Raspberry Pi. |

## Updating

```bash
cd ~/MagicMirror/modules/MMM-NowPlayingOnSpotify
rm -rf node_modules
git pull
npm install
```

## Localization

The re-authorization and warning texts follow MagicMirror's `language` setting. English (`en`) and German (`de`) are included in `translations/`; add another locale by dropping a matching JSON file there.

## Credits

- Original module: [raywo](https://github.com/raywo) — [MMM-NowPlayingOnSpotify](https://github.com/raywo/MMM-NowPlayingOnSpotify) (MIT).
- Fork maintained by [Dr. Ralf Korell](https://github.com/rkorell).
- [Michael Teeuw](https://github.com/MichMich) for the MagicMirror project.
