# StudyBuddy

An AI study partner you can **text and FaceTime**.

You schedule a study session on the dashboard. When it's time, StudyBuddy texts your iPhone. You tap the link, FaceTime the agent, and study with a voice tutor that already knows what you planned to work on. When you hang up, the call is transcribed, summarized and saved, and you can text follow-up questions about it later.

Built with Next.js, SQLite, OpenAI, and [Chert](https://docs.trychert.com) (FaceTime lines + iMessage) over [LiveKit](https://livekit.io).

> **Status:** hackathon prototype. One demo user, local storage, runs on your machine. No real accounts.

---

## How it works

```
Dashboard ── schedule session ──► worker ──► iMessage: "Time to study! Tap to call…"
                                                            │
                                                      you tap & FaceTime
                                                            │
                        Chert line ──► LiveKit room ──► my-agent (OpenAI realtime voice)
                                                            │  asks /api/agent/context for the topic
                                                            │  posts transcript on hang-up
                                                            ▼
History ◄── AI summary ◄── worker ◄── /api/ingest/call

Text the line any time ──► worker ──► reply grounded in your past call summaries
```

Two processes in this repo (`npm run dev` + `npm run worker`) and one voice agent in a separate repo: **[rayan-sharma/my-agent](https://github.com/rayan-sharma/my-agent)**.

---

## Features

- **Tasks & notes** on a dashboard, stored in SQLite.
- **Study sessions**: pick a task, set a time, get a "Time to study" banner and an iMessage when it's due.
- **One-tap call**: `/study/[id]` opens FaceTime to the agent; snooze or dismiss instead.
- **Voice tutor that knows your plan**: the agent pulls the current session's topic before greeting you.
- **Call history**: every call saved with transcript, summary, topics, and "try next" questions.
- **Text replies**: `CALL` → call link, `STOP` / `RESUME` → pause reminders, anything else → a short answer based on your recent calls.
- **Canvas LMS**: active courses and upcoming assignments on the dashboard.
- **Demo button**: "Study time now (linear algebra)" runs the whole flow in one click.

---

## Quick start

You need **Node 24**, an **OpenAI API key**, and for calls/texts a **Chert project** with a FaceTime line (`chert onboard`) plus a **LiveKit Cloud** project.

```bash
npm install
cp .env.example .env.local     # fill in, see Configuration
npm run seed                   # demo tasks & notes
npm run dev                    # http://localhost:3000
npm run worker                 # 2nd terminal: reminders, summaries, text replies
```

Then, in the `my-agent` repo:

```bash
npm start                                                   # LiveKit voice agent
npx @trychert/facetime@0.1.0-beta.10 dev --agent my-agent   # attach it to your Chert line
```

Open <http://localhost:3000/Dashboard> → **Demo: study time now**, or just FaceTime your line.

Without Chert keys the app still runs: reminders show in the web app only and nothing is sent.

---

## Configuration (`.env.local`)

Server-only. Never prefix with `NEXT_PUBLIC_`, never commit.

| Variable | Purpose |
|---|---|
| `OPENAI_API_KEY` | Call summaries and text replies. |
| `OPENAI_TEXT_MODEL` | Default `gpt-5.4-mini`. |
| `INGEST_SECRET` | Shared secret with `my-agent` (same value there, plus `BACKEND_URL=http://localhost:3000`). Protects `/api/ingest/call` and `/api/agent/context`. `openssl rand -hex 32`. |
| `CHERT_API_KEY` | Chert console → Developer → API keys (write). Enables iMessage. |
| `CHERT_PROJECT_ID` | Optional. Sent as `X-Chert-Project-Id`. |
| `CHERT_LINE_ID` | Optional. Defaults to the line matching `FACETIME_LINE_ADDRESS`. |
| `CHERT_MAX_PER_HOUR` | Text budget (trial 10, hackathon 100). |
| `DEMO_USER_PHONE` | Your verified phone, E.164. Texts only go here. |
| `FACETIME_LINE_ADDRESS` | The agent's FaceTime number, used in call links. |
| `DATABASE_PATH` / `DEMO_TIMEZONE` / `DEMO_MODE` | SQLite path, display zone, demo flag. |
| `CANVAS_BASE_URL` | Your Canvas instance, e.g. `https://school.instructure.com`. |

`npm run chert:check` verifies the Chert setup read-only (lists lines and recent messages, numbers masked).

---

## Scripts

| | |
|---|---|
| `npm run dev` | Next.js dev server |
| `npm run worker` | Background loop (5 s): summarize calls, send reminders, answer texts |
| `npm run seed` | Demo data (`-- --reset` to wipe) |
| `npm run chert:check` | Read-only Chert diagnostics |
| `npm run lint` / `npm run build` | Lint / production build |

---

## API

| Route | Method | Purpose |
|---|---|---|
| `/api/state` | GET | Everything the dashboard polls: tasks, notes, next/active session, call line |
| `/api/tasks`, `/api/tasks/[id]` | POST / PATCH / DELETE | Tasks |
| `/api/notes`, `/api/notes/[id]` | POST / PATCH / DELETE | Notes |
| `/api/commitments`, `/api/commitments/[id]` | POST / GET / PATCH | Schedule, snooze (`{action:"snooze"}`), cancel a session |
| `/api/calls/start` | POST | Start the session, return the FaceTime link |
| `/api/demo/start` | POST | Run the linear algebra demo |
| `/api/conversations`, `/api/conversations/[id]` | GET | Call + iMessage history |
| `/api/ingest/call` | POST | Voice agent posts a transcript (`Bearer $INGEST_SECRET`) |
| `/api/agent/context` | GET | Voice agent fetches the current topic (same secret) |
| `/api/canvas/dashboard` | GET | Canvas courses and assignments |

---

## Project layout

```
app/
  Dashboard/            main UI
  study/[id]/           start-call page
  login/                placeholder (redirects to Dashboard)
  api/                  route handlers (see API)
  components/           dashboard, history, Canvas, landing page
lib/server/             server-only: db, chert, imessage, calls, jobs, summarize, openai, …
scripts/
  worker.ts             background worker
  seed.ts               demo data
  chert-check.ts        Chert diagnostics
data/                   git-ignored local state
  studybuddy.db         SQLite: tasks, notes, sessions, job queue
  conversations/*.json  one file per call / text thread (transcript + summary)
  chert-state.json      handled messages + hourly send count
```

---

## Known limitations

- **Inbound calls only.** Chert shared lines can't dial out, so "Start call" opens FaceTime on *your* phone and you call the agent.
- **Camera not verified.** The agent is wired for vision, but whether Chert forwards caller video into the room is unconfirmed. It says "I can't see you" rather than guessing.
- **Chert sessions expire** (~1 h). If calls stop, run `chert onboard --status`; re-verifying may assign a new number → update `FACETIME_LINE_ADDRESS`.
- **Texts** go only to the one verified phone, text-only, rate-limited.
- **No auth.** The login page is cosmetic; web routes are open. Single demo user.
- `npm run build` fails type-checking on unused copied files under `app/components/page/ui/*` and `app/components/library/ai/client.ts`. `npm run dev` is unaffected.
