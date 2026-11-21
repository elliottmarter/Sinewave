# Sinewave

> [!WARNING]
> **This app was written by AI (Claude) for my own personal use.**
> It has not been professionally reviewed or audited. You're free to use, copy and modify it,
> but **entirely at your own risk**. It comes with no warranty and no support, and I accept no
> responsibility for any data loss, security issues or other problems that result from using it.
> Released under the [MIT License](LICENSE).

**Everyone, on the same wavelength.**

Open source digital signage software.

Sinewave aims to make it simple to run digital signage on ordinary hardware: build content, schedule it, and push it to screens without vendor lock-in or licence fees.

![Screens dashboard](docs/images/dashboard-screens-online.png)

## Goals

- **Self-hostable**: run the whole stack on your own server or a single machine.
- **Cheap players**: display content on low-cost devices such as a Raspberry Pi, a mini PC, or any modern browser.
- **Content and scheduling**: manage images, video, web pages and playlists, and schedule what plays where and when.
- **Remote management**: see and control every screen from one dashboard.

## What works today

- **Dashboard** to upload images and videos, add web pages, and build playlists (ordering and per-item durations).
- **Slides**: make a slide right in the dashboard. Pick a template (announcement, welcome, event, menu or list, big message), pick one of the built-in backgrounds or use your own picture, type your text and add a logo or photo if you like. A live preview shows exactly what screens will show, and edits reach screens straight away. Slides are drawn by the player, so they are sharp at any resolution, fit portrait screens too, and work offline.
- **Web page previews**: when you add a web page, the server takes a small screenshot of it with headless Chromium (bundled in the Docker image) and shows it on the media card, so you can tell pages apart at a glance. If a screenshot isn't possible, it uses the picture the page offers for link previews. **Refresh preview** on the card takes a new one.
- **News**: add headlines from popular feeds (BBC, Sky, The Guardian and more) or any RSS or Atom URL, shown as a full-screen slide that rotates through the stories. Pick a layout (feature photo, split, or a headlines list), light or dark theme, accent colour, number of stories and time per story. Feeds are fetched by the server every 15 minutes, and screens keep the latest stories and photos for offline playback.
- **Screen pairing**: open the player on a display, it shows a 6-character code, and you type that code into the dashboard.
- **Live updates**: each player keeps a WebSocket open, so a saved playlist shows up on screen in well under a second. Polling covers the gaps if the socket drops.
- **Offline playback**: the player caches its files with a service worker and keeps looping them if the network or server goes away, including after a reboot.
- **Demo playlist**: a fresh install comes with a playlist called **Demo** (a few getting-started slides and a couple of live web pages), so a new screen has something to show straight away. Delete it whenever you like; it is only created once.
- **Fleet status**: see which screens are online, what they are playing, and send a refresh or reload remotely.

| Playlist editor | Player waiting to pair | Player showing content |
| --- | --- | --- |
| ![](docs/images/dashboard-playlist-editor.png) | ![](docs/images/player-pairing-code.png) | ![](docs/images/player-playing-menu.png) |

![A news slide on the player](docs/images/player-news-feature.png)

![Making a slide in the dashboard](docs/images/dashboard-slide-editor.png)

## Quick start

You need Docker with Compose. Sinewave is a single container with no separate database to run. Save this as `docker-compose.yml`, change the password, and run `docker compose up -d`:

```yaml
services:
  sinewave:
    image: ghcr.io/elliottmarter/sinewave:latest
    container_name: sinewave
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      ADMIN_USERNAME: admin          # username for the first super admin
      ADMIN_PASSWORD: change-me      # password for that account
      # SECURE_COOKIES: "true"       # set both when serving over HTTPS behind a reverse proxy
      # TRUST_PROXY: "true"
      # MAX_UPLOAD_MB: "1024"
      # PUID: "1000"                # owner of /data (Unraid: 99 / 100)
      # PGID: "1000"
    volumes:
      - sinewave-data:/data          # SQLite database and uploaded media

volumes:
  sinewave-data:
```

The image is built for `amd64` and `arm64`, so it also runs on a Raspberry Pi 4 or 5.

1. Open **http://localhost:8080** and sign in with your `ADMIN_USERNAME` and `ADMIN_PASSWORD`.
2. Upload some media under **Media**, then create a playlist under **Playlists** and add items to it. Or skip this and use the **Demo** playlist that comes with a fresh install.
3. On the screen you want to use, open **http://&lt;server&gt;:8080/player/**. It shows a pairing code.
4. In the dashboard, go to **Screens → Pair screen**, type the code, name the screen and pick the playlist.

The screen starts playing straight away.

### Users and roles

The first time Sinewave starts, it creates a **super admin** from `ADMIN_USERNAME` and `ADMIN_PASSWORD`. After that, both are no longer used: change passwords in the dashboard, and add more people under **Users**. There are two roles:

