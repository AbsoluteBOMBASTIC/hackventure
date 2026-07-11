# Hackventure 🗺️⚔️

**Learn to code, Duolingo-style.** 12 programming languages · 500 lessons each · streaks, XP, badges & a global leaderboard.

🌍 **Live at [hackventure.dev](https://hackventure.dev)**

Built by Iaroslav, age 12. 🚀

## How it works

- **`index.html`** — the whole app in one file (no build step, no framework)
- **`worker.js`** — Cloudflare Worker that generates AI lessons on demand and caches them **forever** in Workers KV, so every student gets the same lesson instantly
- First 25 lessons per language are hand-written; the rest are AI-generated on first reach
- Accounts & global leaderboard run on Supabase

## Architecture

```
Student → hackventure.dev (this repo, Cloudflare Pages)
             ↓ (lesson not pre-written?)
          hackventure-ai worker (worker.js)
             ↓ cache miss → Workers AI generates
          Workers KV (lesson saved forever)
```

## Deploying

- **Site:** pushes to `main` auto-deploy via Cloudflare Pages
- **Worker:** paste `worker.js` into the `hackventure-ai` worker in the Cloudflare dashboard (bindings: Workers AI as `AI`, KV namespace `hackventure-lessons` as `LESSONS`)
