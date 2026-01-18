# Cool Frog — Telegram Mini App 

A frog-themed clicker / idle game built as a Telegram Web App using React, TypeScript and Vite.

This README is aimed at developers: it explains how the app is organized, how to run and build it, how Telegram integration works, and practical guidance for extending and deploying the project.

---

## Table of Contents

- Project overview
- Quick start
  - Requirements
  - Environment variables
  - Install & run locally
  - Build & preview
- Telegram Web App integration
  - How initialization works
  - Local development & emulation tips
  - Deep-links / referrals
- Architecture & key modules
  - Routing / pages
  - State management
  - HTTP layer & authentication
  - UI & assets
- API contract (observed endpoints)
  - Examples and typical request/response expectations
- Development workflows
  - Adding pages & components
  - Data fetching & mutations (react-query)
  - Styling & Tailwind
  - Linting & type-checking
- Debugging tips
- Production checklist & deployment notes
- Contributing
- License

---

## Project overview

- Front-end only code for a Telegram Web App game.
- Tech stack:
  - React 18 + TypeScript
  - Vite (dev server & bundling)
  - TailwindCSS (styles)
  - Zustand (local state)
  - @tanstack/react-query (data fetching & caching)
  - Axios wrapper as $http for API calls
  - Swiper, Day.js, Lucide icons, react-toastify
- Key concept: The app expects to be embedded in Telegram as a Web App. It reads Telegram init data and authenticates with the backend using that data.

---

## Quick start

### Requirements

- Node.js (recommended >= 18)
- npm (bundled with Node)
- A backend API compatible with the endpoints used in the app (or mock server)
- For production Telegram integration, an HTTPS host and a Telegram bot with the Web App URL set

### Environment variables

Create a `.env` (or use your preferred env mechanism). Recommended variables:

- VITE_API_URL — base URL for your backend API (if lib/http supports it)
- VITE_BOT_URL — used to build referral URLs in /friends (example in code)
- VITE_GTM_ID — (optional) Google Tag Manager ID if you replace the hard-coded one in index.html

> Note: index.html currently has a GTM ID hard-coded. Replace or parameterize it if required.

### Install

```bash
npm install
```

### Run (dev)

```bash
npm run dev
```

This starts Vite's dev server (hot module replacement). The app reads Telegram WebApp data — see "Telegram Web App integration" for local emulation tips.

### Build & Preview

Build for production:

```bash
npm run build
```

Preview the built app:

```bash
npm run preview
```

---

## Telegram Web App integration

This project is a Telegram Web App. The Telegram web SDK is included in `index.html`:

```html
<script src="https://telegram.org/js/telegram-web-app.js"></script>
```

The hook `useTelegramInitData` (src/hooks/useTelegramInitData.ts) handles parsing `Telegram.WebApp.initData` and exposes the parsed object to the app. In development (import.meta.env.DEV) the hook returns a `fakeData` object so you can run without a Telegram client.

Authentication flow (observed in src/App.tsx):

1. App reads initialization data (user, start_param).
2. If localStorage does not contain `"token"`, the app POSTs to `/auth/telegram-user` with Telegram user info.
3. The backend returns a token and `first_login` flag.
4. The app calls `setBearerToken(token)` (in src/lib/http) and stores the token locally (backend logic may require this).
5. The app syncs the user state performing `/clicker/sync`.

Important: Telegram Web Apps only work when loaded through Telegram (or when you emulate/spy the `window.Telegram` object). See local emulation below.

### Local development & emulation

- The app will use fake init data when run in DEV (useTelegramInitData returns fakeData). For more realistic testing:
  - Mock `window.Telegram = { WebApp: { initData: '<params>', initDataUnsafe: { ... }, ... } }` in the browser console (or create a small dev-only script).
  - Optionally set `window.Telegram = { WebApp: { platform: 'android' | 'ios' | 'tdesktop', initData: '...' }}` to test platform detection in src/App.tsx.
- To test real Telegram behavior:
  - Deploy your build to HTTPS.
  - Create or configure a Telegram bot (BotFather) and set the "Web App" URL to your deployed URL.
  - Open the bot in Telegram and press the Web App button (or deep link).

### Deep links / referrals

- The app expects referral start params like `startapp=ref{telegram_id}` in URLs (see `/friends` and App.tsx).
- `VITE_BOT_URL` is used in `/friends` to compose a referral link that opens the bot and passes start parameters.

---

## Architecture & key modules

High-level file layout (important files):
- src/main.tsx — app entry, wraps App with global Providers
- src/App.tsx — boot sequence (Telegram init, auth flow, splash, routing)
- src/router.tsx — routes:
  - `/` — Home
  - `/boost` — Boosts screen
  - `/leaderboard` — Leaderboard
  - `/friends` — Referral & friends
  - `/earn` — Tasks & rewards
  - `/missions` — Missions and upgrades
