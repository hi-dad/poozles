# Developing Poozles

## Run locally
```
npm install
npm run db:local   # once: create the local database table
npm run dev        # app + backend at http://localhost:8787 ("Developer sign-in" button)
npm test           # board-logic unit tests
```
No build step: `public/` is plain HTML/CSS/ES modules. `npm run dev:static` serves just the
front end (no backend); the app then saves on the device only.

## How saving works
Every device keeps a full copy of each puzzle in the browser (IndexedDB), so play is instant
and works offline. When the backend is reachable, changes are also copied to Cloudflare:

```
browser ──/api/*──► Worker (worker/index.js) ──► D1  (one row per puzzle: positions, timer…)
   ▲                    │                     └─► R2  (the puzzle photo)
   └── HTML/JS/CSS ─────┘  served as static assets
```

- **Who you are**: sign in with Apple or Google (`worker/auth.js`). The Worker checks the
  provider's signed ID token and keeps a session (a random token, stored only as a hash in
  D1). The verified email is the owner key for everything stored. Signing in is optional:
  without it, puzzles stay on the device.
- **Sync** (`public/js/sync.js`): saves upload at most every 30 s while playing, and right away when you leave the puzzle or hide the app; the library and
  open puzzle pull newer copies when the app is opened, focused, or every 2 minutes.
- **Conflicts** (same puzzle advanced on two devices): the most recent save wins; if the
  puzzle on screen is older than the cloud copy, the app asks "Load latest / Keep this one".
- The puzzle clock pauses after 2 idle minutes, so a device left open never saves over
  progress made elsewhere.

## Deploy (Cloudflare)
`wrangler.jsonc` describes the Worker, its D1 database and R2 bucket. The table schema is
in `worker/schema.sql`. Deploys run `npx wrangler deploy`, either from your computer or
automatically from GitHub via Cloudflare Workers Builds.

Live at https://poozles.app. Every push to this branch redeploys it.

### Sign-in setup
- Apple: App ID `app.poozles` and Services ID `app.poozles.web` (Sign in with Apple on,
  domain `poozles.app`, return URL `https://poozles.app/`).
- Google: a Web OAuth client with `https://poozles.app` and `http://localhost:8787` as
  JavaScript origins.
- The public IDs go in `vars` in `wrangler.jsonc`. For local development, `.dev.vars` holds
  `DEV_EMAIL=dev@example.com`, which turns on the "Developer sign-in" button.

## Code map
- `public/js/geometry.js`: grid sizing, seeded jigsaw cut, piece rendering
- `public/js/logic.js`: pure snapping/grouping rules (unit-tested)
- `public/js/game.js`: canvas board, tray, touch/mouse/trackpad input
- `public/js/main.js`: screens (library, new puzzle, play), saving, timer
- `public/js/textures.js`: procedural wood, felt and linen surfaces
- `public/js/sync.js`: cloud sync with the Worker
- `public/js/db.js`, `image.js`, `sound.js`: storage, photo prep, sound effects
- `worker/index.js`: the Cloudflare Worker (API + static assets), `worker/auth.js`: sign-in and sessions, `worker/schema.sql`: D1 tables