| | Super admin | Standard |
| --- | --- | --- |
| Rename screens, change their playlist, refresh and reload them | ✓ | ✓ |
| Upload and edit media and playlists | ✓ | ✓ |
| Pair new screens and remove screens | ✓ | |
| Add, edit and delete users | ✓ | |

There is always at least one super admin. If you're locked out, set `RESET_ADMIN_PASSWORD` to a new password and restart: it resets the `ADMIN_USERNAME` account (or creates it) as a super admin. Remove the variable and restart again afterwards.

### Running the player in kiosk mode

Any Chromium-based browser works. On a Raspberry Pi or mini PC:

```sh
chromium --kiosk --noerrdialogs --disable-infobars --autoplay-policy=no-user-gesture-required \
  https://signage.example.com/player/
```

Add `?debug` to the player URL to show a small status overlay (connection, cache, what's playing).

### HTTPS and offline caching

Browsers only allow service workers, and so offline caching, on **HTTPS** or **localhost**. Over plain `http://` on a LAN the player still works, but it streams everything live and goes blank if the server is unreachable. For real deployments put the server behind a reverse proxy with TLS (Caddy, Traefik, nginx) and set `SECURE_COOKIES=true` and `TRUST_PROXY=true`. For a quick LAN trial you can instead launch Chromium with `--unsafely-treat-insecure-origin-as-secure=http://<server>:8080`.

## Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `ADMIN_PASSWORD` | `admin` | Password for the first super admin, created on first start. Change it. |
| `ADMIN_USERNAME` | `admin` | Username for that first super admin, created on first start. |
| `RESET_ADMIN_PASSWORD` | | Set to reset the `ADMIN_USERNAME` account's password on start. Remove after use. |
| `PORT` | `8080` | Port published on the host. |
| `MAX_UPLOAD_MB` | `1024` | Largest file you can upload. |
| `SECURE_COOKIES` | `false` | Mark the session cookie `Secure` (set when using HTTPS). |
| `NEWS_REFRESH_MINUTES` | `15` | How often news feeds are re-fetched. |
| `WEB_PREVIEW_SCREENSHOTS` | `true` | Set to `false` to skip Chromium screenshots and only use a page's own preview picture (saves memory on very small devices). |
| `TRUST_PROXY` | `false` | Trust `X-Forwarded-*` headers from a reverse proxy. |
| `PUID` / `PGID` | `1000` / `1000` | User and group that own `/data` and run the server. The container fixes ownership of `/data` at startup, so a host folder mounted there just works. On Unraid use `99` / `100`. |

Sinewave runs as a single container. Everything it stores, the SQLite database (`sinewave.db`) and uploaded media, lives in the `data` volume mounted at `/data`, so backing up that one volume backs up everything. Outside Docker, set `DATA_DIR` to choose the folder.

## Project layout

```
server/     Node.js CMS: REST API, WebSocket hub, SQLite migrations, dashboard (server/public)
player/     The web player: plain HTML/JS plus a service worker, served at /player/
e2e/        End-to-end test that drives the dashboard and a headless player with Playwright
docs/       Architecture notes and screenshots
```

See [docs/architecture.md](docs/architecture.md) for how the pieces fit together.

## Development

To build and run from source instead of the published image:

```sh
git clone https://github.com/elliottmarter/Sinewave.git sinewave
cd sinewave
cp .env.example .env        # then set ADMIN_PASSWORD
docker compose up -d --build
```

To run the server without Docker, you need Node.js 22.13 or newer, which has SQLite built in.

```sh
cd server && npm install
npm run dev                 # data goes in ./data unless DATA_DIR is set
```

### Tests

Unit tests for feed parsing: `cd server && npm test`.

### End-to-end test

With the stack running:

```sh
cd e2e
npm install
npx playwright install chromium
npm test        # BASE_URL, ADMIN_USERNAME and ADMIN_PASSWORD can be overridden
```

It uploads media, builds a playlist, pairs a headless player, checks that a playlist change reaches the screen live, makes a slide from a template, adds a news feed (served by the test itself on port 8098, so a Docker server needs `--network host` or `FEED_HOST=host.docker.internal`), then cuts the player's network and checks it keeps playing from its cache. Screenshots are written to `e2e/screenshots/`. If `ffmpeg` is installed the test also covers video.

## Status

Early development. The MVP above works end to end, but there is no scheduling, multi-user support or native player wrapper yet. See the roadmap in [docs/architecture.md](docs/architecture.md).

## Contributing

Issues and pull requests are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for how to run Sinewave locally, run the tests and open a pull request. Please open an issue to discuss larger changes before starting work.

## License

Released under the [MIT License](LICENSE).
