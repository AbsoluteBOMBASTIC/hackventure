# Hackventure 🗺️⚔️

**Learn to code, Duolingo-style.** 12 programming languages · 500 lessons each · streaks, XP, badges & a global leaderboard — plus a Build tab where you write and run real code.

🌍 **Live at [hackventure.dev](https://hackventure.dev)**

Built by Iaroslav, age 12. 🚀

## How it works

- **`index.html`** — the whole app in one file (no build step, no framework)
- **`worker.js`** — Cloudflare Worker doing two jobs: generating AI lessons on demand and caching them **forever** in Workers KV, and running learners' code by proxying to the Piston API
- First 25 lessons per language are hand-written; the rest are AI-generated on first reach
- Accounts & global leaderboard run on Supabase

## The Build tab 🔨

Write real code in whatever language you're learning and actually run it — nothing to install. Three tabs:

- **▶️ Play** — a scratchpad that opens on a "Hello" program. Edit it, hit Run, see the output.
- **🎯 Challenges** — six tasks checked by what your program prints (print a number, count to five, print the even numbers…). Each one unlocks after a set number of lessons in that language, and pays out XP and gems the first time you pass it.
- **🏗️ Projects** — multi-step builds: *Countdown to Blast-off* 🚀 and the *Times-Table Machine* ✖️.

A challenge passes when your output matches the target. Trailing spaces and blank lines at the top and bottom are ignored, so formatting nits don't fail you.

**Runs here:** Python, Lua, JavaScript, TypeScript, Java, C++, C#, Rust, Swift, SQL.
**Doesn't:** Scratch (drag-and-drop blocks, no code to type) and HTML.

Limits: 20,000 characters of code and 5 seconds of run time per go.

## Architecture

```
Student → hackventure.dev (this repo, Cloudflare Pages)
                      ↓
            hackventure-ai worker (worker.js)
            ↙                            ↘
   lesson not cached?               Build tab: Run
          ↓                                ↓
   Workers AI writes it             Piston API runs it
          ↓                                ↓
   Workers KV (kept forever)        output → the editor
```

## Deploying

- **Site:** pushes to `main` auto-deploy via Cloudflare Pages
- **Worker:** paste `worker.js` into the `hackventure-ai` worker in the Cloudflare dashboard (bindings: Workers AI as `AI`, KV namespace `hackventure-lessons` as `LESSONS`)
- **Locally:** serve on `http://localhost:8888` — the worker only answers origins in its `ALLOWED_ORIGINS` list, so on any other port lessons and Run come back 403