- src/pages/* — page components
- src/components/* — reusable components, UI elements, icons
- src/store/* — zustand stores:
  - useUserStore (user state & helper methods like UserTap)
  - uesStore (global game config: boosters, levels, missions, referral info)
- src/lib/http.ts — Axios wrapper. Exposes `$http`, and `setBearerToken`. All API calls in the app go through `$http`.
- src/hooks/useTelegramInitData.ts — handles parsing Telegram initData
- src/config/level-config.ts — mapping of level to assets/backgrounds
- public/images/* — assets served statically

State & fetching:
- Short-lived server data: react-query (useQuery/useMutation).
- Local player state & game progress: zustand (fast sync in UI).
- UI Patterns: modals/drawers use Radix or custom wrapper components in src/components/ui.

---

## API contract (observed endpoints)

These are the endpoints the frontend currently calls (names/parameters inferred from code). Your backend should provide compatible behavior.

- POST /auth/telegram-user
  - payload: { telegram_id, first_name, last_name, username?, referred_by? }
  - returns: { token: string, first_login: boolean }
- GET /clicker/sync
  - returns: { user: UserType, boosters: Record<BoosterTypes, BoosterType>, total_daily_rewards, daily_booster, max_level, levels, level_up, referral, mission_types, total_referals, ...}
- GET /clicker/tasks
  - returns: TaskType[]
- GET /clicker/referral-tasks
  - returns: ReferralTaskType[]
- POST /clicker/buy-booster
  - payload: { booster_type: string }
  - returns: updated user data and booster info
- POST /clicker/use-daily-booster
  - returns: current_energy, daily_booster_uses, next_available_at, etc.
- GET /clicker/missions?type=ID
  - returns: Mission[]
- GET /clicker/leaderboard?level_id=ID
  - returns: UserType[]
- GET /referred-users
  - returns: PaginationResponse<UserType>

Request headers:
- After authentication, the client sets a Bearer token via `setBearerToken()`; implement server-side token validation.

Notes:
- The front-end expects numeric values and some shaped objects (UserType, BoosterType, MissionType). Refer to `src/types` for local TypeScript definitions (UserType.ts, TaskType.ts, etc.) to mirror the DTOs on the backend.

---

## Development workflows

### Adding a page

1. Create the page component in `src/pages/`.
2. Add a route in `src/router.tsx`.
3. Reuse shared components from `src/components/`.
4. Add styles with Tailwind utility classes (config at tailwind.config.js).

### Adding a new API call

- Use the `$http` wrapper in `src/lib/http.ts`:
  - `$http.$get<T>(url, options)` — returns data typed as T
  - `$http.post(url, payload)` — regular Axios pattern (see usage in Boost, App)
- Use react-query:
  - Use `useQuery({ queryKey: [...], queryFn: () => $http.$get(...) })`
  - Use `useMutation` for POST actions (see src/pages/Boost.tsx).

### State updates / optimistic UI

- Global user state lives in `useUserStore` (Zustand). Mutations can set or merge state with `useUserStore.setState(...)`.
- For cached server data, rely on react-query invalidation or set new data in mutation `onSuccess`.

### Styling & Tailwind

- The project uses Tailwind; global styles in `src/index.css`.
- Assets (icons, images) are in `public/images`.
- For component variants, use `clsx`/`class-variance-authority` patterns and `cn` helper in `src/lib/utils`.

### Linting & types

- Lint: `npm run lint` (eslint).
- Type check: `tsc` is run during build (`npm run build` runs `tsc && vite build`).
- Add / update TypeScript types in `src/types`.

---

## Debugging tips

- Telegram Web App:
  - For local debugging, toggle DEV behavior in `useTelegramInitData` (it already returns fake data in DEV). To simulate real init data, set `window.Telegram.WebApp.initData` in the browser console.
- Inspect network:
  - The app uses Axios. Check requests/responses in browser DevTools Network tab.
- State debugging:
  - Use `useUserStore.getState()` in console to inspect Zustand store (call in console while app running).
- React Query devtools:
  - Add react-query devtools temporarily to inspect caches while developing.

---

## Production checklist & deployment notes

- Ensure your production host uses HTTPS (required for Telegram Web Apps).
- Configure your Telegram bot (via BotFather):
  - Set the Web App URL to the deployed app URL.
  - When using referrals, set the bot to accept start parameters.
- Replace or parameterize the GTM ID in `index.html` (currently hard-coded).
- Environment variables:
  - Set VITE_API_URL and VITE_BOT_URL appropriately for production.
- Build optimization:
  - `npm run build` — the build runs `tsc` then `vite build`. Confirm no TypeScript errors.
- Monitor:
  - Track errors and logs; toast-based user-visible errors are shown in the UI using react-toastify.

---

## Contributing

- Fork, create a feature branch, and open a PR with a clear title and description.
- Keep components small and reusable. Add tests where relevant and update types.
- Run:
  - `npm install`
  - `npm run lint`
  - `npm run build` locally before opening a PR
- Coding style:
  - Prefer TypeScript types for new interfaces and reuse `src/types`.
  - Use Tailwind classes. If a component gets large, move logic to a hook or helper.

---

## Useful references

- Telegram Web Apps: https://core.telegram.org/bots/webapps
- Vite: https://vitejs.dev/
- React Query: https://tanstack.com/query/latest
- Zustand: https://github.com/pmndrs/zustand
- TailwindCSS: https://tailwindcss.com/

---

## Files of interest

- src/App.tsx — app initialization and auth flow
- src/hooks/useTelegramInitData.ts — Telegram initData parsing
- src/lib/http.ts — axios wrapper and token handling
- src/store/user-store.ts — core player state & actions
- src/config/level-config.ts — background/frog imagery per level
- src/pages/* — business logic & UI pages

---

## Troubleshooting common issues

- "Telegram is undefined" when running locally — use the DEV fake initData or define `window.Telegram` manually.
- Authentication failing — confirm backend `/auth/telegram-user` returns `{ token }` and that setBearerToken sets Authorization header.
- Images missing in production — ensure `public/images` is included in deployment and that assets are referenced with absolute paths (`/images/...`).

---
